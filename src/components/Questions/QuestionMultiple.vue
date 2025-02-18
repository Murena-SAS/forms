<!--
  - @copyright Copyright (c) 2020 John Molakvoæ <skjnldsv@protonmail.com>
  -
  - @author John Molakvoæ <skjnldsv@protonmail.com>
  -
  - @license AGPL-3.0-or-later
  -
  - This program is free software: you can redistribute it and/or modify
  - it under the terms of the GNU Affero General Public License as
  - published by the Free Software Foundation, either version 3 of the
  - License, or (at your option) any later version.
  -
  - This program is distributed in the hope that it will be useful,
  - but WITHOUT ANY WARRANTY; without even the implied warranty of
  - MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the
  - GNU Affero General Public License for more details.
  -
  - You should have received a copy of the GNU Affero General Public License
  - along with this program. If not, see <http://www.gnu.org/licenses/>.
  -
  -->

<template>
	<Question v-bind="questionProps"
		:title-placeholder="answerType.titlePlaceholder"
		:warning-invalid="answerType.warningInvalid"
		:content-valid="contentValid"
		:shift-drag-handle="shiftDragHandle"
		v-on="commonListeners">
		<template #actions>
			<NcActionCheckbox :checked="extraSettings?.shuffleOptions"
				@update:checked="onShuffleOptionsChange">
				{{ t('forms', 'Shuffle options') }}
			</NcActionCheckbox>
			<NcActionCheckbox :checked="allowOtherAnswer"
				@update:checked="onAllowOtherAnswerChange">
				{{ t('forms', 'Add "other"') }}
			</NcActionCheckbox>
		</template>
		<template v-if="readOnly">
			<fieldset :name="name || undefined" :aria-labelledby="titleId">
				<NcCheckboxRadioSwitch v-for="(answer) in sortedOptions"
					:key="answer.id"
					:checked="questionValues"
					:value="answer.id.toString()"
					:name="`${id}-answer`"
					:type="isUnique ? 'radio' : 'checkbox'"
					:required="checkRequired(answer.id)"
					@update:checked="onChange"
					@keydown.enter.exact.prevent="onKeydownEnter">
					{{ answer.text }}
				</NcCheckboxRadioSwitch>
				<div v-if="allowOtherAnswer" class="question__other-answer">
					<NcCheckboxRadioSwitch :checked="questionValues"
						:value="otherAnswer ?? QUESTION_EXTRASETTINGS_OTHER_PREFIX"
						:name="`${id}-answer`"
						:type="isUnique ? 'radio' : 'checkbox'"
						:required="checkRequired('other-answer')"
						class="question__label"
						@update:checked="onChangeOther"
						@keydown.enter.exact.prevent="onKeydownEnter">
						{{ t('forms', 'Other:') }}
					</NcCheckboxRadioSwitch>
					<NcInputField class="question__input"
						:label="placeholderOtherAnswer"
						:required="otherAnswer !== undefined"
						:value.sync="otherAnswerText" />
				</div>
			</fieldset>
		</template>

		<template v-else>
			<ul class="question__content">
				<!-- Answer text input edit -->
				<AnswerInput v-for="(answer, index) in sortedOptions"
					:key="index /* using index to keep the same vnode after new answer creation */"
					ref="input"
					:answer="answer"
					:index="index"
					:is-unique="isUnique"
					:is-dropdown="false"
					:max-option-length="maxStringLengths.optionText"
					@delete="deleteOption"
					@update:answer="updateAnswer"
					@focus-next="focusNextInput"
					@tabbed-out="checkValidOption" />
				<li v-if="allowOtherAnswer" class="question__item">
					<div :is="pseudoIcon" class="question__item__pseudoInput" />
					<input :placeholder="t('forms', 'Other')"
						class="question__input"
						:maxlength="maxStringLengths.optionText"
						minlength="1"
						type="text"
						:readonly="!readOnly">
				</li>
				<li v-if="!isLastEmpty || hasNoAnswer" class="question__item">
					<div :is="pseudoIcon" class="question__item__pseudoInput" />
					<input ref="pseudoInput"
						class="question__input"
						:aria-label="t('forms', 'Add a new answer')"
						:placeholder="t('forms', 'Add a new answer')"
						:maxlength="maxStringLengths.optionText"
						minlength="1"
						type="text"
						@input="addNewEntry">
				</li>
			</ul>
		</template>
	</Question>
</template>

<script>
import { showError } from '@nextcloud/dialogs'
import { emit } from '@nextcloud/event-bus'
import { generateOcsUrl } from '@nextcloud/router'
import axios from '@nextcloud/axios'
import NcActionCheckbox from '@nextcloud/vue/dist/Components/NcActionCheckbox.js'
import NcCheckboxRadioSwitch from '@nextcloud/vue/dist/Components/NcCheckboxRadioSwitch.js'
import NcInputField from '@nextcloud/vue/dist/Components/NcInputField.js'
import IconCheckboxBlankOutline from 'vue-material-design-icons/CheckboxBlankOutline.vue'
import IconRadioboxBlank from 'vue-material-design-icons/RadioboxBlank.vue'

import AnswerInput from './AnswerInput.vue'
import QuestionMixin from '../../mixins/QuestionMixin.js'
import GenRandomId from '../../utils/GenRandomId.js'
import logger from '../../utils/Logger.js'

const QUESTION_EXTRASETTINGS_OTHER_PREFIX = 'system-other-answer:'

export default {
	name: 'QuestionMultiple',

	components: {
		AnswerInput,
		IconCheckboxBlankOutline,
		IconRadioboxBlank,
		NcActionCheckbox,
		NcCheckboxRadioSwitch,
		NcInputField,
	},

	mixins: [QuestionMixin],

	data() {
		return {
			/**
			 * This is used to cache the "other" answer, meaning if the user:
			 * checks "other" types text, unchecks "other" and then re-check "other" the typed text is preserved
			 */
			cachedOtherAnswerText: '',
			QUESTION_EXTRASETTINGS_OTHER_PREFIX,
		}
	},

	computed: {
		contentValid() {
			return this.answerType.validate(this)
		},

		isLastEmpty() {
			const value = this.options[this.options.length - 1]
			return value?.text?.trim?.().length === 0
		},

		isUnique() {
			return this.answerType.unique === true
		},

		hasNoAnswer() {
			return this.options.length === 0
		},

		areNoneChecked() {
			return this.values.length === 0
		},

		shiftDragHandle() {
			return !this.readonly && this.options.length !== 0 && !this.isLastEmpty
		},

		pseudoIcon() {
			return this.isUnique ? IconRadioboxBlank : IconCheckboxBlankOutline
		},

		placeholderOtherAnswer() {
			if (this.readOnly) {
				return this.answerType.submitPlaceholder
			}
			return this.answerType.createPlaceholder
		},

		questionValues() {
			return this.isUnique ? this.values?.[0] : this.values
		},

		titleId() {
			return `q${this.index}_title`
		},

		allowOtherAnswer() {
			return this.extraSettings?.allowOtherAnswer ?? false
		},

		/**
		 * The full "other" answer including prefix, undefined if no "other answer"
		 */
		otherAnswer() {
			return this.values.find((v) => v.startsWith(QUESTION_EXTRASETTINGS_OTHER_PREFIX))
		},

		/**
		 * The text value of the "other" answer
		 */
		otherAnswerText: {
			get() {
				return this.cachedOtherAnswerText
			},
			/**
			 * Called when the value of the "other" anwer is changed input
			 * @param {string} value the new text of the "other" answer
			 */
			set(value) {
				this.cachedOtherAnswerText = value
				// Prefix the value
				const prefixedValue = `${QUESTION_EXTRASETTINGS_OTHER_PREFIX}${value}`
				// emit the values and add the "other" answer
				this.$emit(
					'update:values',
					this.isUnique ? [prefixedValue] : [...this.values.filter((v) => !v.startsWith(QUESTION_EXTRASETTINGS_OTHER_PREFIX)), prefixedValue],
				)
			},
		},
	},

	watch: {
		// Ensure that the "other" answer is reset after toggling the checkbox
		otherAnswer() {
			this.resetOtherAnswerText()
		},
	},

	mounted() {
		// Ensure the initial "other" answer is set
		this.resetOtherAnswerText()
	},

	methods: {
		/**
		 * Resets the local "other" answer text to the one from the options if available
		 */
		resetOtherAnswerText() {
			if (this.otherAnswer) {
				// make sure to use cached value if empty value is passed
				this.cachedOtherAnswerText = this.otherAnswer.slice(QUESTION_EXTRASETTINGS_OTHER_PREFIX.length) || this.cachedOtherAnswerText
			}
		},

		onChange(value) {
			this.$emit('update:values', this.isUnique ? [value] : value)
		},

		/**
		 * Handle toggling the "other"-answer checkbox / radio switch
		 * @param {string|string[]} value The new value of the answer(s)
		 */
		onChangeOther(value) {
			value = [value].flat()
			const pureValue = value.filter((v) => !v.startsWith(QUESTION_EXTRASETTINGS_OTHER_PREFIX))

			if (value.length > pureValue.length) {
				// make sure to add the cached test on re-enable
				this.onChange([...pureValue, `${QUESTION_EXTRASETTINGS_OTHER_PREFIX}${this.cachedOtherAnswerText}`])
			} else {
				this.onChange(value)
			}
		},

		/**
		 * Is the provided answer required ?
		 * This is needed for checkboxes as html5
		 * doesn't allow to require at least ONE checked.
		 * So we require the one that are checked or all
		 * if none are checked yet.
		 *
		 * @param {number} id the answer id
		 * @return {boolean}
		 */
		checkRequired(id) {
			// false, if question not required
			if (!this.isRequired) {
				return false
			}

			// true for Radiobuttons
			if (this.isUnique) {
				return true
			}

			// For checkboxes, only required if no other is checked
			return this.areNoneChecked
		},

		/**
		 * Remove any empty options when leaving an option
		 */
		checkValidOption() {
			// When leaving edit mode, filter and delete empty options
			this.options.forEach(option => {
				if (!option.text) {
					this.deleteOption(option.id)
				}
			})
		},

		/**
		 * Set focus on next AnswerInput
		 *
		 * @param {number} index Index of current option
		 */
		focusNextInput(index) {
			if (index < this.options.length - 1) {
				this.$refs.input[index + 1].focus()
			} else if (!this.isLastEmpty || this.hasNoAnswer) {
				this.$refs.pseudoInput.focus()
			}
		},

		/**
		 * Update the options
		 * This will handle updating the form (emitting the changes) and update last changed property
		 *
		 * @param {Array} options options to change
		 */
		updateOptions(options) {
			this.$emit('update:options', options)
			emit('forms:last-updated:set', this.formId)
		},

		/**
		 * Update an existing answer locally
		 *
		 * @param {string|number} id the answer id
		 * @param {object} answer the answer to update
		 */
		updateAnswer(id, answer) {
			const options = [...this.options]
			const answerIndex = options.findIndex(option => option.id === id)
			options[answerIndex] = answer

			this.updateOptions(options)
		},

		/**
		 * Add a new empty answer locally
		 * @param {InputEvent} event The input event that triggered adding a new entry
		 */
		addNewEntry({ target }) {
			// Add local entry
			const options = [
				...this.options,
				{
					id: GenRandomId(),
					questionId: this.id,
					text: target?.value ?? '',
					local: true,
				},
			]

			// Reset the "new answer" input if needed
			if (this.$refs.pseudoInput) {
				this.$refs.pseudoInput.value = ''
			}

			// Update questions
			this.updateOptions(options)

			this.$nextTick(() => {
				// Set focus to the created input element
				this.focusIndex(options.length - 1)

				// Trigger onInput on new AnswerInput for posting the new option to the API
				this.$refs.input[options.length - 1].onInput()
			})
		},

		/**
		 * Restore an option locally
		 *
		 * @param {object} option the option
		 * @param {number} index the options index in this.options
		 */
		restoreOption(option, index) {
			const options = this.options.slice()
			options.splice(index, 0, option)

			this.updateOptions(options)
			this.focusIndex(index)
		},

		/**
		 * Delete an option
		 *
		 * @param {number} id the options id
		 */
		deleteOption(id) {
			const options = this.options.slice()
			const optionIndex = options.findIndex(option => option.id === id)

			if (options.length === 1) {
				// Clear Text, but don't remove. Will be removed, when leaving edit-mode
				options[0].text = ''
			} else {
				// Remove entry
				const option = Object.assign({}, this.options[optionIndex])

				// delete locally
				options.splice(optionIndex, 1)

				// delete from Db
				this.deleteOptionFromDatabase(option)
			}

			// Update question
			this.updateOptions(options)

			this.$nextTick(() => {
				this.focusIndex(optionIndex - 1)
			})
		},

		/**
		 * Delete the option from Db in background.
		 * Restore option if delete not possible
		 *
		 * @param {object} option The option to delete
		 */
		deleteOptionFromDatabase(option) {
			const optionIndex = this.options.findIndex(opt => opt.id === option.id)

			if (!option.local) {
				// let's not await, deleting in background
				axios.delete(generateOcsUrl('apps/forms/api/v2.2/option/{id}', { id: option.id }))
					.catch(error => {
						logger.error('Error while deleting an option', { error, option })
						showError(t('forms', 'There was an issue deleting this option'))
						// restore option
						this.restoreOption(option, optionIndex)
					})
			}
		},

		/**
		 * Focus the input matching the index
		 *
		 * @param {number} index the value index
		 */
		focusIndex(index) {
			const inputs = this.$refs.input
			if (inputs && inputs[index]) {
				const input = inputs[index]
				input.focus()
			}
		},

		/**
		 * Update status extra setting allowOtherAnswer and save on DB
		 *
		 * @param {boolean} allowOtherAnswer show/hide field for other answer
		 */
		onAllowOtherAnswerChange(allowOtherAnswer) {
			return this.onExtraSettingsChange({ allowOtherAnswer })
		},
	},
}
</script>

<style lang="scss" scoped>
.question__content {
	display: flex;
	flex-direction: column;
}

.question__item {
	position: relative;
	display: inline-flex;
	min-height: 44px;

	&__pseudoInput {
		color: var(--color-primary-element);
		margin-inline-start: -2px;
		z-index: 1;
	}

	.question__input {
		width: 100%;
		position: relative;
		inset-inline-start: -34px;
		inset-block-start: 1px;
		margin-inline-end: 10px !important;
		padding-inline-start: 36px !important;
	}

	.question__label {
		flex: 1 1 100%;
		// Overwrite guest page core styles
		text-align: start !important;
		// Some rounding issues lead to this strange number, so label and answerInput show up a the same position, working on different browsers.
		padding-block: 6.5px 0;
		padding-inline: 30px 0;
		line-height: 22px;
		min-height: 34px;
		height: min-content;
		position: relative;

		&::before {
			box-sizing: border-box;
			// Adjust position manually for proper position to text
			position: absolute;
			inset-block-start: 10px;
			width: 16px;
			height: 16px;
			margin-inline: -30px 14px !important;
			margin-block-end: 0;
		}
	}
}

.question__other-answer {
	display: flex;
	gap: 4px 16px;
	flex-wrap: wrap;

	.question__label {
		flex-basis: content;
	}

	.question__input {
		flex: 1;
		min-width: 260px;
	}

	.input-field__input {
		min-height: 44px;
	}
}

.question__other-answer:deep() .input-field__input {
	min-height: 44px;
}

</style>

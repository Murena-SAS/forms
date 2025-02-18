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
		</template>
		<NcSelect v-if="readOnly"
			v-model="selectedOption"
			:name="name || undefined"
			:placeholder="selectOptionPlaceholder"
			:multiple="isMultiple"
			:required="isRequired"
			:options="sortedOptions"
			:searchable="false"
			label="text"
			@input="onInput" />

		<ol v-if="!readOnly" class="question__content">
			<!-- Answer text input edit -->
			<AnswerInput v-for="(answer, index) in options"
				:key="index /* using index to keep the same vnode after new answer creation */"
				ref="input"
				:answer="answer"
				:index="index"
				:is-unique="!isMultiple"
				:is-dropdown="true"
				:max-option-length="maxStringLengths.optionText"
				@delete="deleteOption"
				@update:answer="updateAnswer"
				@focus-next="focusNextInput"
				@tabbed-out="checkValidOption" />

			<li v-if="!isLastEmpty || hasNoAnswer" class="question__item">
				<input ref="pseudoInput"
					v-model="inputValue"
					:aria-label="t('forms', 'Add a new answer')"
					:placeholder="t('forms', 'Add a new answer')"
					class="question__input"
					:maxlength="maxStringLengths.optionText"
					minlength="1"
					type="text"
					@input="addNewEntry">
			</li>
		</ol>
	</Question>
</template>

<script>
import { showError } from '@nextcloud/dialogs'
import { emit } from '@nextcloud/event-bus'
import { generateOcsUrl } from '@nextcloud/router'
import axios from '@nextcloud/axios'
import NcActionCheckbox from '@nextcloud/vue/dist/Components/NcActionCheckbox.js'
import NcSelect from '@nextcloud/vue/dist/Components/NcSelect.js'

import AnswerInput from './AnswerInput.vue'
import QuestionMixin from '../../mixins/QuestionMixin.js'
import GenRandomId from '../../utils/GenRandomId.js'
import logger from '../../utils/Logger.js'

export default {
	name: 'QuestionDropdown',

	components: {
		AnswerInput,
		NcActionCheckbox,
		NcSelect,
	},

	mixins: [QuestionMixin],

	data() {
		return {
			selectedOption: null,
			inputValue: '',
		}
	},

	computed: {
		selectOptionPlaceholder() {
			if (this.readOnly) {
				return this.answerType.submitPlaceholder
			}
			return this.answerType.createPlaceholder
		},

		contentValid() {
			return this.answerType.validate(this)
		},

		isLastEmpty() {
			const value = this.options[this.options.length - 1]
			return value?.text?.trim().length === 0
		},

		isMultiple() {
			// This can be extended if we want to include support for <select multiple>
			return false
		},

		hasNoAnswer() {
			return this.options.length === 0
		},

		areNoneChecked() {
			return this.values.length === 0
		},

		shiftDragHandle() {
			return !this.readOnly && this.options.length !== 0 && !this.isLastEmpty
		},
	},

	mounted() {
		// Init selected options from values prop
		if (this.values) {
			const selected = this.values.map(id => this.options.find(option => option.id === id))
			this.selectedOption = this.isMultiple ? selected : selected[0]
		}
	},

	methods: {
		onInput(option) {
			if (Array.isArray(option)) {
				this.$emit('update:values', [...new Set(option.map((opt) => opt.id))])
				return
			}

			// Simple select
			this.$emit('update:values', option ? [option.id] : [])
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
			const options = this.options.slice()
			const answerIndex = options.findIndex(option => option.id === id)
			options[answerIndex] = answer

			this.updateOptions(options)
		},

		/**
		 * Add a new empty answer locally
		 */
		addNewEntry() {
			// Add local entry
			const options = this.options.slice()
			options.push({
				id: GenRandomId(),
				questionId: this.id,
				text: this.inputValue,
				local: true,
			})

			this.inputValue = ''

			// Update question
			this.updateOptions(options)

			this.$nextTick(() => {
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
						logger.error('Error while deleting an option', { option, error })
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

	.question__input {
		width: 100%;
		position: relative;
		inset-inline-start: -12px;
		margin-inline-end: 32px !important;
	}
}
</style>

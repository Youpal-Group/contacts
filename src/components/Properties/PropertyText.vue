<!--
  - @copyright Copyright (c) 2018 John Molakvoæ <skjnldsv@protonmail.com>
  -
  - @author John Molakvoæ <skjnldsv@protonmail.com>
  -
  - @license GNU AGPL version 3 or any later version
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
	<div v-if="propModel" class="property">
		<!-- title if first element -->
		<PropertyTitle v-if="isFirstProperty && propModel.icon"
			:icon="propModel.icon"
			:readable-name="propModel.readableName" />

		<div class="property__row">
			<!-- type selector -->
			<Multiselect v-if="propModel.options"
				v-model="localType"
				:options="options"
				:placeholder="t('contacts', 'Select type')"
				:taggable="true"
				tag-placeholder="create"
				:disabled="isReadOnly"
				class="property__label"
				track-by="id"
				label="name"
				@tag="createLabel"
				@input="updateType" />

			<!-- if we do not support any type on our model but one is set anyway -->
			<div v-else-if="selectType" class="property__label">
				{{ selectType.name }}
			</div>

			<!-- no options, empty space -->
			<div v-else class="property__label">
				{{ propModel.readableName }}
			</div>

			<!-- textarea for note -->
			<textarea v-if="propName === 'note'"
				id="textarea"
				ref="textarea"
				v-model.trim="localValue"
				:inputmode="inputmode"
				:readonly="isReadOnly"
				class="property__value"
				@input="updateValueNoDebounce"
				@mousemove="resizeHeight"
				@keypress="resizeHeight" />

			<!-- OR default to input -->
			<input v-else
				v-model.trim="localValue"
				:inputmode="inputmode"
				:readonly="isReadOnly"
				:class="{'property__value--with-ext': haveExtHandler}"
				type="text"
				class="property__value"
				:placeholder="placeholder"
				@input="updateValue">

			<!-- external link -->
			<a v-if="haveExtHandler"
				:href="externalHandler"
				:class="{'property__ext': true, 'icon-external': true, 'no-move': isReadOnly}"
				target="_blank" />

			<!-- props actions -->
			<PropertyActions
				v-if="!isReadOnly"
				:actions="actions"
				:property-component="this"
				@delete="deleteProperty" />
		</div>
	</div>
</template>

<script>
import Multiselect from '@nextcloud/vue/dist/Components/NcMultiselect'
import debounce from 'debounce'
import PropertyMixin from '../../mixins/PropertyMixin'
import PropertyTitle from './PropertyTitle'
import PropertyActions from './PropertyActions'

export default {
	name: 'PropertyText',

	components: {
		Multiselect,
		PropertyTitle,
		PropertyActions,
	},

	mixins: [PropertyMixin],

	props: {
		propName: {
			type: String,
			default: 'text',
			required: true,
		},
		value: {
			type: String,
			default: '',
			required: true,
		},
	},

	computed: {
		inputmode() {
			if (this.propName === 'tel') {
				return 'tel'
			} else if (this.propName === 'email') {
				return 'email'
			} else if (this.propType === 'uri') {
				return 'url'
			}
			return false
		},
		URLScheme() {
			if (this.propName === 'tel') {
				return 'tel:'
			} else if (this.propName === 'email') {
				return 'mailto:'
			// if no scheme (roughly checking for the colon char)
			} else if (this.propType === 'uri' && this.localValue && this.localValue.indexOf(':') === -1) {
				return 'https://'
			} else if (this.propType === 'uri') {
				return '' // return empty, we already have a scheme in the value
			}
			return false
		},

		// format external link
		externalHandler() {
			if (this.URLScheme !== false) {
				return `${this.URLScheme}${this.localValue}`
			}
			return ''
		},

		haveExtHandler() {
			return this.externalHandler.trim() !== '' && this.localValue && this.localValue.length > 0
		},

		/**
		 * Return the selected type placeholder if any
		 * or the propModel default placeholder
		 *
		 * @return {string}
		 */
		placeholder() {
			if (this.localType?.placeholder) {
				return this.localType.placeholder
			}
			return this.propModel.placeholder
		},
	},

	mounted() {
		this.resizeHeight()
	},

	methods: {
		/**
		 * Watch textarea resize and update the gridSize accordingly
		 */
		resizeHeight: debounce(function(e) {
			if (this.$refs.textarea && this.$refs.textarea.offsetHeight) {
				// adjust textarea size to content (2 = border)
				this.$refs.textarea.style.height = `${this.$refs.textarea.scrollHeight + 2}px`
				// send resize event to warn we changed from the inside
				this.$emit('resize')
			}
		}, 100),

		/**
		 * Since we want to also trigger a resize
		 * but both of them have different debounce
		 * let's use a standard methods and call them both
		 *
		 * @param {object} e event
		 */
		updateValueNoDebounce(e) {
			this.resizeHeight(e)
			this.updateValue(e)
		},
	},
}
</script>
<style lang="scss" scoped>
.property__label:not(.multiselect) {
	overflow: hidden;
	white-space: nowrap;
	text-overflow: ellipsis;
	opacity: 0.7;
}
.property__row {
	position: relative;
	display: flex;
	align-items: center;
}
.property__label, .property__label.multiselect {
	flex: 1 0;
	width: 60px;
	min-width: 60px !important;
	max-width: 120px;
	user-select: none;
	text-align: right;
	background-size: 16px;
}
</style>

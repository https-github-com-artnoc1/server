 <!--
  - @copyright Copyright (c) 2020 John Molakvoæ <skjnldsv@protonmail.com>
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
  - MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
  - GNU Affero General Public License for more details.
  -
  - You should have received a copy of the GNU Affero General Public License
  - along with this program.  If not, see <http://www.gnu.org/licenses/>.
  -
  -->
<template>
	<HeaderMenu id="unified-search"
		class="unified-search"
		@open="onOpen"
		@close="onClose">
		<!-- Header icon -->
		<template #trigger>
			<span class="icon-search-white" />
		</template>

		<!-- Search input -->
		<div class="unified-search__input-wrapper">
			<input ref="input"
				v-model="query"
				class="unified-search__input"
				type="search"
				:placeholder="t('core', 'Search for {types} …', { types: typesNames.join(', ') })"
				@input="onInputDebounced">
		</div>

		<EmptyContent v-if="!hasResults && !isLoading" icon="icon-search">
			{{ t('core', 'Start typing to search') }}
			<template v-if="isShortQuery" #desc>
				{{ n('core',
					'Please enter {minSearchLength} character or more to search',
					'Please enter {minSearchLength} characters  or more to search',
					minSearchLength,
					{minSearchLength}) }}
			</template>
		</EmptyContent>

		<EmptyContent v-else-if="isLoading" icon="icon-loading">
			{{ t('core', 'Searching …') }}
		</EmptyContent>

		<!-- Grouped search results -->
		<template v-else>
			<ul v-for="(list, type) in results"
				:key="type"
				class="unified-search__results"
				:class="`unified-search__results-${type}`"
				:aria-label="typesMap[type]">
				<!-- Search results -->
				<SearchResult v-for="result in limitIfAny(list, type)" :key="result.resourceUrl" v-bind="result">
					<ActionLink icon="icon-external" :href="result.resourceUrl" />
				</SearchResult>

				<!-- Load more button -->
				<SearchResult v-if="!reached[type]"
					:title="loading[type]
						? t('core', 'Loading more results …')
						: t('core', 'Load more results')"
					:icon="loading[type] ? 'icon-loading-small' : ''"
					@click="loadMore(type)">
					<ActionButton v-if="!loading[type]" icon="icon-confirm" @click.stop="loadMore(type)" />
				</SearchResult>
			</ul>
		</template>
	</HeaderMenu>
</template>

<script>
import { getTypes, search, defaultLimit } from '../services/UnifiedSearchService'
import ActionButton from '@nextcloud/vue/dist/Components/ActionButton'
import ActionLink from '@nextcloud/vue/dist/Components/ActionLink'
import EmptyContent from '@nextcloud/vue/dist/Components/EmptyContent'

import debounce from 'debounce'

import HeaderMenu from '../components/HeaderMenu'
import SearchResult from '../components/UnifiedSearch/SearchResult'

const minSearchLength = 2

export default {
	name: 'UnifiedSearch',

	components: {
		ActionButton,
		ActionLink,
		EmptyContent,
		HeaderMenu,
		SearchResult,
	},

	data() {
		return {
			types: [],

			cursors: {},
			limits: {},
			loading: {},
			reached: {},
			results: {},

			query: '',

			defaultLimit,
			minSearchLength,
		}
	},

	computed: {
		typesIDs() {
			return this.types.map(type => type.id)
		},
		typesNames() {
			return this.types.map(type => type.name)
		},
		typesMap() {
			return this.types.reduce((prev, curr) => {
				prev[curr.id] = curr.name
				return prev
			}, {})
		},

		/**
		 * Is there any result to display
		 * @returns {boolean}
		 */
		hasResults() {
			return Object.keys(this.results).length !== 0
		},

		/**
		 * Is the current search too short
		 * @returns {boolean}
		 */
		isShortQuery() {
			return this.query && this.query.trim().length < minSearchLength
		},

		/**
		 * Is there any search in progress
		 * @returns {boolean}
		 */
		isLoading() {
			return Object.values(this.loading).indexOf(true) !== -1
		},
	},

	async created() {
		this.types = await getTypes()
		console.debug('Unified Search initialized with the following providers', this.types)
	},

	methods: {
		async onOpen() {
			this.focusInput()
			// Update types list in the background
			this.types = await getTypes()
		},
		onClose() {
			this.cursors = {}
			this.limits = {}
			this.loading = {}
			this.reached = {}
			this.results = {}
			this.query = ''
		},

		/**
		 * Focus the search input on next tick
		 */
		focusInput() {
			this.$nextTick(() => {
				this.$refs.input.focus()
				this.$refs.input.select()
			})
		},

		/**
		 * Start searching on input
		 * @param {Event} e the input event
		 */
		async onInput(e) {
			// Do not search if not long enough
			if (this.query.trim() === '' || this.isShortQuery) {
				return
			}

			this.typesIDs.forEach(async type => {
				this.$set(this.loading, type, true)
				const request = await search(type, this.query)

				// Process results
				if (request.data.entries.length > 0) {
					this.$set(this.results, type, request.data.entries)
				} else {
					this.$delete(this.results, type)
				}

				// Save cursor if any
				if (request.data.cursor) {
					this.$set(this.cursors, type, request.data.cursor)
				} else if (!request.data.isPaginated) {
					// If no cursor and no pagination, we save the default amount
					// provided by server's initial state `defaultLimit`
					this.$set(this.limits, type, this.defaultLimit)
				}

				// Check if we reached end of pagination
				if (request.data.entries.length < this.defaultLimit) {
					this.$set(this.reached, type, true)
				}

				this.$set(this.loading, type, false)
			})
		},
		onInputDebounced: debounce(function(e) {
			this.onInput(e)
		}, 200),

		/**
		 * Load more results for the provided type
		 * @param {String} type type
		 */
		async loadMore(type) {
			// If already loading, ignore
			if (this.loading[type]) {
				return
			}
			this.$set(this.loading, type, true)

			if (this.cursors[type]) {
				const request = await search(type, this.query)

				// Save cursor if any
				if (request.data.cursor) {
					this.$set(this.cursors, type, request.data.cursor)
				}

				if (request.data.entries.length > 0) {
					this.results[type].push(...request.data.entries)
				}

				// Check if we reached end of pagination
				if (request.data.entries.length < this.defaultLimit) {
					this.$set(this.reached, type, true)
				}
			} else

			// If no cursor, we might have all the results already,
			// let's fake pagination and show the next xxx entries
			if (this.limits[type] && this.limits[type] >= 0) {
				this.limits[type] += this.defaultLimit

				// Check if we reached end of pagination
				if (this.limits[type] >= this.results[type].length) {
					this.$set(this.reached, type, true)
				}
			}

			this.$set(this.loading, type, false)
		},

		/**
		 * Return a subset of the array if the search provider
		 * doesn't supports pagination
		 *
		 * @param {Array} list the results
		 * @param {string} type the type
		 * @returns {Array}
		 */
		limitIfAny(list, type) {
			if (!this.limits[type]) {
				return list
			}
			return list.slice(0, this.limits[type])
		},
	},
}
</script>

<style lang="scss" scoped>
$margin: 10px;

.unified-search {
	&__input-wrapper {
		position: sticky;
		// above search results
		z-index: 2;
		top: 0;
		background-color: var(--color-main-background);
	}

	&__input {
		width: calc(100% - 2 * 8px);
		height: 34px;
		margin: 8px;
	}

	&__results {
		&::before {
			content: attr(aria-label);
			display: block;
			color: var(--color-primary);
			font-weight: bold;
			margin: $margin;
		}
	}

	.empty-content {
		margin: 10vh 0;
	}
}

</style>

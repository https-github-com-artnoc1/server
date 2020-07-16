<template>
	<li class="unified-search__result">
		<div class="unified-search__result-icon"
			role="img"
			:class="icon"
			:style="{ backgroundImage: `url(${thumbnailUrl})` }" />
		<div class="unified-search__result-content">
			<h3 class="unified-search__result-line-one" v-text="title" />
			<h4 v-if="subline" class="unified-search__result-line-two" v-text="subline" />
		</div>
		<Actions class="unified-search__result-actions">
			<slot />
		</Actions>
	</li>
</template>

<script>
import Actions from '@nextcloud/vue/dist/Components/Actions'

export default {
	name: 'SearchResult',

	components: {
		Actions,
	},

	props: {
		icon: {
			type: String,
			default: null,
		},
		thumbnailUrl: {
			type: String,
			default: null,
		},
		title: {
			type: String,
			required: true,
		},
		subline: {
			type: String,
			default: null,
		},
		resourceUrl: {
			type: String,
			default: null,
		},
	},
}
</script>

<style lang="scss" scoped>
$clickable-area: 44px;
$margin: 10px;
$margin-right: 2px;

.unified-search__result {
	display: flex;
	height: $clickable-area;
	padding: $margin $margin-right $margin $margin - $margin-right;
	border-bottom: 1px solid var(--color-border);

	// Load more entry,
	&:last-child {
		border-bottom: none;
	}

	&:hover,
	&:focus {
		background-color: var(--color-background-hover);
	}

	* {
		cursor: pointer;
	}

	&-icon {
		overflow: hidden;
		width: $clickable-area;
		height: $clickable-area;
		border-radius: var(--border-radius);
		background-position: center center;
		background-size: cover;
	}

	&-icon,
	&-actions {
		flex: 0 0 $clickable-area;
	}

	&-content {
		display: flex;
		align-items: center;
		flex: 1 1 100%;
		flex-wrap: wrap;
		// Set to minimum and gro from it
		min-width: 0;
		padding: 0 $margin - $margin-right;
	}

	&-line-one,
	&-line-two {
		overflow: hidden;
		flex: 1 1 100%;
		margin: 0;
		white-space: nowrap;
		text-overflow: ellipsis;
		font-size: inherit;
	}
	&-line-two {
		opacity: .7;
		font-size: 14px;
	}
}

</style>

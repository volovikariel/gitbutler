<script lang="ts">
	import AuthorIcon from './AuthorIcon.svelte';
	import type { Author } from '$lib/vbranches/types';

	export let authors: Author[];

	let maximumAuthors = 4;
</script>

<div class="gravatar-icons">
	{#each authors as author, index}
		{#if author.gravatarUrl && index < maximumAuthors}
			<AuthorIcon {author} />
		{:else if index === maximumAuthors}
			<div class="authors-left-counter" title={authors.length - maximumAuthors + ' more authors'}>
				<span class="text-base-10 text-semibold">
					+{authors.length - maximumAuthors}
				</span>
			</div>
		{/if}
	{/each}
</div>

<style lang="postcss">
	.gravatar-icons {
		display: flex;
		align-items: center;
	}

	.authors-left-counter {
		display: flex;
		align-items: center;
		justify-content: center;
		height: var(--space-16);
		padding: 0 var(--space-4);
		background-color: var(--clr-theme-container-light);
		border-radius: var(--space-16);
		margin-left: calc(var(--space-4) * -1);
		outline: 1px solid var(--clr-theme-container-outline-light);

		& span {
			color: var(--clr-theme-scale-ntrl-40);
		}
	}
</style>

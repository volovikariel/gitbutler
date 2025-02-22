<script lang="ts">
	import BranchLabel from './BranchLabel.svelte';
	import BranchLanePopupMenu from './BranchLanePopupMenu.svelte';
	import MergeButton from './MergeButton.svelte';
	import Tag from './Tag.svelte';
	import { clickOutside } from '$lib/clickOutside';
	import Button from '$lib/components/Button.svelte';
	import Icon, { type IconColor } from '$lib/components/Icon.svelte';
	import { normalizeBranchName } from '$lib/utils/branch';
	import * as toasts from '$lib/utils/toasts';
	import { tooltip } from '$lib/utils/tooltip';
	import { open } from '@tauri-apps/api/shell';
	import toast from 'svelte-french-toast';
	import type { BranchService } from '$lib/branches/service';
	import type { GitHubService } from '$lib/github/service';
	import type { PrStatus } from '$lib/github/types';
	import type { BranchController } from '$lib/vbranches/branchController';
	import type { BaseBranch, Branch } from '$lib/vbranches/types';
	import type iconsJson from '../icons/icons.json';
	import { goto } from '$app/navigation';

	export let isUnapplied = false;
	export let branch: Branch;
	export let base: BaseBranch | undefined | null;
	export let branchController: BranchController;
	export let branchService: BranchService;
	export let projectId: string;

	export let githubService: GitHubService;
	$: pr$ = githubService.get(branch.upstreamName);

	let meatballButton: HTMLDivElement;
	let visible = false;
	let container: HTMLDivElement;
	let isApplying = false;
	let isDeleting = false;
	let isMerging = false;
	let isFetching = false;
	let prStatus: PrStatus | undefined;

	function handleBranchNameChange() {
		branchController.updateBranchName(branch.id, branch.name);
	}

	async function fetchPrStatus() {
		isFetching = true;
		try {
			prStatus = await githubService.getStatus($pr$?.targetBranch);
		} catch (e: any) {
			if (!e.message.includes('No commit found')) {
				toasts.error('Failed to update PR status');
				console.error(e);
			}
			prStatus = undefined;
		} finally {
			isFetching = false;
		}

		if (prStatus) scheduleNextPrFetch();
	}

	function scheduleNextPrFetch() {
		if (!prStatus || prStatus.completed) {
			return;
		}
		const startedAt = prStatus.startedAt;
		const secondsAgo = (new Date().getTime() - startedAt.getTime()) / 1000;

		let timeUntilUdate: number | undefined = undefined;
		if (secondsAgo < 600) {
			timeUntilUdate = 30;
		} else if (secondsAgo < 1200) {
			timeUntilUdate = 60;
		} else if (secondsAgo < 3600) {
			timeUntilUdate = 120;
		}

		if (!timeUntilUdate) {
			// Stop polling for status.
			return;
		}

		setTimeout(() => fetchPrStatus(), timeUntilUdate * 1000);
	}

	$: prColor = statusToColor(prStatus);
	$: prIcon = statusToIcon(prStatus);
	$: if ($pr$) fetchPrStatus();

	function statusToColor(status: PrStatus | undefined): IconColor {
		if (!status) return 'neutral';
		if (status && !status.hasChecks) return 'neutral';
		if (status.completed) {
			return status.success ? 'success' : 'error';
		}
		return 'warn';
	}

	function statusToIcon(status: PrStatus | undefined): keyof typeof iconsJson | undefined {
		if (!status) return;
		if (status && !status.hasChecks) return;
		if (status.completed) {
			return status.success ? 'success' : 'error';
		}
		return 'warning';
	}

	function statusToTooltip(status: PrStatus | undefined): string | undefined {
		if (!status) return;
		if (status && !status.hasChecks) return;
		if (status.completed) {
			return status.success ? 'All checks succeeded' : 'Some check(s) have failed';
		}
		return 'Checks are running';
	}

	$: hasIntegratedCommits = branch.commits?.some((b) => b.isIntegrated);
</script>

<div class="header__wrapper">
	<div class="header card" bind:this={container} class:isUnapplied>
		<div class="header__info">
			<div class="header__label">
				<BranchLabel
					bind:name={branch.name}
					on:change={handleBranchNameChange}
					disabled={isUnapplied}
				/>
			</div>
			<div class="header__remote-branch">
				{#if !branch.upstream}
					{#if !branch.active}
						<Tag
							icon="virtual-branch-small"
							color="light"
							help="These changes are stashed away from your working directory."
							reversedDirection>unapplied</Tag
						>
					{:else if hasIntegratedCommits}
						<Tag
							icon="removed-branch-small"
							color="success"
							help="These changes have been integrated upstream, update your workspace to make this lane disappear."
							reversedDirection>integrated</Tag
						>
					{:else}
						<Tag
							icon="virtual-branch-small"
							color="light"
							help="These changes are in your working directory."
							reversedDirection>virtual</Tag
						>
					{/if}
					{#if !isUnapplied}
						<Tag
							disabled
							help="Branch name that will be used when pushing. You can change it from the lane menu."
						>
							origin/{branch.upstreamName
								? branch.upstreamName
								: normalizeBranchName(branch.name)}</Tag
						>
					{/if}
				{:else}
					<Tag
						color="dark"
						icon="remote-branch-small"
						help="At least some of your changes have been pushed"
						reversedDirection>remote</Tag
					>
					<Tag
						icon="open-link"
						color="ghost"
						border
						clickable
						shrinkable
						on:click={(e) => {
							const url = base?.branchUrl(branch.upstream?.name);
							if (url) open(url);
							e.preventDefault();
							e.stopPropagation();
						}}
					>
						origin/{branch.upstreamName}
					</Tag>
					{#if $pr$?.htmlUrl}
						<Tag
							icon="pr-small"
							color="ghost"
							border
							clickable
							on:click={(e) => {
								const url = $pr$?.htmlUrl;
								if (url) open(url);
								e.preventDefault();
								e.stopPropagation();
							}}
						>
							View PR
						</Tag>
					{/if}
					{#if prIcon}
						<div
							class="pr-status"
							role="button"
							tabindex="0"
							on:click={fetchPrStatus}
							on:keypress={fetchPrStatus}
							use:tooltip={statusToTooltip(prStatus)}
						>
							{#if isFetching}
								<Icon name="spinner" />
							{:else}
								<Icon name={prIcon} color={prColor} />
							{/if}
						</div>
					{/if}
				{/if}
				{#await branch.isMergeable then isMergeable}
					{#if !isMergeable}
						<Tag
							icon="locked-small"
							color="warning"
							help="Applying this branch will add merge conflict markers that you will have to resolve"
						>
							Conflict
						</Tag>
					{/if}
				{/await}
			</div>
			<div class="draggable" data-drag-handle>
				<Icon name="draggable-narrow" />
			</div>
		</div>
		<div class="header__actions">
			<div class="header__buttons">
				{#if branch.active}
					{#if branch.selectedForChanges}
						<Button
							help="New changes will land here"
							icon="target"
							notClickable
							disabled={isUnapplied}>Default branch</Button
						>
					{:else}
						<Button
							help="When selected, new changes will land here"
							icon="target"
							kind="outlined"
							color="neutral"
							disabled={isUnapplied}
							on:click={async () => {
								await branchController.setSelectedForChanges(branch.id);
							}}
						>
							Set as default
						</Button>
					{/if}
				{/if}
				<!-- We can't show the merge button until we've waited for checks

                We use a octokit.checks.listForRef to find checks running for a PR, but right after
                creation this request succeeds but returns an empty array. So we need a better way
                determining "no checks will run for this PR" such that we can show the merge button
                immediately.  -->
				{#if $pr$ && !isFetching && (!prStatus || prStatus?.success)}
					<MergeButton
						{projectId}
						disabled={isUnapplied || !$pr$}
						loading={isMerging}
						help="Merge pull request and refresh"
						on:click={async (e) => {
							isMerging = true;
							const method = e.detail.method;
							try {
								if ($pr$) {
									await githubService.merge($pr$.number, method);
								}
							} catch {
								// TODO: Should we show the error from GitHub?
								toasts.error('Failed to merge pull request');
							} finally {
								isMerging = false;
								await fetchPrStatus();
								await branchService.reloadVirtualBranches();
							}
						}}
					/>
				{/if}
			</div>
			<div class="relative" bind:this={meatballButton}>
				{#if isUnapplied}
					<Button
						help="Deletes the local virtual branch (only)"
						icon="bin-small"
						color="neutral"
						kind="outlined"
						loading={isDeleting}
						on:click={async () => {
							isDeleting = true;
							try {
								await branchController.deleteBranch(branch.id);
								goto(`/${projectId}/board`);
							} catch (e) {
								const err = 'Failed to delete branch';
								toasts.error(err);
								console.error(err, e);
							} finally {
								isDeleting = false;
							}
						}}
					>
						Delete
					</Button>
					<Button
						help="Restores these changes into your working directory"
						icon="plus-small"
						color="primary"
						kind="outlined"
						loading={isApplying}
						on:click={async () => {
							isApplying = true;
							try {
								await branchController.applyBranch(branch.id);
								goto(`/${projectId}/board`);
							} catch (e) {
								const err = 'Failed to apply branch';
								toast.error(err);
								console.error(err, e);
							} finally {
								isApplying = false;
							}
						}}
					>
						Apply
					</Button>
				{:else}
					<Button
						icon="kebab"
						kind="outlined"
						color="neutral"
						on:click={() => (visible = !visible)}
					/>
					<div
						class="branch-popup-menu"
						use:clickOutside={{
							trigger: meatballButton,
							handler: () => (visible = false)
						}}
					>
						<BranchLanePopupMenu
							{branchController}
							{branch}
							{projectId}
							{isUnapplied}
							bind:visible
							on:action
						/>
					</div>
				{/if}
			</div>
		</div>
	</div>
	<div class="header__top-overlay" data-remove-from-draggable data-tauri-drag-region />
</div>

<style lang="postcss">
	.header__wrapper {
		z-index: 10;
		position: sticky;
		top: var(--space-12);
	}
	.header {
		z-index: 2;
		position: relative;
		flex-direction: column;
		gap: var(--space-2);

		&:hover {
			& .draggable {
				opacity: 1;
			}
		}
		&.isUnapplied {
			background: var(--clr-theme-container-pale);
		}
	}
	.header__top-overlay {
		z-index: 1;
		position: absolute;
		top: calc(var(--space-16) * -1);
		left: 0;
		width: 100%;
		height: var(--space-20);
		background: var(--target-branch-background);
		/* background-color: red; */
	}
	.header__info {
		display: flex;
		flex-direction: column;
		transition: margin var(--transition-slow);
		padding: var(--space-12);
		gap: var(--space-10);
	}
	.header__actions {
		display: flex;
		gap: var(--space-4);
		background: var(--clr-theme-container-pale);
		padding: var(--space-12);
		justify-content: space-between;
		border-radius: 0 0 var(--radius-m) var(--radius-m);
		user-select: none;
	}
	.isUnapplied .header__actions {
		background: var(--clr-theme-container-dim);
	}
	.header__buttons {
		display: flex;
		position: relative;
		gap: var(--space-4);
	}
	.header__label {
		display: flex;
		flex-grow: 1;
		align-items: center;
		gap: var(--space-4);
	}
	.draggable {
		position: absolute;
		right: var(--space-4);
		top: var(--space-6);
		opacity: 0;
		display: flex;
		cursor: grab;
		color: var(--clr-theme-scale-ntrl-50);
		transition:
			opacity var(--transition-slow),
			color var(--transition-slow);

		&:hover {
			color: var(--clr-theme-scale-ntrl-40);
		}
	}

	.branch-popup-menu {
		position: absolute;
		top: calc(100% + var(--space-4));
		right: 0;
		z-index: 10;
	}

	.header__remote-branch {
		color: var(--clr-theme-scale-ntrl-50);
		padding-left: var(--space-4);
		display: flex;
		gap: var(--space-4);
		text-overflow: ellipsis;
		overflow-x: hidden;
		white-space: nowrap;
		align-items: center;
	}

	.pr-status {
		cursor: default;
	}
</style>

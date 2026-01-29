<script lang="ts">
	import { DropdownMenu } from 'bits-ui';
	import { getContext } from 'svelte';
	import type { Writable } from 'svelte/store';
	import type { i18n as i18nType } from 'i18next';

	import Tooltip from '$lib/components/common/Tooltip.svelte';
	import Spinner from '$lib/components/common/Spinner.svelte';
	import { flyAndScale } from '$lib/utils/transitions';
	import { settings } from '$lib/stores';

	const i18n = getContext<Writable<i18nType>>('i18n');

	export let isLastMessage = true;
	export let showContinue = true;
	export let showSummarize = true;
	export let showBranch = true;
	export let summarizing = false;
	export let branchingChat = false;

	export let onContinue: Function = () => {};
	export let onSummarize: Function = () => {};
	export let onBranch: Function = () => {};

	let show = false;

	const menuItemClass =
		'flex gap-2 items-center px-3 py-1.5 text-sm cursor-pointer hover:bg-gray-50 dark:hover:bg-gray-800 rounded-xl';

	const closeMenu = () => {
		show = false;
	};
</script>

{#if showContinue || showSummarize || showBranch}
	<DropdownMenu.Root bind:open={show} closeFocus={false} typeahead={false}>
		<DropdownMenu.Trigger>
			<Tooltip content={$i18n.t('更多')} placement="bottom">
				<button
					type="button"
					aria-label={$i18n.t('更多')}
					class="{isLastMessage || ($settings?.highContrastMode ?? false)
						? 'visible'
						: 'invisible group-hover:visible'} p-1 hover:bg-black/5 dark:hover:bg-white/5 rounded-lg dark:hover:text-white hover:text-black transition"
				>
					<svg
						xmlns="http://www.w3.org/2000/svg"
						viewBox="0 0 24 24"
						fill="currentColor"
						aria-hidden="true"
						class="size-4"
					>
						<circle cx="5" cy="12" r="2" />
						<circle cx="12" cy="12" r="2" />
						<circle cx="19" cy="12" r="2" />
					</svg>
				</button>
			</Tooltip>
		</DropdownMenu.Trigger>

		<DropdownMenu.Content
			class="w-full max-w-[200px] rounded-2xl px-1 py-1 border border-gray-100 dark:border-gray-800 z-50 bg-white dark:bg-gray-850 dark:text-white shadow-lg transition"
			sideOffset={-2}
			side="bottom"
			align="end"
			transition={flyAndScale}
		>
			{#if showContinue}
				<DropdownMenu.Item
					type="button"
					class={menuItemClass}
					on:click={(e) => {
						e.stopPropagation();
						e.preventDefault();
						onContinue();
						closeMenu();
					}}
				>
					<svg
						xmlns="http://www.w3.org/2000/svg"
						fill="none"
						viewBox="0 0 24 24"
						stroke-width="2"
						stroke="currentColor"
						aria-hidden="true"
						class="size-4"
					>
						<path
							stroke-linecap="round"
							stroke-linejoin="round"
							d="M5.25 5.653c0-.856.917-1.398 1.667-.986l11.54 6.347a1.125 1.125 0 0 1 0 1.972l-11.54 6.347a1.125 1.125 0 0 1-1.667-.986V5.653Z"
						/>
					</svg>
					<div class="flex items-center">{$i18n.t('继续生成')}</div>
				</DropdownMenu.Item>
			{/if}

			{#if showSummarize}
				<DropdownMenu.Item
					type="button"
					class={`${menuItemClass} ${summarizing ? 'opacity-60 cursor-not-allowed' : ''}`}
					on:click={(e) => {
						e.stopPropagation();
						e.preventDefault();
						if (summarizing) return;
						onSummarize();
						closeMenu();
					}}
				>
					{#if summarizing}
						<Spinner className="size-4" />
					{:else}
						<svg
							xmlns="http://www.w3.org/2000/svg"
							fill="none"
							viewBox="0 0 24 24"
							stroke-width="2"
							stroke="currentColor"
							aria-hidden="true"
							class="size-4"
						>
							<path
								stroke-linecap="round"
								stroke-linejoin="round"
								d="M19.5 14.25v-2.625a3.375 3.375 0 0 0-3.375-3.375h-1.5A1.125 1.125 0 0 1 13.5 7.125v-1.5a3.375 3.375 0 0 0-3.375-3.375H8.25m0 12.75h7.5m-7.5 3H12M10.5 2.25H5.625c-.621 0-1.125.504-1.125 1.125v17.25c0 .621.504 1.125 1.125 1.125h12.75c.621 0 1.125-.504 1.125-1.125V11.25a9 9 0 0 0-9-9Z"
							/>
						</svg>
					{/if}
					<div class="flex items-center">{$i18n.t('总结所有内容')}</div>
				</DropdownMenu.Item>
			{/if}

			{#if showBranch}
				<DropdownMenu.Item
					type="button"
					class={`${menuItemClass} ${branchingChat ? 'opacity-60 cursor-not-allowed' : ''}`}
					on:click={(e) => {
						e.stopPropagation();
						e.preventDefault();
						if (branchingChat) return;
						onBranch();
						closeMenu();
					}}
				>
					{#if branchingChat}
						<Spinner className="size-4" />
					{:else}
						<svg
							xmlns="http://www.w3.org/2000/svg"
							fill="none"
							viewBox="0 0 24 24"
							stroke-width="2"
							stroke="currentColor"
							aria-hidden="true"
							class="size-4"
						>
							<path
								stroke-linecap="round"
								stroke-linejoin="round"
								d="M4 12h8l7-6m0 0h-3m3 0v3M12 12l7 6m0 0h-3m3 0v-3"
							/>
						</svg>
					{/if}
					<div class="flex items-center">{$i18n.t('分支到新对话')}</div>
				</DropdownMenu.Item>
			{/if}
		</DropdownMenu.Content>
	</DropdownMenu.Root>
{/if}

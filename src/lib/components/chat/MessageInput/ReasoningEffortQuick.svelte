<script lang="ts">
	import { DropdownMenu } from 'bits-ui';
	import { getContext } from 'svelte';
	import { settings } from '$lib/stores';
	import { updateUserSettings } from '$lib/apis/users';
	import Tooltip from '$lib/components/common/Tooltip.svelte';
	import { flyAndScale } from '$lib/utils/transitions';
	import LightBulbAuto from '$lib/components/icons/LightBulbAuto.svelte';
	import LightBulbOff from '$lib/components/icons/LightBulbOff.svelte';
	import LightBulbMin from '$lib/components/icons/LightBulbMin.svelte';
	import LightBulbLow from '$lib/components/icons/LightBulbLow.svelte';
	import LightBulbMedium from '$lib/components/icons/LightBulbMedium.svelte';
	import LightBulbHigh from '$lib/components/icons/LightBulbHigh.svelte';
	import LightBulbMax from '$lib/components/icons/LightBulbMax.svelte';

	const i18n = getContext('i18n');

	let show = false;

	const options = [
		{ value: null, label: 'Default', cn: '默认', icon: LightBulbAuto },
		{ value: 'none', label: 'None', cn: '关闭', icon: LightBulbOff },
		{ value: 'minimal', label: 'Minimal', cn: '最低', icon: LightBulbMin },
		{ value: 'low', label: 'Low', cn: '浮想', icon: LightBulbLow },
		{ value: 'medium', label: 'Medium', cn: '斟酌', icon: LightBulbMedium },
		{ value: 'high', label: 'High', cn: '沉思', icon: LightBulbHigh },
		{ value: 'xhigh', label: 'Xhigh', cn: '最高', icon: LightBulbMax }
	];

	const menuItemClass =
		'flex gap-2 items-center px-3 py-1.5 text-sm cursor-pointer hover:bg-gray-50 dark:hover:bg-gray-800 rounded-xl';

	$: currentValue = $settings?.params?.reasoning_effort ?? null;
	$: activeOption = options.find((opt) => opt.value === currentValue) ?? options[0];

	async function selectOption(option: typeof options[0]) {
		const newParams = { ...($settings?.params || {}), reasoning_effort: option.value };
		await settings.set({ ...$settings, params: newParams });
		await updateUserSettings(localStorage.token, { ui: $settings });
		show = false;
	}
</script>

<DropdownMenu.Root bind:open={show} closeFocus={false} typeahead={false}>
	<DropdownMenu.Trigger>
		<Tooltip content={`${$i18n.t('Reasoning Effort')}: ${activeOption.cn}`} placement="top">
			<button
				type="button"
				aria-label={$i18n.t('Reasoning Effort')}
				class="bg-transparent hover:bg-gray-100 text-gray-700 dark:text-white dark:hover:bg-gray-800 rounded-full size-8 flex justify-center items-center outline-hidden focus:outline-hidden"
			>
				<svelte:component this={activeOption.icon} className="size-4" strokeWidth="1.5" />
			</button>
		</Tooltip>
	</DropdownMenu.Trigger>

	<DropdownMenu.Content
		class="w-full max-w-[200px] rounded-2xl px-1 py-1 border border-gray-100 dark:border-gray-800 z-50 bg-white dark:bg-gray-850 dark:text-white shadow-lg transition"
		sideOffset={8}
		side="top"
		align="start"
		transition={flyAndScale}
	>
		{#each options as option}
			<DropdownMenu.Item
				type="button"
				class={`${menuItemClass} ${option.value === currentValue ? 'bg-gray-50 dark:bg-gray-800' : ''}`}
				on:click={(event) => {
					event.stopPropagation();
					event.preventDefault();
					selectOption(option);
				}}
			>
				<svelte:component this={option.icon} className="size-4" strokeWidth="1.5" />
				<div class="flex items-center gap-2">
					<span>{option.cn}</span>
					<span class="text-gray-400 text-xs">{option.label}</span>
				</div>
			</DropdownMenu.Item>
		{/each}
	</DropdownMenu.Content>
</DropdownMenu.Root>

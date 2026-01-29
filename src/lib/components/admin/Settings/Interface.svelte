<script lang="ts">
	// @ts-ignore
	import fileSaver from 'file-saver';
	const { saveAs } = fileSaver;

	// @ts-ignore
	import { v4 as uuidv4 } from 'uuid';
	import { toast } from 'svelte-sonner';
	import { slide } from 'svelte/transition';
	import { quintOut } from 'svelte/easing';

	import { getBackendConfig, getModels, getTaskConfig, updateTaskConfig } from '$lib/apis';
	import { setDefaultPromptSuggestions } from '$lib/apis/configs';
	import { config, settings, user, tasksConfigCache, bannersCache } from '$lib/stores';
	import { createEventDispatcher, onMount, getContext } from 'svelte';

	import { banners as _banners } from '$lib/stores';
	import type { Banner } from '$lib/types';

	import { getBaseModels } from '$lib/apis/models';
	import { getBanners, setBanners } from '$lib/apis/configs';

	import Tooltip from '$lib/components/common/Tooltip.svelte';
	import Switch from '$lib/components/common/Switch.svelte';
	import Textarea from '$lib/components/common/Textarea.svelte';
	import Spinner from '$lib/components/common/Spinner.svelte';
	import Banners from './Interface/Banners.svelte';
	import PromptSuggestions from '$lib/components/workspace/Models/PromptSuggestions.svelte';
	import ChevronDown from '$lib/components/icons/ChevronDown.svelte';

	const dispatch = createEventDispatcher();

	import type { Writable } from 'svelte/store';
	const i18n: Writable<any> = getContext('i18n');

	// 折叠状态
	let expandedSections = {
		tasks: false,
		ui: false,
		conversation: false,
		input: false,
		admin: false
	};

	let taskConfig: any = {
		TASK_MODEL: '',
		TASK_MODEL_EXTERNAL: '',
		ENABLE_TITLE_GENERATION: true,
		TITLE_GENERATION_PROMPT_TEMPLATE: '',
		ENABLE_FOLLOW_UP_GENERATION: true,
		FOLLOW_UP_GENERATION_PROMPT_TEMPLATE: '',
		IMAGE_PROMPT_GENERATION_PROMPT_TEMPLATE: '',
		ENABLE_AUTOCOMPLETE_GENERATION: true,
		AUTOCOMPLETE_GENERATION_INPUT_MAX_LENGTH: -1,
		TAGS_GENERATION_PROMPT_TEMPLATE: '',
		ENABLE_TAGS_GENERATION: true,
		ENABLE_SEARCH_QUERY_GENERATION: true,
		ENABLE_RETRIEVAL_QUERY_GENERATION: true,
		QUERY_GENERATION_PROMPT_TEMPLATE: '',
		TOOLS_FUNCTION_CALLING_PROMPT_TEMPLATE: '',
		VOICE_MODE_PROMPT_TEMPLATE: ''
	};

	let promptSuggestions: any[] = [];
	let banners: Banner[] = [];

	const updateInterfaceHandler = async () => {
		taskConfig = await updateTaskConfig(localStorage.token, taskConfig);
		// Update cache after saving
		tasksConfigCache.set(taskConfig);

		promptSuggestions = promptSuggestions.filter((p) => p.content !== '');
		// @ts-ignore
		promptSuggestions = await setDefaultPromptSuggestions(localStorage.token, promptSuggestions);
		await updateBanners();

		await config.set(await getBackendConfig());
	};

	const updateBanners = async () => {
		const updatedBanners = await setBanners(localStorage.token, banners);
		_banners.set(updatedBanners);
		// Update cache after saving
		bannersCache.set(updatedBanners);
	};

	let workspaceModels: any = null;
	let baseModels: any = null;

	let models: any = null;

	const init = async () => {
		// Use cached taskConfig if available
		if ($tasksConfigCache) {
			taskConfig = JSON.parse(JSON.stringify($tasksConfigCache));
		} else {
			taskConfig = await getTaskConfig(localStorage.token);
			tasksConfigCache.set(taskConfig);
		}

		promptSuggestions = $config?.default_prompt_suggestions ?? [];

		// Use cached banners if available
		if ($bannersCache) {
			banners = JSON.parse(JSON.stringify($bannersCache));
		} else {
			banners = await getBanners(localStorage.token);
			bannersCache.set(banners);
		}

		workspaceModels = await getBaseModels(localStorage.token);
		baseModels = await getModels(localStorage.token, null, false);

		models = baseModels.map((m: any) => {
			const workspaceModel = workspaceModels.find((wm: any) => wm.id === m.id);

			if (workspaceModel) {
				return {
					...m,
					...workspaceModel
				};
			} else {
				return {
					...m,
					id: m.id,
					name: m.name,

					is_active: true
				};
			}
		});

		console.debug('models', models);
	};

	onMount(async () => {
		await init();
	});
</script>

{#if models !== null && taskConfig}
	<form
		class="flex flex-col h-full justify-between space-y-3 text-sm"
		on:submit|preventDefault={() => {
			updateInterfaceHandler();
			dispatch('save');
		}}
	>
		<div class="overflow-y-scroll scrollbar-hidden h-full pr-2 space-y-3">
			<!-- 任务 Tasks -->
			<div class="max-w-5xl mx-auto rounded-xl border border-gray-200 dark:border-gray-800">
				<button
					type="button"
					class="w-full flex items-center justify-between px-5 py-4 text-left"
					on:click={() => (expandedSections.tasks = !expandedSections.tasks)}
				>
					<div class="flex items-center gap-3">
						<svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="1.5" stroke="currentColor" class="size-5 text-gray-500">
							<path stroke-linecap="round" stroke-linejoin="round" d="M19.5 14.25v-2.625a3.375 3.375 0 0 0-3.375-3.375h-1.5A1.125 1.125 0 0 1 13.5 7.125v-1.5a3.375 3.375 0 0 0-3.375-3.375H8.25m0 12.75h7.5m-7.5 3H12M10.5 2.25H5.625c-.621 0-1.125.504-1.125 1.125v17.25c0 .621.504 1.125 1.125 1.125h12.75c.621 0 1.125-.504 1.125-1.125V11.25a9 9 0 0 0-9-9Z" />
						</svg>
						<span class="text-base font-medium text-gray-900 dark:text-gray-100">{$i18n.t('任务')}</span>
					</div>
					<div class="transform transition-transform duration-200 {expandedSections.tasks ? 'rotate-180' : ''}">
						<ChevronDown className="size-5 text-gray-400" />
					</div>
				</button>

				{#if expandedSections.tasks}
					<div transition:slide={{ duration: 200, easing: quintOut }} class="px-5 pb-5 space-y-4">
						<!-- Task Model -->
						<div class="bg-white dark:bg-gray-900 rounded-lg p-4 border border-gray-100 dark:border-gray-800">
							<div class="mb-3 font-medium flex items-center">
								<div class="text-sm mr-1">{$i18n.t('Task Model')}</div>
								<Tooltip content={$i18n.t('A task model is used when performing tasks such as generating titles for chats and web search queries')}>
									<svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="1.5" stroke="currentColor" class="size-3.5">
										<path stroke-linecap="round" stroke-linejoin="round" d="m11.25 11.25.041-.02a.75.75 0 0 1 1.063.852l-.708 2.836a.75.75 0 0 0 1.063.853l.041-.021M21 12a9 9 0 1 1-18 0 9 9 0 0 1 18 0Zm-9-3.75h.008v.008H12V8.25Z" />
									</svg>
								</Tooltip>
							</div>
							<div class="flex w-full gap-3">
								<div class="flex-1">
									<div class="text-xs mb-1 text-gray-500">{$i18n.t('Local Task Model')}</div>
									<select
										class="w-full rounded-lg py-2 px-3 text-sm bg-gray-50 dark:text-gray-300 dark:bg-gray-850 outline-hidden border border-gray-200 dark:border-gray-700"
										bind:value={taskConfig.TASK_MODEL}
										placeholder={$i18n.t('Select a model')}
										on:change={() => {
											if (taskConfig.TASK_MODEL) {
												const model = models.find((m) => m.id === taskConfig.TASK_MODEL);
												if (model) {
													if (model?.access_control !== null) {
														toast.error($i18n.t('This model is not publicly available. Please select another model.'));
													}
													taskConfig.TASK_MODEL = model.id;
												} else {
													taskConfig.TASK_MODEL = '';
												}
											}
										}}
									>
										<option value="" selected>{$i18n.t('Current Model')}</option>
										{#each models as model}
											<option value={model.id}>{model.name} {model?.connection_type === 'local' ? `(${$i18n.t('Local')})` : ''}</option>
										{/each}
									</select>
								</div>
								<div class="flex-1">
									<div class="text-xs mb-1 text-gray-500">{$i18n.t('External Task Model')}</div>
									<select
										class="w-full rounded-lg py-2 px-3 text-sm bg-gray-50 dark:text-gray-300 dark:bg-gray-850 outline-hidden border border-gray-200 dark:border-gray-700"
										bind:value={taskConfig.TASK_MODEL_EXTERNAL}
										placeholder={$i18n.t('Select a model')}
										on:change={() => {
											if (taskConfig.TASK_MODEL_EXTERNAL) {
												const model = models.find((m) => m.id === taskConfig.TASK_MODEL_EXTERNAL);
												if (model) {
													if (model?.access_control !== null) {
														toast.error($i18n.t('This model is not publicly available. Please select another model.'));
													}
													taskConfig.TASK_MODEL_EXTERNAL = model.id;
												} else {
													taskConfig.TASK_MODEL_EXTERNAL = '';
												}
											}
										}}
									>
										<option value="" selected>{$i18n.t('Current Model')}</option>
										{#each models as model}
											<option value={model.id}>{model.name} {model?.connection_type === 'local' ? `(${$i18n.t('Local')})` : ''}</option>
										{/each}
									</select>
								</div>
							</div>
						</div>

						<!-- Title Generation -->
						<div class="bg-white dark:bg-gray-900 rounded-lg p-4 border border-gray-100 dark:border-gray-800">
							<div class="flex items-center justify-between mb-3">
								<div class="text-sm font-medium">{$i18n.t('Title Generation')}</div>
								<Switch bind:state={taskConfig.ENABLE_TITLE_GENERATION} />
							</div>
							{#if taskConfig.ENABLE_TITLE_GENERATION}
								<div>
									<div class="mb-1 text-xs text-gray-500">{$i18n.t('Title Generation Prompt')}</div>
									<Textarea bind:value={taskConfig.TITLE_GENERATION_PROMPT_TEMPLATE} placeholder={$i18n.t('Leave empty to use the default prompt, or enter a custom prompt')} />
								</div>
							{/if}
						</div>

						<!-- Voice Mode -->
						<div class="bg-white dark:bg-gray-900 rounded-lg p-4 border border-gray-100 dark:border-gray-800">
							<div class="flex items-center justify-between mb-3">
								<div class="text-sm font-medium">{$i18n.t('Voice Mode Custom Prompt')}</div>
								<Switch state={taskConfig.VOICE_MODE_PROMPT_TEMPLATE != null} on:change={(e) => { if (e.detail) { taskConfig.VOICE_MODE_PROMPT_TEMPLATE = ''; } else { taskConfig.VOICE_MODE_PROMPT_TEMPLATE = null; } }} />
							</div>
							{#if taskConfig.VOICE_MODE_PROMPT_TEMPLATE != null}
								<div>
									<div class="mb-1 text-xs text-gray-500">{$i18n.t('Voice Mode Prompt')}</div>
									<Textarea bind:value={taskConfig.VOICE_MODE_PROMPT_TEMPLATE} placeholder={$i18n.t('Leave empty to use the default prompt, or enter a custom prompt')} />
								</div>
							{/if}
						</div>

						<!-- Follow Up Generation -->
						<div class="bg-white dark:bg-gray-900 rounded-lg p-4 border border-gray-100 dark:border-gray-800">
							<div class="flex items-center justify-between mb-3">
								<div class="text-sm font-medium">{$i18n.t('Follow Up Generation')}</div>
								<Switch bind:state={taskConfig.ENABLE_FOLLOW_UP_GENERATION} />
							</div>
							{#if taskConfig.ENABLE_FOLLOW_UP_GENERATION}
								<div>
									<div class="mb-1 text-xs text-gray-500">{$i18n.t('Follow Up Generation Prompt')}</div>
									<Textarea bind:value={taskConfig.FOLLOW_UP_GENERATION_PROMPT_TEMPLATE} placeholder={$i18n.t('Leave empty to use the default prompt, or enter a custom prompt')} />
								</div>
							{/if}
						</div>

						<!-- Tags Generation -->
						<div class="bg-white dark:bg-gray-900 rounded-lg p-4 border border-gray-100 dark:border-gray-800">
							<div class="flex items-center justify-between mb-3">
								<div class="text-sm font-medium">{$i18n.t('Tags Generation')}</div>
								<Switch bind:state={taskConfig.ENABLE_TAGS_GENERATION} />
							</div>
							{#if taskConfig.ENABLE_TAGS_GENERATION}
								<div>
									<div class="mb-1 text-xs text-gray-500">{$i18n.t('Tags Generation Prompt')}</div>
									<Textarea bind:value={taskConfig.TAGS_GENERATION_PROMPT_TEMPLATE} placeholder={$i18n.t('Leave empty to use the default prompt, or enter a custom prompt')} />
								</div>
							{/if}
						</div>

						<!-- Query Generation -->
						<div class="bg-white dark:bg-gray-900 rounded-lg p-4 border border-gray-100 dark:border-gray-800">
							<div class="flex items-center justify-between mb-3">
								<div class="text-sm font-medium">{$i18n.t('Query Generation')}</div>
								<Switch bind:state={taskConfig.ENABLE_SEARCH_QUERY_GENERATION} />
							</div>
							<div class="space-y-2 mb-3">
								<div class="flex items-center justify-between">
									<div class="text-xs text-gray-500">{$i18n.t('Retrieval Query Generation')}</div>
									<Switch bind:state={taskConfig.ENABLE_RETRIEVAL_QUERY_GENERATION} />
								</div>
								<div class="flex items-center justify-between">
									<div class="text-xs text-gray-500">{$i18n.t('Web Search Query Generation')}</div>
									<Switch bind:state={taskConfig.ENABLE_SEARCH_QUERY_GENERATION} />
								</div>
							</div>
							<div>
								<div class="mb-1 text-xs text-gray-500">{$i18n.t('Query Generation Prompt')}</div>
								<Textarea bind:value={taskConfig.QUERY_GENERATION_PROMPT_TEMPLATE} placeholder={$i18n.t('Leave empty to use the default prompt, or enter a custom prompt')} />
							</div>
						</div>

						<!-- Autocomplete Generation -->
						<div class="bg-white dark:bg-gray-900 rounded-lg p-4 border border-gray-100 dark:border-gray-800">
							<div class="flex items-center justify-between mb-3">
								<div class="text-sm font-medium">{$i18n.t('Autocomplete Generation')}</div>
								<Tooltip content={$i18n.t('Enable autocomplete generation for chat messages')}>
									<Switch bind:state={taskConfig.ENABLE_AUTOCOMPLETE_GENERATION} />
								</Tooltip>
							</div>
							{#if taskConfig.ENABLE_AUTOCOMPLETE_GENERATION}
								<div>
									<div class="mb-1 text-xs text-gray-500">{$i18n.t('Autocomplete Generation Input Max Length')}</div>
									<input class="w-full rounded-lg py-2 px-3 text-sm bg-gray-50 dark:bg-gray-850 outline-hidden border border-gray-200 dark:border-gray-700" bind:value={taskConfig.AUTOCOMPLETE_GENERATION_INPUT_MAX_LENGTH} placeholder={$i18n.t('-1 for no limit, or a positive integer for a specific limit')} />
								</div>
							{/if}
						</div>

						<!-- Image Prompt Generation -->
						<div class="bg-white dark:bg-gray-900 rounded-lg p-4 border border-gray-100 dark:border-gray-800">
							<div class="text-sm font-medium mb-3">{$i18n.t('Image Prompt Generation')}</div>
							<div>
								<div class="mb-1 text-xs text-gray-500">{$i18n.t('Image Prompt Generation Prompt')}</div>
								<Textarea bind:value={taskConfig.IMAGE_PROMPT_GENERATION_PROMPT_TEMPLATE} placeholder={$i18n.t('Leave empty to use the default prompt, or enter a custom prompt')} />
							</div>
						</div>

						<!-- Tools Function Calling -->
						<div class="bg-white dark:bg-gray-900 rounded-lg p-4 border border-gray-100 dark:border-gray-800">
							<div class="text-sm font-medium mb-3">{$i18n.t('Tools Function Calling')}</div>
							<div>
								<div class="mb-1 text-xs text-gray-500">{$i18n.t('Tools Function Calling Prompt')}</div>
								<Textarea bind:value={taskConfig.TOOLS_FUNCTION_CALLING_PROMPT_TEMPLATE} placeholder={$i18n.t('Leave empty to use the default prompt, or enter a custom prompt')} />
							</div>
						</div>
					</div>
				{/if}
			</div>

			<!-- 用户界面 UI -->
			<div class="max-w-5xl mx-auto rounded-xl border border-gray-200 dark:border-gray-800">
				<button
					type="button"
					class="w-full flex items-center justify-between px-5 py-4 text-left"
					on:click={() => (expandedSections.ui = !expandedSections.ui)}
				>
					<div class="flex items-center gap-3">
						<svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="1.5" stroke="currentColor" class="size-5 text-gray-500">
							<path stroke-linecap="round" stroke-linejoin="round" d="M9 17.25v1.007a3 3 0 0 1-.879 2.122L7.5 21h9l-.621-.621A3 3 0 0 1 15 18.257V17.25m6-12V15a2.25 2.25 0 0 1-2.25 2.25H5.25A2.25 2.25 0 0 1 3 15V5.25m18 0A2.25 2.25 0 0 0 18.75 3H5.25A2.25 2.25 0 0 0 3 5.25m18 0V12a2.25 2.25 0 0 1-2.25 2.25H5.25A2.25 2.25 0 0 1 3 12V5.25" />
						</svg>
						<span class="text-base font-medium text-gray-900 dark:text-gray-100">{$i18n.t('用户界面')}</span>
					</div>
					<div class="transform transition-transform duration-200 {expandedSections.ui ? 'rotate-180' : ''}">
						<ChevronDown className="size-5 text-gray-400" />
					</div>
				</button>

				{#if expandedSections.ui}
					<div transition:slide={{ duration: 200, easing: quintOut }} class="px-5 pb-5 space-y-4">
						<!-- Banners -->
						<div class="bg-white dark:bg-gray-900 rounded-lg p-4 border border-gray-100 dark:border-gray-800">
							<div class="flex w-full justify-between items-center mb-3">
								<div class="text-sm font-medium">{$i18n.t('Banners')}</div>
								<button
									class="p-1.5 rounded-lg hover:bg-gray-100 dark:hover:bg-gray-800 transition"
									type="button"
									aria-label="Add Banner"
									on:click={() => {
										if (banners.length === 0 || banners.at(-1)?.content !== '') {
											banners = [
												...banners,
												{
													id: uuidv4(),
													type: '',
													title: '',
													content: '',
													dismissible: true,
													timestamp: Math.floor(Date.now() / 1000)
												}
											];
										}
									}}
								>
									<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 20 20" fill="currentColor" class="w-4 h-4">
										<path d="M10.75 4.75a.75.75 0 00-1.5 0v4.5h-4.5a.75.75 0 000 1.5h4.5v4.5a.75.75 0 001.5 0v-4.5h4.5a.75.75 0 000-1.5h-4.5v-4.5z" />
									</svg>
								</button>
							</div>
							<Banners bind:banners />
						</div>

						<!-- Prompt Suggestions -->
						{#if $user?.role === 'admin'}
							<div class="bg-white dark:bg-gray-900 rounded-lg p-4 border border-gray-100 dark:border-gray-800">
								<PromptSuggestions bind:promptSuggestions />
								{#if promptSuggestions.length > 0}
									<div class="text-xs text-gray-500 mt-2">
										{$i18n.t('Adjusting these settings will apply changes universally to all users.')}
									</div>
								{/if}
							</div>
						{/if}
					</div>
				{/if}
			</div>
		</div>

		<div class="flex justify-end text-sm font-medium">
			<button
				class="px-3.5 py-1.5 text-sm font-medium bg-black hover:bg-gray-900 text-white dark:bg-white dark:text-black dark:hover:bg-gray-100 transition rounded-full"
				type="submit"
			>
				{$i18n.t('Save')}
			</button>
		</div>
	</form>
{:else}
	<div class=" h-full w-full flex justify-center items-center">
		<Spinner className="size-5" />
	</div>
{/if}

<script lang="ts">
	import type { Writable } from 'svelte/store';
	import { toast } from 'svelte-sonner';
	import { slide } from 'svelte/transition';
	import { quintOut } from 'svelte/easing';

	import { onMount, getContext, createEventDispatcher } from 'svelte';

	const dispatch = createEventDispatcher();

	import {
		getQuerySettings,
		updateQuerySettings,
		resetVectorDB,
		getEmbeddingConfig,
		updateEmbeddingConfig,
		getRerankingConfig,
		updateRerankingConfig,
		getRAGConfig,
		updateRAGConfig
	} from '$lib/apis/retrieval';

	import { reindexKnowledgeFiles } from '$lib/apis/knowledge';
	import { deleteAllFiles } from '$lib/apis/files';

	import ResetUploadDirConfirmDialog from '$lib/components/common/ConfirmDialog.svelte';
	import ResetVectorDBConfirmDialog from '$lib/components/common/ConfirmDialog.svelte';
	import ReindexKnowledgeFilesConfirmDialog from '$lib/components/common/ConfirmDialog.svelte';
	import SensitiveInput from '$lib/components/common/SensitiveInput.svelte';
	import Tooltip from '$lib/components/common/Tooltip.svelte';
	import Switch from '$lib/components/common/Switch.svelte';
	import Textarea from '$lib/components/common/Textarea.svelte';
	import Spinner from '$lib/components/common/Spinner.svelte';
	import ChevronDown from '$lib/components/icons/ChevronDown.svelte';

	import { ragConfigCache, retrievalEmbeddingCache } from '$lib/stores';

	const i18n: Writable<any> = getContext('i18n');

	// 折叠状态
	let expandedSections = {
		general: true,
		embedding: false,
		retrieval: false,
		files: false,
		integration: false,
		danger: false
	};

	let updateEmbeddingModelLoading = false;
	let updateRerankingModelLoading = false;

	let showResetConfirm = false;
	let showResetUploadDirConfirm = false;
	let showReindexConfirm = false;

	let RAG_EMBEDDING_ENGINE = '';
	let RAG_EMBEDDING_MODEL = '';
	let RAG_EMBEDDING_BATCH_SIZE = 1;
	let ENABLE_ASYNC_EMBEDDING = true;

	let rerankingModel = '';

	let OpenAIUrl = '';
	let OpenAIKey = '';

	let AzureOpenAIUrl = '';
	let AzureOpenAIKey = '';
	let AzureOpenAIVersion = '';

	let OllamaUrl = '';
	let OllamaKey = '';

	let querySettings = {
		template: '',
		r: 0.0,
		k: 4,
		k_reranker: 4,
		hybrid: false
	};

	let RAGConfig: any = null;

	const embeddingModelUpdateHandler = async () => {
		if (RAG_EMBEDDING_ENGINE === '' && RAG_EMBEDDING_MODEL.split('/').length - 1 > 1) {
			toast.error(
				$i18n.t(
					'Model filesystem path detected. Model shortname is required for update, cannot continue.'
				)
			);
			return;
		}
		if (RAG_EMBEDDING_ENGINE === 'ollama' && RAG_EMBEDDING_MODEL === '') {
			toast.error(
				$i18n.t(
					'Model filesystem path detected. Model shortname is required for update, cannot continue.'
				)
			);
			return;
		}

		if (RAG_EMBEDDING_ENGINE === 'openai' && RAG_EMBEDDING_MODEL === '') {
			toast.error(
				$i18n.t(
					'Model filesystem path detected. Model shortname is required for update, cannot continue.'
				)
			);
			return;
		}

		if (
			RAG_EMBEDDING_ENGINE === 'azure_openai' &&
			(AzureOpenAIKey === '' || AzureOpenAIUrl === '' || AzureOpenAIVersion === '')
		) {
			toast.error($i18n.t('OpenAI URL/Key required.'));
			return;
		}

		console.debug('Update embedding model attempt:', {
			RAG_EMBEDDING_ENGINE,
			RAG_EMBEDDING_MODEL,
			RAG_EMBEDDING_BATCH_SIZE,
			ENABLE_ASYNC_EMBEDDING
		});

		updateEmbeddingModelLoading = true;
		const res = await updateEmbeddingConfig(localStorage.token, {
			RAG_EMBEDDING_ENGINE: RAG_EMBEDDING_ENGINE,
			RAG_EMBEDDING_MODEL: RAG_EMBEDDING_MODEL,
			RAG_EMBEDDING_BATCH_SIZE: RAG_EMBEDDING_BATCH_SIZE,
			ENABLE_ASYNC_EMBEDDING: ENABLE_ASYNC_EMBEDDING,
			ollama_config: {
				key: OllamaKey,
				url: OllamaUrl
			},
			openai_config: {
				key: OpenAIKey,
				url: OpenAIUrl
			},
			azure_openai_config: {
				key: AzureOpenAIKey,
				url: AzureOpenAIUrl,
				version: AzureOpenAIVersion
			}
		} as any).catch(async (error) => {
			toast.error(`${error}`);
			await setEmbeddingConfig();
			return null;
		});
		updateEmbeddingModelLoading = false;

		if (res) {
			console.debug('embeddingModelUpdateHandler:', res);
			// Clear cache after update
			retrievalEmbeddingCache.set(null);
		}
	};

	const submitHandler = async () => {
		if (
			RAGConfig.CONTENT_EXTRACTION_ENGINE === 'external' &&
			RAGConfig.EXTERNAL_DOCUMENT_LOADER_URL === ''
		) {
			toast.error($i18n.t('External Document Loader URL required.'));
			return;
		}
		if (RAGConfig.CONTENT_EXTRACTION_ENGINE === 'tika' && RAGConfig.TIKA_SERVER_URL === '') {
			toast.error($i18n.t('Tika Server URL required.'));
			return;
		}
		if (RAGConfig.CONTENT_EXTRACTION_ENGINE === 'docling' && RAGConfig.DOCLING_SERVER_URL === '') {
			toast.error($i18n.t('Docling Server URL required.'));
			return;
		}
		if (
			RAGConfig.CONTENT_EXTRACTION_ENGINE === 'datalab_marker' &&
			RAGConfig.DATALAB_MARKER_ADDITIONAL_CONFIG &&
			RAGConfig.DATALAB_MARKER_ADDITIONAL_CONFIG.trim() !== ''
		) {
			try {
				JSON.parse(RAGConfig.DATALAB_MARKER_ADDITIONAL_CONFIG);
			} catch (e) {
				toast.error($i18n.t('Invalid JSON format in Additional Config'));
				return;
			}
		}

		if (
			RAGConfig.CONTENT_EXTRACTION_ENGINE === 'document_intelligence' &&
			RAGConfig.DOCUMENT_INTELLIGENCE_ENDPOINT === ''
		) {
			toast.error($i18n.t('Document Intelligence endpoint required.'));
			return;
		}
		if (
			RAGConfig.CONTENT_EXTRACTION_ENGINE === 'mistral_ocr' &&
			RAGConfig.MISTRAL_OCR_API_KEY === ''
		) {
			toast.error($i18n.t('Mistral OCR API Key required.'));
			return;
		}

		if (
			RAGConfig.CONTENT_EXTRACTION_ENGINE === 'mineru' &&
			RAGConfig.MINERU_API_MODE === 'cloud' &&
			RAGConfig.MINERU_API_KEY === ''
		) {
			toast.error($i18n.t('MinerU API Key required for Cloud API mode.'));
			return;
		}

		if (!RAGConfig.BYPASS_EMBEDDING_AND_RETRIEVAL) {
			await embeddingModelUpdateHandler();
		}

		if (RAGConfig.DOCLING_PARAMS) {
			try {
				JSON.parse(RAGConfig.DOCLING_PARAMS);
			} catch (e) {
				toast.error(
					$i18n.t('Invalid JSON format in {{NAME}}', {
						NAME: $i18n.t('Docling Parameters')
					})
				);
				return;
			}
		}
		if (RAGConfig.MINERU_PARAMS) {
			try {
				JSON.parse(RAGConfig.MINERU_PARAMS);
			} catch (e) {
				toast.error($i18n.t('Invalid JSON format in MinerU Parameters'));
				return;
			}
		}

		const res = await updateRAGConfig(localStorage.token, {
			...RAGConfig,
			ALLOWED_FILE_EXTENSIONS: RAGConfig.ALLOWED_FILE_EXTENSIONS.split(',')
				.map((ext) => ext.trim())
				.filter((ext) => ext !== ''),
			DOCLING_PARAMS:
				typeof RAGConfig.DOCLING_PARAMS === 'string' && RAGConfig.DOCLING_PARAMS.trim() !== ''
					? JSON.parse(RAGConfig.DOCLING_PARAMS)
					: {},
			MINERU_PARAMS:
				typeof RAGConfig.MINERU_PARAMS === 'string' && RAGConfig.MINERU_PARAMS.trim() !== ''
					? JSON.parse(RAGConfig.MINERU_PARAMS)
					: {}
		});
		// Clear cache after save to ensure fresh data on next visit
		ragConfigCache.set(null);
		dispatch('save');
	};

	const setEmbeddingConfig = async () => {
		// Use cached embedding config if available
		let embeddingConfig;
		if ($retrievalEmbeddingCache) {
			embeddingConfig = $retrievalEmbeddingCache;
		} else {
			embeddingConfig = await getEmbeddingConfig(localStorage.token);
			retrievalEmbeddingCache.set(embeddingConfig);
		}

		if (embeddingConfig) {
			RAG_EMBEDDING_ENGINE = embeddingConfig.RAG_EMBEDDING_ENGINE;
			RAG_EMBEDDING_MODEL = embeddingConfig.RAG_EMBEDDING_MODEL;
			RAG_EMBEDDING_BATCH_SIZE = embeddingConfig.RAG_EMBEDDING_BATCH_SIZE ?? 1;
			ENABLE_ASYNC_EMBEDDING = embeddingConfig.ENABLE_ASYNC_EMBEDDING ?? true;

			OpenAIKey = embeddingConfig.openai_config.key;
			OpenAIUrl = embeddingConfig.openai_config.url;

			OllamaKey = embeddingConfig.ollama_config.key;
			OllamaUrl = embeddingConfig.ollama_config.url;

			AzureOpenAIKey = embeddingConfig.azure_openai_config.key;
			AzureOpenAIUrl = embeddingConfig.azure_openai_config.url;
			AzureOpenAIVersion = embeddingConfig.azure_openai_config.version;
		}
	};
	onMount(async () => {
		try {
			await setEmbeddingConfig();

			// Use cached RAG config if available
			let config;
			if ($ragConfigCache) {
				config = JSON.parse(JSON.stringify($ragConfigCache)); // Deep clone
			} else {
				config = await getRAGConfig(localStorage.token);
				ragConfigCache.set(JSON.parse(JSON.stringify(config)));
			}

			// Ensure ALLOWED_FILE_EXTENSIONS is an array before joining
			const extensions = Array.isArray(config?.ALLOWED_FILE_EXTENSIONS)
				? config.ALLOWED_FILE_EXTENSIONS
				: [];
			config.ALLOWED_FILE_EXTENSIONS = extensions.join(', ');

			config.DOCLING_PARAMS =
				typeof config.DOCLING_PARAMS === 'object'
					? JSON.stringify(config.DOCLING_PARAMS ?? {}, null, 2)
					: config.DOCLING_PARAMS;

			config.MINERU_PARAMS =
				typeof config.MINERU_PARAMS === 'object'
					? JSON.stringify(config.MINERU_PARAMS ?? {}, null, 2)
					: config.MINERU_PARAMS;

			RAGConfig = config;
		} catch (error) {
			console.error('Failed to load RAG config:', error);
			// Clear cache and show error
			ragConfigCache.set(null);
			toast.error($i18n.t('Failed to load document settings'));
			// Set empty config to prevent infinite loading
			RAGConfig = {};
		}
	});
</script>

<ResetUploadDirConfirmDialog
	bind:show={showResetUploadDirConfirm}
	on:confirm={async () => {
		const res = await deleteAllFiles(localStorage.token).catch((error) => {
			toast.error(`${error}`);
			return null;
		});

		if (res) {
			toast.success($i18n.t('Success'));
		}
	}}
/>

<ResetVectorDBConfirmDialog
	bind:show={showResetConfirm}
	on:confirm={() => {
		const res = resetVectorDB(localStorage.token).catch((error) => {
			toast.error(`${error}`);
			return null;
		});

		if (res) {
			toast.success($i18n.t('Success'));
		}
	}}
/>

<ReindexKnowledgeFilesConfirmDialog
	bind:show={showReindexConfirm}
	on:confirm={async () => {
		const res = await reindexKnowledgeFiles(localStorage.token).catch((error) => {
			toast.error(`${error}`);
			return null;
		});

		if (res) {
			toast.success($i18n.t('Success'));
		}
	}}
/>

<form
	class="flex flex-col h-full justify-between space-y-3 text-sm"
	on:submit|preventDefault={() => {
		submitHandler();
	}}
>
	{#if RAGConfig}
		<div class="space-y-3 overflow-y-scroll scrollbar-hidden h-full pr-2">
			<!-- 通用设置 General -->
			<div class="max-w-5xl mx-auto rounded-xl border border-gray-200 dark:border-gray-800">
				<button
					type="button"
					class="w-full flex items-center justify-between px-5 py-4 text-left"
					on:click={() => (expandedSections.general = !expandedSections.general)}
				>
					<div class="flex items-center gap-3">
						<svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="1.5" stroke="currentColor" class="size-5 text-gray-500">
							<path stroke-linecap="round" stroke-linejoin="round" d="M9.594 3.94c.09-.542.56-.94 1.11-.94h2.593c.55 0 1.02.398 1.11.94l.213 1.281c.063.374.313.686.645.87.074.04.147.083.22.127.325.196.72.257 1.075.124l1.217-.456a1.125 1.125 0 0 1 1.37.49l1.296 2.247a1.125 1.125 0 0 1-.26 1.431l-1.003.827c-.293.241-.438.613-.43.992a7.723 7.723 0 0 1 0 .255c-.008.378.137.75.43.991l1.004.827c.424.35.534.955.26 1.43l-1.298 2.247a1.125 1.125 0 0 1-1.369.491l-1.217-.456c-.355-.133-.75-.072-1.076.124a6.47 6.47 0 0 1-.22.128c-.331.183-.581.495-.644.869l-.213 1.281c-.09.543-.56.94-1.11.94h-2.594c-.55 0-1.019-.398-1.11-.94l-.213-1.281c-.062-.374-.312-.686-.644-.87a6.52 6.52 0 0 1-.22-.127c-.325-.196-.72-.257-1.076-.124l-1.217.456a1.125 1.125 0 0 1-1.369-.49l-1.297-2.247a1.125 1.125 0 0 1 .26-1.431l1.004-.827c.292-.24.437-.613.43-.991a6.932 6.932 0 0 1 0-.255c.007-.38-.138-.751-.43-.992l-1.004-.827a1.125 1.125 0 0 1-.26-1.43l1.297-2.247a1.125 1.125 0 0 1 1.37-.491l1.216.456c.356.133.751.072 1.076-.124.072-.044.146-.086.22-.128.332-.183.582-.495.644-.869l.214-1.28Z" />
							<path stroke-linecap="round" stroke-linejoin="round" d="M15 12a3 3 0 1 1-6 0 3 3 0 0 1 6 0Z" />
						</svg>
						<span class="text-base font-medium text-gray-900 dark:text-gray-100">{$i18n.t('General')}</span>
					</div>
					<div class="transform transition-transform duration-200 {expandedSections.general ? 'rotate-180' : ''}">
						<ChevronDown className="size-5 text-gray-400" />
					</div>
				</button>

				{#if expandedSections.general}
					<div transition:slide={{ duration: 200, easing: quintOut }} class="px-5 pb-5 space-y-4">
						<!-- 内容提取引擎 -->
						<div class="bg-white dark:bg-gray-900 rounded-lg p-4 border border-gray-100 dark:border-gray-800">
							<div class="flex items-center justify-between mb-3">
								<div class="text-sm font-medium">{$i18n.t('Content Extraction Engine')}</div>
								<select
									class="dark:bg-gray-850 w-fit pr-8 rounded-lg px-2 py-1.5 text-sm bg-gray-50 outline-none border border-gray-200 dark:border-gray-700 text-right cursor-pointer"
									bind:value={RAGConfig.CONTENT_EXTRACTION_ENGINE}
								>
									<option value="">{$i18n.t('Default')}</option>
									<option value="external">{$i18n.t('External')}</option>
									<option value="tika">{$i18n.t('Tika')}</option>
									<option value="docling">{$i18n.t('Docling')}</option>
									<option value="datalab_marker">{$i18n.t('Datalab Marker API')}</option>
									<option value="document_intelligence">{$i18n.t('Document Intelligence')}</option>
									<option value="mistral_ocr">{$i18n.t('Mistral OCR')}</option>
									<option value="mineru">{$i18n.t('MinerU')}</option>
								</select>
							</div>

							{#if RAGConfig.CONTENT_EXTRACTION_ENGINE === ''}
								<div class="flex w-full mt-1">
									<div class="flex-1 flex justify-between">
										<div class=" self-center text-xs font-medium">
											{$i18n.t('PDF Extract Images (OCR)')}
										</div>
										<div class="flex items-center relative">
											<Switch bind:state={RAGConfig.PDF_EXTRACT_IMAGES} />
										</div>
									</div>
								</div>
							{:else if RAGConfig.CONTENT_EXTRACTION_ENGINE === 'datalab_marker'}
								<div class="my-0.5 flex gap-2 pr-2">
									<Tooltip
										content={$i18n.t(
											'API Base URL for Datalab Marker service. Defaults to: https://www.datalab.to/api/v1/marker'
										)}
										placement="top-start"
										className="w-full"
									>
										<input
											class="flex-1 w-full rounded-lg py-2 px-4 text-sm bg-white dark:bg-gray-900 dark:text-gray-300 border border-gray-100 dark:border-gray-800 outline-none focus:border-gray-300 dark:focus:border-gray-700 transition"
											placeholder={$i18n.t('Enter Datalab Marker API Base URL')}
											bind:value={RAGConfig.DATALAB_MARKER_API_BASE_URL}
										/>
									</Tooltip>
								</div>
								<div class="my-0.5 flex gap-2 pr-2">
									<SensitiveInput
										placeholder={$i18n.t('Enter Datalab Marker API Key')}
										required={false}
										bind:value={RAGConfig.DATALAB_MARKER_API_KEY}
									/>
								</div>

								<div class="flex flex-col gap-2 mt-2">
									<div class=" flex flex-col w-full justify-between">
										<div class=" mb-1 text-xs font-medium">
											{$i18n.t('Additional Config')}
										</div>
										<div class="flex w-full items-center relative">
											<Tooltip
												content={$i18n.t(
													'Additional configuration options for marker. This should be a JSON string with key-value pairs. For example, \'{"key": "value"}\'. Supported keys include: disable_links, keep_pageheader_in_output, keep_pagefooter_in_output, filter_blank_pages, drop_repeated_text, layout_coverage_threshold, merge_threshold, height_tolerance, gap_threshold, image_threshold, min_line_length, level_count, default_level'
												)}
												placement="top-start"
												className="w-full"
											>
												<Textarea
													bind:value={RAGConfig.DATALAB_MARKER_ADDITIONAL_CONFIG}
													placeholder={$i18n.t('Enter JSON config (e.g., {"disable_links": true})')}
												/>
											</Tooltip>
										</div>
									</div>
								</div>

								<div class="flex justify-between w-full mt-2">
									<div class="self-center text-xs font-medium">
										<Tooltip
											content={$i18n.t(
												'Significantly improves accuracy by using an LLM to enhance tables, forms, inline math, and layout detection. Will increase latency. Defaults to False.'
											)}
											placement="top-start"
										>
											{$i18n.t('Use LLM')}
										</Tooltip>
									</div>
									<div class="flex items-center">
										<Switch bind:state={RAGConfig.DATALAB_MARKER_USE_LLM} />
									</div>
								</div>
								<div class="flex justify-between w-full mt-2">
									<div class="self-center text-xs font-medium">
										<Tooltip
											content={$i18n.t(
												'Skip the cache and re-run the inference. Defaults to False.'
											)}
											placement="top-start"
										>
											{$i18n.t('Skip Cache')}
										</Tooltip>
									</div>
									<div class="flex items-center">
										<Switch bind:state={RAGConfig.DATALAB_MARKER_SKIP_CACHE} />
									</div>
								</div>
								<div class="flex justify-between w-full mt-2">
									<div class="self-center text-xs font-medium">
										<Tooltip
											content={$i18n.t(
												'Force OCR on all pages of the PDF. This can lead to worse results if you have good text in your PDFs. Defaults to False.'
											)}
											placement="top-start"
										>
											{$i18n.t('Force OCR')}
										</Tooltip>
									</div>
									<div class="flex items-center">
										<Switch bind:state={RAGConfig.DATALAB_MARKER_FORCE_OCR} />
									</div>
								</div>
								<div class="flex justify-between w-full mt-2">
									<div class="self-center text-xs font-medium">
										<Tooltip
											content={$i18n.t(
												'Whether to paginate the output. Each page will be separated by a horizontal rule and page number. Defaults to False.'
											)}
											placement="top-start"
										>
											{$i18n.t('Paginate')}
										</Tooltip>
									</div>
									<div class="flex items-center">
										<Switch bind:state={RAGConfig.DATALAB_MARKER_PAGINATE} />
									</div>
								</div>
								<div class="flex justify-between w-full mt-2">
									<div class="self-center text-xs font-medium">
										<Tooltip
											content={$i18n.t(
												'Strip existing OCR text from the PDF and re-run OCR. Ignored if Force OCR is enabled. Defaults to False.'
											)}
											placement="top-start"
										>
											{$i18n.t('Strip Existing OCR')}
										</Tooltip>
									</div>
									<div class="flex items-center">
										<Switch bind:state={RAGConfig.DATALAB_MARKER_STRIP_EXISTING_OCR} />
									</div>
								</div>
								<div class="flex justify-between w-full mt-2">
									<div class="self-center text-xs font-medium">
										<Tooltip
											content={$i18n.t(
												'Disable image extraction from the PDF. If Use LLM is enabled, images will be automatically captioned. Defaults to False.'
											)}
											placement="top-start"
										>
											{$i18n.t('Disable Image Extraction')}
										</Tooltip>
									</div>
									<div class="flex items-center">
										<Switch bind:state={RAGConfig.DATALAB_MARKER_DISABLE_IMAGE_EXTRACTION} />
									</div>
								</div>
								<div class="flex justify-between w-full mt-2">
									<div class="self-center text-xs font-medium">
										<Tooltip
											content={$i18n.t(
												'Format the lines in the output. Defaults to False. If set to True, the lines will be formatted to detect inline math and styles.'
											)}
											placement="top-start"
										>
											{$i18n.t('Format Lines')}
										</Tooltip>
									</div>
									<div class="flex items-center">
										<Switch bind:state={RAGConfig.DATALAB_MARKER_FORMAT_LINES} />
									</div>
								</div>
								<div class="flex justify-between w-full mt-2">
									<div class="self-center text-xs font-medium">
										<Tooltip
											content={$i18n.t(
												"The output format for the text. Can be 'json', 'markdown', or 'html'. Defaults to 'markdown'."
											)}
											placement="top-start"
										>
											{$i18n.t('Output Format')}
										</Tooltip>
									</div>
									<div class="">
										<select
											class="dark:bg-gray-900 w-fit pr-8 rounded-lg px-2 py-1 text-sm bg-transparent outline-none focus:ring-0 focus:border-gray-300 border-none text-right cursor-pointer"
											bind:value={RAGConfig.DATALAB_MARKER_OUTPUT_FORMAT}
										>
											<option value="markdown">{$i18n.t('Markdown')}</option>
											<option value="json">{$i18n.t('JSON')}</option>
											<option value="html">{$i18n.t('HTML')}</option>
										</select>
									</div>
								</div>
							{:else if RAGConfig.CONTENT_EXTRACTION_ENGINE === 'external'}
								<hr class="border-gray-100 dark:border-gray-800 my-3" />

								<div class="flex flex-col gap-3">
									<div class="w-full">
										<div class="text-sm font-medium text-gray-700 dark:text-gray-300 mb-2">
											{$i18n.t('API Base URL')}
										</div>
										<input
											class="w-full rounded-lg py-2 px-4 text-sm bg-white dark:bg-gray-900 dark:text-gray-300 border border-gray-100 dark:border-gray-800 outline-none focus:border-gray-300 dark:focus:border-gray-700 transition"
											placeholder={$i18n.t('Enter External Document Loader URL')}
											bind:value={RAGConfig.EXTERNAL_DOCUMENT_LOADER_URL}
										/>
									</div>
									<div class="w-full">
										<div class="text-sm font-medium text-gray-700 dark:text-gray-300 mb-2">
											{$i18n.t('API Key')}
										</div>
										<div class="w-full rounded-lg py-2 px-4 text-sm bg-white dark:bg-gray-900 dark:text-gray-300 border border-gray-100 dark:border-gray-800 transition flex items-center">
											<SensitiveInput
												placeholder={$i18n.t('Enter External Document Loader API Key')}
												required={false}
												bind:value={RAGConfig.EXTERNAL_DOCUMENT_LOADER_API_KEY}
											/>
										</div>
									</div>
								</div>
							{:else if RAGConfig.CONTENT_EXTRACTION_ENGINE === 'tika'}
								<div class="flex w-full mt-1">
									<div class="flex-1 mr-2">
										<input
											class="flex-1 w-full rounded-lg py-2 px-4 text-sm bg-white dark:bg-gray-900 dark:text-gray-300 border border-gray-100 dark:border-gray-800 outline-none focus:border-gray-300 dark:focus:border-gray-700 transition"
											placeholder={$i18n.t('Enter Tika Server URL')}
											bind:value={RAGConfig.TIKA_SERVER_URL}
										/>
									</div>
								</div>
							{:else if RAGConfig.CONTENT_EXTRACTION_ENGINE === 'docling'}
								<hr class="border-gray-100 dark:border-gray-800 my-3" />

								<div class="flex flex-col gap-3">
									<div class="w-full">
										<div class="text-sm font-medium text-gray-700 dark:text-gray-300 mb-2">
											{$i18n.t('API Base URL')}
										</div>
										<input
											class="w-full rounded-lg py-2 px-4 text-sm bg-white dark:bg-gray-900 dark:text-gray-300 border border-gray-100 dark:border-gray-800 outline-none focus:border-gray-300 dark:focus:border-gray-700 transition"
											placeholder={$i18n.t('Enter Docling Server URL')}
											bind:value={RAGConfig.DOCLING_SERVER_URL}
										/>
									</div>
									<div class="w-full">
										<div class="text-sm font-medium text-gray-700 dark:text-gray-300 mb-2">
											{$i18n.t('API Key')}
										</div>
										<div class="w-full rounded-lg py-2 px-4 text-sm bg-white dark:bg-gray-900 dark:text-gray-300 border border-gray-100 dark:border-gray-800 transition flex items-center">
											<SensitiveInput
												placeholder={$i18n.t('Enter Docling API Key')}
												bind:value={RAGConfig.DOCLING_API_KEY}
												required={false}
											/>
										</div>
									</div>
								</div>

								<div class="flex flex-col gap-2 mt-2">
									<div class=" flex flex-col w-full justify-between">
										<div class=" mb-1 text-xs font-medium">
											{$i18n.t('Parameters')}
										</div>
										<div class="flex w-full items-center relative">
											<Textarea
												bind:value={RAGConfig.DOCLING_PARAMS}
												placeholder={$i18n.t('Enter additional parameters in JSON format')}
												minSize={100}
											/>
										</div>
									</div>
								</div>
							{:else if RAGConfig.CONTENT_EXTRACTION_ENGINE === 'document_intelligence'}
								<hr class="border-gray-100 dark:border-gray-800 my-3" />

								<div class="flex flex-col gap-3">
									<div class="w-full">
										<div class="text-sm font-medium text-gray-700 dark:text-gray-300 mb-2">
											{$i18n.t('Document Intelligence Endpoint')}
										</div>
										<input
											class="w-full rounded-lg py-2 px-4 text-sm bg-white dark:bg-gray-900 dark:text-gray-300 border border-gray-100 dark:border-gray-800 outline-none focus:border-gray-300 dark:focus:border-gray-700 transition"
											placeholder={$i18n.t('Enter Document Intelligence Endpoint')}
											bind:value={RAGConfig.DOCUMENT_INTELLIGENCE_ENDPOINT}
										/>
									</div>
									<div class="w-full">
										<div class="text-sm font-medium text-gray-700 dark:text-gray-300 mb-2">
											{$i18n.t('API Key')}
										</div>
										<div class="w-full rounded-lg py-2 px-4 text-sm bg-white dark:bg-gray-900 dark:text-gray-300 border border-gray-100 dark:border-gray-800 transition flex items-center">
											<SensitiveInput
												placeholder={$i18n.t('Enter Document Intelligence Key')}
												bind:value={RAGConfig.DOCUMENT_INTELLIGENCE_KEY}
												required={false}
											/>
										</div>
									</div>
								</div>
								<div class="my-0.5 flex flex-col w-full">
									<div class=" mb-1 text-xs font-medium">
										{$i18n.t('Document Intelligence Model')}
									</div>
									<div class="flex w-full">
										<div class="flex-1 mr-2">
											<input
												class="flex-1 w-full rounded-lg py-2 px-4 text-sm bg-white dark:bg-gray-900 dark:text-gray-300 border border-gray-100 dark:border-gray-800 outline-none focus:border-gray-300 dark:focus:border-gray-700 transition"
												placeholder={$i18n.t('Enter Document Intelligence Model')}
												bind:value={RAGConfig.DOCUMENT_INTELLIGENCE_MODEL}
											/>
										</div>
									</div>
								</div>
							{:else if RAGConfig.CONTENT_EXTRACTION_ENGINE === 'mistral_ocr'}
								<hr class="border-gray-100 dark:border-gray-800 my-3" />

								<div class="flex flex-col gap-3">
									<div class="w-full">
										<div class="text-sm font-medium text-gray-700 dark:text-gray-300 mb-2">
											{$i18n.t('API Base URL')}
										</div>
										<input
											class="w-full rounded-lg py-2 px-4 text-sm bg-white dark:bg-gray-900 dark:text-gray-300 border border-gray-100 dark:border-gray-800 outline-none focus:border-gray-300 dark:focus:border-gray-700 transition"
											placeholder={$i18n.t('Enter Mistral API Base URL')}
											bind:value={RAGConfig.MISTRAL_OCR_API_BASE_URL}
										/>
									</div>
									<div class="w-full">
										<div class="text-sm font-medium text-gray-700 dark:text-gray-300 mb-2">
											{$i18n.t('API Key')}
										</div>
										<div class="w-full rounded-lg py-2 px-4 text-sm bg-white dark:bg-gray-900 dark:text-gray-300 border border-gray-100 dark:border-gray-800 transition flex items-center">
											<SensitiveInput
												placeholder={$i18n.t('Enter Mistral API Key')}
												bind:value={RAGConfig.MISTRAL_OCR_API_KEY}
											/>
										</div>
									</div>
								</div>
							{:else if RAGConfig.CONTENT_EXTRACTION_ENGINE === 'mineru'}
								<!-- API Mode Selection -->
								<div class="flex w-full mt-2">
									<div class="flex-1 flex justify-between">
										<div class="self-center text-xs font-medium">
											{$i18n.t('API Mode')}
										</div>
										<select
											class="dark:bg-gray-900 w-fit pr-8 rounded-lg px-2 py-1 text-sm bg-transparent outline-none focus:ring-0 focus:border-gray-300 border-none text-right cursor-pointer"
											bind:value={RAGConfig.MINERU_API_MODE}
											on:change={() => {
												// Auto-update URL when switching modes if it's empty or matches the opposite mode's default
												const cloudUrl = 'https://mineru.net/api/v4';
												const localUrl = 'http://localhost:8000';

												if (RAGConfig.MINERU_API_MODE === 'cloud') {
													if (!RAGConfig.MINERU_API_URL || RAGConfig.MINERU_API_URL === localUrl) {
														RAGConfig.MINERU_API_URL = cloudUrl;
													}
												} else {
													if (!RAGConfig.MINERU_API_URL || RAGConfig.MINERU_API_URL === cloudUrl) {
														RAGConfig.MINERU_API_URL = localUrl;
													}
												}
											}}
										>
											<option value="local">{$i18n.t('local')}</option>
											<option value="cloud">{$i18n.t('cloud')}</option>
										</select>
									</div>
								</div>

								<!-- API URL -->
								<div class="flex w-full mt-2">
									<input
										class="flex-1 w-full rounded-lg py-2 px-4 text-sm bg-white dark:bg-gray-900 dark:text-gray-300 border border-gray-100 dark:border-gray-800 outline-none focus:border-gray-300 dark:focus:border-gray-700 transition"
										placeholder={RAGConfig.MINERU_API_MODE === 'cloud'
											? $i18n.t('https://mineru.net/api/v4')
											: $i18n.t('http://localhost:8000')}
										bind:value={RAGConfig.MINERU_API_URL}
									/>
								</div>

								<div class="flex w-full mt-2">
									<div class="text-xs font-medium text-gray-500 mb-1.5">
										{$i18n.t('API Key')}
									</div>
								</div>
								<div class="w-full rounded-lg py-2 px-4 text-sm bg-white dark:bg-gray-900 dark:text-gray-300 border border-gray-100 dark:border-gray-800 transition flex items-center">
									<SensitiveInput
										placeholder={$i18n.t('Enter MinerU API Key')}
										bind:value={RAGConfig.MINERU_API_KEY}
									/>
								</div>

								<div class="flex w-full mt-2">
									<div class="flex-1 flex justify-between">
										<div class="self-center text-xs font-medium">
											{$i18n.t('API Timeout')}
										</div>
										<input
											class="w-16 text-sm bg-transparent outline-hidden text-right"
											type="number"
											min="1"
											bind:value={RAGConfig.MINERU_API_TIMEOUT}
											placeholder="60"
										/>
									</div>
								</div>

								<!-- Parameters -->
								<div class="flex flex-col justify-between w-full mt-2">
									<div class="text-xs font-medium">
										<Tooltip
											content={$i18n.t(
												'Advanced parameters for MinerU parsing (enable_ocr, enable_formula, enable_table, language, model_version, page_ranges)'
											)}
											placement="top-start"
										>
											{$i18n.t('Parameters')}
										</Tooltip>
									</div>
									<div class="mt-1.5">
										<Textarea
											bind:value={RAGConfig.MINERU_PARAMS}
											placeholder={`{\n  "enable_ocr": false,\n  "enable_formula": true,\n  "enable_table": true,\n  "language": "en",\n  "model_version": "pipeline",\n  "page_ranges": ""\n}`}
											minSize={100}
										/>
									</div>
								</div>
							{/if}
						</div>

						<hr class="border-gray-100 dark:border-gray-800 my-2.5" />

						<div class="  mb-2.5 flex w-full justify-between">
							<div class=" self-center text-xs font-medium">
								<Tooltip content={$i18n.t('Full Context Mode')} placement="top-start">
									{$i18n.t('Bypass Embedding and Retrieval')}
								</Tooltip>
							</div>
							<div class="flex items-center relative">
								<Tooltip
									content={RAGConfig.BYPASS_EMBEDDING_AND_RETRIEVAL
										? $i18n.t(
												'Inject the entire content as context for comprehensive processing, this is recommended for complex queries.'
											)
										: $i18n.t(
												'Default to segmented retrieval for focused and relevant content extraction, this is recommended for most cases.'
											)}
								>
									<Switch bind:state={RAGConfig.BYPASS_EMBEDDING_AND_RETRIEVAL} />
								</Tooltip>
							</div>
						</div>

						{#if !RAGConfig.BYPASS_EMBEDDING_AND_RETRIEVAL}
							<hr class="border-gray-100 dark:border-gray-800 my-2.5" />

							<div class="  mb-2.5 flex w-full justify-between">
								<div class=" self-center text-xs font-medium">{$i18n.t('Text Splitter')}</div>
								<div class="flex items-center relative">
									<select
										class="dark:bg-gray-900 w-fit pr-8 rounded-lg px-2 py-1 text-sm bg-transparent outline-none focus:ring-0 focus:border-gray-300 border-none text-right cursor-pointer"
										bind:value={RAGConfig.TEXT_SPLITTER}
									>
										<option value="">{$i18n.t('Default')} ({$i18n.t('Character')})</option>
										<option value="token">{$i18n.t('Token')} ({$i18n.t('Tiktoken')})</option>
									</select>
								</div>
							</div>

							<div class="  mb-2.5 flex w-full justify-between">
								<div class=" self-center text-xs font-medium">
									<Tooltip
										placement="top-start"
										content={$i18n.t(
											'Split documents by markdown headers before applying character/token splitting.'
										)}
									>
										{$i18n.t('Markdown Header Text Splitter')}
									</Tooltip>
								</div>
								<div class="flex items-center relative">
									<Switch bind:state={RAGConfig.ENABLE_MARKDOWN_HEADER_TEXT_SPLITTER} />
								</div>
							</div>

							<div class="  mb-2.5 flex w-full justify-between">
								<div class=" flex gap-1.5 w-full">
									<div class="  w-full justify-between">
										<div class="self-center text-xs font-medium min-w-fit mb-1">
											{$i18n.t('Chunk Size')}
										</div>
										<div class="self-center">
											<input
												class=" w-full rounded-lg py-1.5 px-4 text-sm bg-gray-50 dark:text-gray-300 dark:bg-gray-850 outline-hidden"
												type="number"
												placeholder={$i18n.t('Enter Chunk Size')}
												bind:value={RAGConfig.CHUNK_SIZE}
												autocomplete="off"
												min="0"
											/>
										</div>
									</div>

									<div class="w-full">
										<div class=" self-center text-xs font-medium min-w-fit mb-1">
											{$i18n.t('Chunk Overlap')}
										</div>

										<div class="self-center">
											<input
												class="w-full rounded-lg py-2 px-4 text-sm bg-white dark:bg-gray-900 dark:text-gray-300 border border-gray-100 dark:border-gray-800 outline-none focus:border-gray-300 dark:focus:border-gray-700 transition"
												type="number"
												placeholder={$i18n.t('Enter Chunk Overlap')}
												bind:value={RAGConfig.CHUNK_OVERLAP}
												autocomplete="off"
												min="0"
											/>
										</div>
									</div>
								</div>
							</div>

							{#if RAGConfig.ENABLE_MARKDOWN_HEADER_TEXT_SPLITTER}
								<div class="  mb-2.5 flex w-full justify-between">
									<div class=" flex gap-1.5 w-full">
										<div class="w-full">
											<div class="self-center text-xs font-medium min-w-fit mb-1">
												<Tooltip
													placement="top-start"
													content={$i18n.t(
														'Chunks smaller than this threshold will be merged with neighboring chunks when possible. Set to 0 to disable merging.'
													)}
												>
													{$i18n.t('Chunk Min Size Target')}
												</Tooltip>
											</div>
											<div class="self-center">
												<input
													class="w-full rounded-lg py-2 px-4 text-sm bg-white dark:bg-gray-900 dark:text-gray-300 border border-gray-100 dark:border-gray-800 outline-none focus:border-gray-300 dark:focus:border-gray-700 transition"
													type="number"
													placeholder={$i18n.t('Enter Chunk Min Size Target')}
													bind:value={RAGConfig.CHUNK_MIN_SIZE_TARGET}
													autocomplete="off"
													min="0"
												/>
											</div>
										</div>
									</div>
								</div>
							{/if}
						{/if}
					</div>
				{/if}
			</div>

			{#if !RAGConfig.BYPASS_EMBEDDING_AND_RETRIEVAL}
				<!-- 嵌入模型 Embedding -->
				<div class="max-w-5xl mx-auto rounded-xl border border-gray-200 dark:border-gray-800">
					<button
						type="button"
						class="w-full flex items-center justify-between px-5 py-4 text-left"
						on:click={() => (expandedSections.embedding = !expandedSections.embedding)}
					>
						<div class="flex items-center gap-3">
							<svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="1.5" stroke="currentColor" class="size-5 text-gray-500">
								<path stroke-linecap="round" stroke-linejoin="round" d="M3.75 6A2.25 2.25 0 0 1 6 3.75h2.25A2.25 2.25 0 0 1 10.5 6v2.25a2.25 2.25 0 0 1-2.25 2.25H6a2.25 2.25 0 0 1-2.25-2.25V6ZM3.75 15.75A2.25 2.25 0 0 1 6 13.5h2.25a2.25 2.25 0 0 1 2.25 2.25V18a2.25 2.25 0 0 1-2.25 2.25H6A2.25 2.25 0 0 1 3.75 18v-2.25ZM13.5 6a2.25 2.25 0 0 1 2.25-2.25H18A2.25 2.25 0 0 1 20.25 6v2.25A2.25 2.25 0 0 1 18 10.5h-2.25a2.25 2.25 0 0 1-2.25-2.25V6ZM13.5 15.75a2.25 2.25 0 0 1 2.25-2.25H18a2.25 2.25 0 0 1 2.25 2.25V18A2.25 2.25 0 0 1 18 20.25h-2.25A2.25 2.25 0 0 1 13.5 18v-2.25Z" />
							</svg>
							<span class="text-base font-medium text-gray-900 dark:text-gray-100">{$i18n.t('Embedding')}</span>
						</div>
						<div class="transform transition-transform duration-200 {expandedSections.embedding ? 'rotate-180' : ''}">
							<ChevronDown className="size-5 text-gray-400" />
						</div>
					</button>

					{#if expandedSections.embedding}
						<div transition:slide={{ duration: 200, easing: quintOut }} class="px-5 pb-5 space-y-4">
							<!-- 嵌入引擎选择 -->
							<div class="bg-white dark:bg-gray-900 rounded-lg p-4 border border-gray-100 dark:border-gray-800">
								<div class="flex items-center justify-between mb-3">
									<div class="text-sm font-medium">{$i18n.t('Embedding Model Engine')}</div>
									<select
										class="dark:bg-gray-850 w-fit pr-8 rounded-lg px-2 py-1.5 text-sm bg-gray-50 outline-none border border-gray-200 dark:border-gray-700 text-right cursor-pointer"
										bind:value={RAG_EMBEDDING_ENGINE}
										placeholder={$i18n.t('Select an embedding model engine')}
										on:change={(e) => {
											const target = e.target as HTMLSelectElement;
											if (target.value === 'ollama') {
												RAG_EMBEDDING_MODEL = '';
											} else if (target.value === 'openai') {
												RAG_EMBEDDING_MODEL = 'text-embedding-3-small';
											} else if (target.value === 'azure_openai') {
												RAG_EMBEDDING_MODEL = 'text-embedding-3-small';
											} else if (target.value === '') {
												RAG_EMBEDDING_MODEL = 'sentence-transformers/all-MiniLM-L6-v2';
											}
										}}
									>
										<option value="">{$i18n.t('Default (SentenceTransformers)')}</option>
										<option value="ollama">{$i18n.t('Ollama')}</option>
										<option value="openai">{$i18n.t('OpenAI')}</option>
										<option value="azure_openai">{$i18n.t('Azure OpenAI')}</option>
									</select>
								</div>

								{#if RAG_EMBEDDING_ENGINE === 'openai'}
									<div class="space-y-3 pt-2 border-t border-gray-100 dark:border-gray-800">
										<div class="flex w-full justify-between items-center">
											<div class="text-xs text-gray-500">{$i18n.t('API Base URL')}</div>
											<div class="w-1/2">
												<input
													class="w-full text-sm bg-transparent outline-hidden text-right"
													placeholder={$i18n.t('API Base URL')}
													bind:value={OpenAIUrl}
													required
												/>
											</div>
										</div>
										<div class="flex w-full justify-between items-center">
											<div class="text-xs text-gray-500">{$i18n.t('API Key')}</div>
											<div class="w-1/2">
												<SensitiveInput
													inputClassName="text-right w-full text-sm"
													placeholder={$i18n.t('API Key')}
													bind:value={OpenAIKey}
													required={false}
												/>
											</div>
										</div>
									</div>
								{:else if RAG_EMBEDDING_ENGINE === 'ollama'}
									<div class="space-y-3 pt-2 border-t border-gray-100 dark:border-gray-800">
										<div class="flex w-full justify-between items-center">
											<div class="text-xs text-gray-500">{$i18n.t('API Base URL')}</div>
											<div class="w-1/2">
												<input
													class="w-full text-sm bg-transparent outline-hidden text-right"
													placeholder={$i18n.t('API Base URL')}
													bind:value={OllamaUrl}
													required
												/>
											</div>
										</div>
										<div class="flex w-full justify-between items-center">
											<div class="text-xs text-gray-500">{$i18n.t('API Key')}</div>
											<div class="w-1/2">
												<SensitiveInput
													inputClassName="text-right w-full text-sm"
													placeholder={$i18n.t('API Key')}
													bind:value={OllamaKey}
													required={false}
												/>
											</div>
										</div>
									</div>
								{:else if RAG_EMBEDDING_ENGINE === 'azure_openai'}
									<div class="space-y-3 pt-2 border-t border-gray-100 dark:border-gray-800">
										<div class="flex w-full justify-between items-center">
											<div class="text-xs text-gray-500">{$i18n.t('API Base URL')}</div>
											<div class="w-1/2">
												<input
													class="w-full text-sm bg-transparent outline-hidden text-right"
													placeholder={$i18n.t('API Base URL')}
													bind:value={AzureOpenAIUrl}
													required
												/>
											</div>
										</div>
										<div class="flex w-full justify-between items-center">
											<div class="text-xs text-gray-500">{$i18n.t('API Key')}</div>
											<div class="w-1/2">
												<SensitiveInput
													inputClassName="text-right w-full text-sm"
													placeholder={$i18n.t('API Key')}
													bind:value={AzureOpenAIKey}
												/>
											</div>
										</div>
										<div class="flex w-full justify-between items-center">
											<div class="text-xs text-gray-500">{$i18n.t('Version')}</div>
											<div class="w-1/2">
												<input
													class="w-full text-sm bg-transparent outline-hidden text-right"
													placeholder={$i18n.t('Version')}
													bind:value={AzureOpenAIVersion}
													required
												/>
											</div>
										</div>
									</div>
								{/if}
								<div class=" mb-1 text-xs font-medium">{$i18n.t('Embedding Model')}</div>

								<div class="">
									{#if RAG_EMBEDDING_ENGINE === 'ollama'}
										<div class="flex w-full">
											<div class="flex-1 mr-2">
												<input
													class="flex-1 w-full rounded-lg py-2 px-4 text-sm bg-white dark:bg-gray-900 dark:text-gray-300 border border-gray-100 dark:border-gray-800 outline-none focus:border-gray-300 dark:focus:border-gray-700 transition"
													bind:value={RAG_EMBEDDING_MODEL}
													placeholder={$i18n.t('Set embedding model')}
													required
												/>
											</div>
										</div>
									{:else}
										<div class="flex w-full">
											<div class="flex-1 mr-2">
												<input
													class="flex-1 w-full rounded-lg py-2 px-4 text-sm bg-white dark:bg-gray-900 dark:text-gray-300 border border-gray-100 dark:border-gray-800 outline-none focus:border-gray-300 dark:focus:border-gray-700 transition"
													placeholder={$i18n.t('Set embedding model (e.g. {{model}})', {
														model: RAG_EMBEDDING_MODEL.slice(-40)
													})}
													bind:value={RAG_EMBEDDING_MODEL}
												/>
											</div>

											{#if RAG_EMBEDDING_ENGINE === ''}
												<button
													class="px-2.5 bg-transparent text-gray-800 dark:bg-transparent dark:text-gray-100 rounded-lg transition hover:bg-gray-100 dark:hover:bg-gray-800 hover:scale-105 active:scale-95"
													on:click={() => {
														embeddingModelUpdateHandler();
													}}
													disabled={updateEmbeddingModelLoading}
												>
													{#if updateEmbeddingModelLoading}
														<div class="self-center">
															<Spinner />
														</div>
													{:else}
														<svg
															xmlns="http://www.w3.org/2000/svg"
															viewBox="0 0 16 16"
															fill="currentColor"
															class="w-4 h-4"
														>
															<path
																d="M8.75 2.75a.75.75 0 0 0-1.5 0v5.69L5.03 6.22a.75.75 0 0 0-1.06 1.06l3.5 3.5a.75.75 0 0 0 1.06 0l3.5-3.5a.75.75 0 0 0-1.06-1.06L8.75 8.44V2.75Z"
															/>
															<path
																d="M3.5 9.75a.75.75 0 0 0-1.5 0v1.5A2.75 2.75 0 0 0 4.75 14h6.5A2.75 2.75 0 0 0 14 11.25v-1.5a.75.75 0 0 0-1.5 0v1.5c0 .69-.56 1.25-1.25 1.25h-6.5c-.69 0-1.25-.56-1.25-1.25v-1.5Z"
															/>
														</svg>
													{/if}
												</button>
											{/if}
										</div>
									{/if}
								</div>

								<div class="mt-1 mb-1 text-xs text-gray-400 dark:text-gray-500">
									{$i18n.t(
										'After updating or changing the embedding model, you must reindex the knowledge base for the changes to take effect. You can do this using the "Reindex" button below.'
									)}
								</div>
							</div>

							<div class="  mb-2.5 flex w-full justify-between">
								<div class=" self-center text-xs font-medium">
									{$i18n.t('Embedding Batch Size')}
								</div>

								<div class="">
									<input
										bind:value={RAG_EMBEDDING_BATCH_SIZE}
										type="number"
										class=" bg-transparent text-center w-14 outline-none"
										min="-2"
										max="16000"
										step="1"
									/>
								</div>
							</div>

							{#if RAG_EMBEDDING_ENGINE === 'ollama' || RAG_EMBEDDING_ENGINE === 'openai' || RAG_EMBEDDING_ENGINE === 'azure_openai'}
								<div class="  mb-2.5 flex w-full justify-between">
									<div class="self-center text-xs font-medium">
										<Tooltip
											content={$i18n.t(
												'Runs embedding tasks concurrently to speed up processing. Turn off if rate limits become an issue.'
											)}
											placement="top-start"
										>
											{$i18n.t('Async Embedding Processing')}
										</Tooltip>
									</div>
									<div class="flex items-center relative">
										<Switch bind:state={ENABLE_ASYNC_EMBEDDING} />
									</div>
								</div>
							{/if}
						</div>
					{/if}
				</div>

				<!-- 检索设置 Retrieval -->
				<div class="max-w-5xl mx-auto rounded-xl border border-gray-200 dark:border-gray-800">
					<button
						type="button"
						class="w-full flex items-center justify-between px-5 py-4 text-left"
						on:click={() => (expandedSections.retrieval = !expandedSections.retrieval)}
					>
						<div class="flex items-center gap-3">
							<svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="1.5" stroke="currentColor" class="size-5 text-gray-500">
								<path stroke-linecap="round" stroke-linejoin="round" d="m21 21-5.197-5.197m0 0A7.5 7.5 0 1 0 5.196 5.196a7.5 7.5 0 0 0 10.607 10.607Z" />
							</svg>
							<span class="text-base font-medium text-gray-900 dark:text-gray-100">{$i18n.t('Retrieval')}</span>
						</div>
						<div class="transform transition-transform duration-200 {expandedSections.retrieval ? 'rotate-180' : ''}">
							<ChevronDown className="size-5 text-gray-400" />
						</div>
					</button>

					{#if expandedSections.retrieval}
						<div transition:slide={{ duration: 200, easing: quintOut }} class="px-5 pb-5 space-y-4">
							<div class="bg-white dark:bg-gray-900 rounded-lg p-4 border border-gray-100 dark:border-gray-800">
								<div class="flex items-center justify-between mb-3">
									<div class="text-sm font-medium">{$i18n.t('Full Context Mode')}</div>
									<Tooltip
										content={RAGConfig.RAG_FULL_CONTEXT
											? $i18n.t('Inject the entire content as context for comprehensive processing, this is recommended for complex queries.')
											: $i18n.t('Default to segmented retrieval for focused and relevant content extraction, this is recommended for most cases.')}
									>
										<Switch bind:state={RAGConfig.RAG_FULL_CONTEXT} />
									</Tooltip>
								</div>

							{#if !RAGConfig.RAG_FULL_CONTEXT}
								<hr class="border-gray-100 dark:border-gray-800 my-2.5" />

								<div class="  mb-2.5 flex w-full justify-between">
									<div class=" self-center text-xs font-medium">{$i18n.t('Hybrid Search')}</div>
									<div class="flex items-center relative">
										<Switch bind:state={RAGConfig.ENABLE_RAG_HYBRID_SEARCH} />
									</div>
								</div>

								{#if RAGConfig.ENABLE_RAG_HYBRID_SEARCH === true}
									<hr class="border-gray-100 dark:border-gray-800 my-2.5" />

									<div class="mb-2.5 flex w-full justify-between">
										<div class="self-center text-xs font-medium">
											{$i18n.t('Enrich Hybrid Search Text')}
										</div>
										<div class="flex items-center relative">
											<Tooltip
												content={$i18n.t(
													'Adds filenames, titles, sections, and snippets into the BM25 text to improve lexical recall.'
												)}
											>
												<Switch bind:state={RAGConfig.ENABLE_RAG_HYBRID_SEARCH_ENRICHED_TEXTS} />
											</Tooltip>
										</div>
									</div>

									<hr class="border-gray-100 dark:border-gray-800 my-2.5" />

									<div class="  mb-2.5 flex flex-col w-full justify-between">
										<div class="flex w-full justify-between">
											<div class=" self-center text-xs font-medium">
												{$i18n.t('Reranking Engine')}
											</div>
											<div class="flex items-center relative">
												<select
													class="dark:bg-gray-900 w-fit pr-8 rounded-lg px-2 py-1 text-sm bg-transparent outline-none focus:ring-0 focus:border-gray-300 border-none text-right cursor-pointer"
													bind:value={RAGConfig.RAG_RERANKING_ENGINE}
													placeholder={$i18n.t('Select a reranking model engine')}
													on:change={(e) => {
														if (e.target.value === 'external') {
															RAGConfig.RAG_RERANKING_MODEL = '';
														} else if (e.target.value === '') {
															RAGConfig.RAG_RERANKING_MODEL = 'BAAI/bge-reranker-v2-m3';
														}
													}}
												>
													<option value="">{$i18n.t('Default (SentenceTransformers)')}</option>
													<option value="external">{$i18n.t('External')}</option>
												</select>
											</div>
										</div>

										{#if RAGConfig.RAG_RERANKING_ENGINE === 'external'}
											<hr class="border-gray-100 dark:border-gray-800 my-3" />

											<div class="flex flex-col gap-3">
												<div class="w-full">
													<div class="text-sm font-medium text-gray-700 dark:text-gray-300 mb-2">
														{$i18n.t('API Base URL')}
													</div>
													<input
														class="w-full rounded-lg py-2 px-4 text-sm bg-white dark:bg-gray-900 dark:text-gray-300 border border-gray-100 dark:border-gray-800 outline-none focus:border-gray-300 dark:focus:border-gray-700 transition"
														placeholder={$i18n.t('API Base URL')}
														bind:value={RAGConfig.RAG_EXTERNAL_RERANKER_URL}
														required
													/>
												</div>
												<div class="w-full">
													<div class="text-sm font-medium text-gray-700 dark:text-gray-300 mb-2">
														{$i18n.t('API Key')}
													</div>
													<div class="w-full rounded-lg py-2 px-4 text-sm bg-white dark:bg-gray-900 dark:text-gray-300 border border-gray-100 dark:border-gray-800 transition flex items-center">
														<SensitiveInput
															placeholder={$i18n.t('API Key')}
															bind:value={RAGConfig.RAG_EXTERNAL_RERANKER_API_KEY}
															required={false}
														/>
													</div>
												</div>
											</div>
										{/if}
									</div>

									<div class="  mb-2.5 flex flex-col w-full">
										<div class=" mb-1 text-xs font-medium">{$i18n.t('Reranking Model')}</div>

										<div class="">
											<div class="flex w-full">
												<div class="flex-1 mr-2">
													<input
														class="flex-1 w-full rounded-lg py-2 px-4 text-sm bg-white dark:bg-gray-900 dark:text-gray-300 border border-gray-100 dark:border-gray-800 outline-none focus:border-gray-300 dark:focus:border-gray-700 transition"
														placeholder={$i18n.t('Set reranking model (e.g. {{model}})', {
															model: 'BAAI/bge-reranker-v2-m3'
														})}
														bind:value={RAGConfig.RAG_RERANKING_MODEL}
													/>
												</div>
											</div>
										</div>
									</div>
								{/if}

								<div class="  mb-2.5 flex w-full justify-between">
									<div class=" self-center text-xs font-medium">{$i18n.t('Top K')}</div>
									<div class="flex items-center relative">
										<input
											class="flex-1 w-full rounded-lg py-2 px-4 text-sm bg-white dark:bg-gray-900 dark:text-gray-300 border border-gray-100 dark:border-gray-800 outline-none focus:border-gray-300 dark:focus:border-gray-700 transition"
											type="number"
											placeholder={$i18n.t('Enter Top K')}
											bind:value={RAGConfig.TOP_K}
											autocomplete="off"
											min="0"
										/>
									</div>
								</div>

								{#if RAGConfig.ENABLE_RAG_HYBRID_SEARCH === true}
									<hr class="border-gray-100 dark:border-gray-800 my-2.5" />

									<div class="mb-2.5 flex w-full justify-between">
										<div class="self-center text-xs font-medium">{$i18n.t('Top K Reranker')}</div>
										<div class="flex items-center relative">
											<input
												class="flex-1 w-full rounded-lg py-2 px-4 text-sm bg-white dark:bg-gray-900 dark:text-gray-300 border border-gray-100 dark:border-gray-800 outline-none focus:border-gray-300 dark:focus:border-gray-700 transition"
												type="number"
												placeholder={$i18n.t('Enter Top K Reranker')}
												bind:value={RAGConfig.TOP_K_RERANKER}
												autocomplete="off"
												min="0"
											/>
										</div>
									</div>

									<hr class="border-gray-100 dark:border-gray-800 my-2.5" />

									<div class="  mb-2.5 flex flex-col w-full justify-between">
										<div class=" flex w-full justify-between">
											<div class=" self-center text-xs font-medium">
												{$i18n.t('Relevance Threshold')}
											</div>
											<div class="flex items-center relative">
												<input
													class="flex-1 w-full rounded-lg py-2 px-4 text-sm bg-white dark:bg-gray-900 dark:text-gray-300 border border-gray-100 dark:border-gray-800 outline-none focus:border-gray-300 dark:focus:border-gray-700 transition"
													type="number"
													step="0.01"
													placeholder={$i18n.t('Enter Score')}
													bind:value={RAGConfig.RELEVANCE_THRESHOLD}
													autocomplete="off"
													min="0.0"
													title={$i18n.t(
														'The score should be a value between 0.0 (0%) and 1.0 (100%).'
													)}
												/>
											</div>
										</div>
										<div class="mt-1 text-xs text-gray-400 dark:text-gray-500">
											{$i18n.t(
												'Note: If you set a minimum score, the search will only return documents with a score greater than or equal to the minimum score.'
											)}
										</div>
									</div>

									<hr class="border-gray-100 dark:border-gray-800 my-2.5" />

									<div class="mb-2.5 py-0.5 w-full justify-between">
										<Tooltip
											content={$i18n.t(
												'The Weight of BM25 Hybrid Search. 0 more semantic, 1 more lexical. Default 0.5'
											)}
											placement="top-start"
											className="inline-tooltip"
										>
											<div class="flex w-full justify-between">
												<div class=" self-center text-xs font-medium">
													{$i18n.t('BM25 Weight')}
												</div>
												<button
													class="p-1 px-3 text-xs flex rounded-lg transition shrink-0 outline-none"
													type="button"
													on:click={() => {
														RAGConfig.HYBRID_BM25_WEIGHT =
															(RAGConfig?.HYBRID_BM25_WEIGHT ?? null) === null ? 0.5 : null;
													}}
												>
													{#if (RAGConfig?.HYBRID_BM25_WEIGHT ?? null) === null}
														<span class="ml-2 self-center"> {$i18n.t('Default')} </span>
													{:else}
														<span class="ml-2 self-center"> {$i18n.t('Custom')} </span>
													{/if}
												</button>
											</div>
										</Tooltip>

										{#if (RAGConfig?.HYBRID_BM25_WEIGHT ?? null) !== null}
											<div class="flex mt-2 space-x-3 items-center">
												<div class="flex-1 max-w-md">
													<input
														id="steps-range"
														type="range"
														min="0"
														max="1"
														step="0.05"
														bind:value={RAGConfig.HYBRID_BM25_WEIGHT}
														class="w-full h-2 rounded-lg appearance-none cursor-pointer dark:bg-gray-700"
													/>

													<div class="py-1">
														<div class="flex w-full justify-between">
															<div class="text-left text-xs text-gray-500">
																{$i18n.t('semantic')}
															</div>
															<div class="text-right text-xs text-gray-500">
																{$i18n.t('lexical')}
															</div>
														</div>
													</div>
												</div>
												<div>
													<input
														bind:value={RAGConfig.HYBRID_BM25_WEIGHT}
														type="number"
														class="w-16 rounded-lg py-1.5 px-2 text-sm bg-white dark:bg-gray-900 dark:text-gray-300 border border-gray-100 dark:border-gray-800 outline-none focus:border-gray-300 dark:focus:border-gray-700 transition text-center"
														min="0"
														max="1"
														step="any"
													/>
												</div>
											</div>
										{/if}
									</div>
								{/if}
							{/if}

							<div class="  mb-2.5 flex flex-col w-full justify-between">
								<div class=" mb-1 text-xs font-medium">{$i18n.t('RAG Template')}</div>
								<div class="flex w-full items-center relative">
									<Tooltip
										content={$i18n.t(
											'Leave empty to use the default prompt, or enter a custom prompt'
										)}
										placement="top-start"
										className="w-full"
									>
										<Textarea
											bind:value={RAGConfig.RAG_TEMPLATE}
											placeholder={$i18n.t(
												'Leave empty to use the default prompt, or enter a custom prompt'
											)}
										/>
									</Tooltip>
								</div>
							</div>
							</div>
						</div>
					{/if}
				</div>
			{/if}

			<!-- 文件设置 Files -->
			<div class="max-w-5xl mx-auto rounded-xl border border-gray-200 dark:border-gray-800">
				<button
					type="button"
					class="w-full flex items-center justify-between px-5 py-4 text-left"
					on:click={() => (expandedSections.files = !expandedSections.files)}
				>
					<div class="flex items-center gap-3">
						<svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="1.5" stroke="currentColor" class="size-5 text-gray-500">
							<path stroke-linecap="round" stroke-linejoin="round" d="M19.5 14.25v-2.625a3.375 3.375 0 0 0-3.375-3.375h-1.5A1.125 1.125 0 0 1 13.5 7.125v-1.5a3.375 3.375 0 0 0-3.375-3.375H8.25m2.25 0H5.625c-.621 0-1.125.504-1.125 1.125v17.25c0 .621.504 1.125 1.125 1.125h12.75c.621 0 1.125-.504 1.125-1.125V11.25a9 9 0 0 0-9-9Z" />
						</svg>
						<span class="text-base font-medium text-gray-900 dark:text-gray-100">{$i18n.t('Files')}</span>
					</div>
					<div class="transform transition-transform duration-200 {expandedSections.files ? 'rotate-180' : ''}">
						<ChevronDown className="size-5 text-gray-400" />
					</div>
				</button>

				{#if expandedSections.files}
					<div transition:slide={{ duration: 200, easing: quintOut }} class="px-5 pb-5 space-y-4">
						<div class="bg-white dark:bg-gray-900 rounded-lg p-4 border border-gray-100 dark:border-gray-800">
							<div class="grid grid-cols-1 md:grid-cols-2 gap-4">
								<div class="md:col-span-2">
									<div class="text-xs font-medium text-gray-500 mb-1.5">
										{$i18n.t('Allowed File Extensions')}
									</div>
									<Tooltip
										content={$i18n.t('Allowed file extensions for upload. Separate multiple extensions with commas. Leave empty for all file types.')}
										placement="top-start"
									>
										<input
											class="w-full rounded-lg py-2 px-4 text-sm bg-gray-50 dark:bg-gray-850 dark:text-gray-300 border border-gray-200 dark:border-gray-700 outline-none"
											type="text"
											placeholder={$i18n.t('e.g. pdf, docx, txt')}
											bind:value={RAGConfig.ALLOWED_FILE_EXTENSIONS}
											autocomplete="off"
										/>
									</Tooltip>
								</div>

								<div>
									<div class="text-xs font-medium text-gray-500 mb-1.5">
										{$i18n.t('Max Upload Size')}
									</div>
									<Tooltip
										content={$i18n.t('The maximum file size in MB. If the file size exceeds this limit, the file will not be uploaded.')}
										placement="top-start"
									>
										<input
											class="w-full rounded-lg py-2 px-4 text-sm bg-gray-50 dark:bg-gray-850 dark:text-gray-300 border border-gray-200 dark:border-gray-700 outline-none"
											type="number"
											placeholder={$i18n.t('Leave empty for unlimited')}
											bind:value={RAGConfig.FILE_MAX_SIZE}
											autocomplete="off"
											min="0"
										/>
									</Tooltip>
								</div>

								<div>
									<div class="text-xs font-medium text-gray-500 mb-1.5">
										{$i18n.t('Max Upload Count')}
									</div>
									<Tooltip
										content={$i18n.t('The maximum number of files that can be used at once in chat. If the number of files exceeds this limit, the files will not be uploaded.')}
										placement="top-start"
									>
										<input
											class="w-full rounded-lg py-2 px-4 text-sm bg-gray-50 dark:bg-gray-850 dark:text-gray-300 border border-gray-200 dark:border-gray-700 outline-none"
											type="number"
											placeholder={$i18n.t('Leave empty for unlimited')}
											bind:value={RAGConfig.FILE_MAX_COUNT}
											autocomplete="off"
											min="0"
										/>
									</Tooltip>
								</div>

								<div>
									<div class="text-xs font-medium text-gray-500 mb-1.5">
										{$i18n.t('Image Compression Width')}
									</div>
									<Tooltip
										content={$i18n.t('The width in pixels to compress images to. Leave empty for no compression.')}
										placement="top-start"
									>
										<input
											class="w-full rounded-lg py-2 px-4 text-sm bg-gray-50 dark:bg-gray-850 dark:text-gray-300 border border-gray-200 dark:border-gray-700 outline-none"
											type="number"
											placeholder={$i18n.t('Leave empty for no compression')}
											bind:value={RAGConfig.FILE_IMAGE_COMPRESSION_WIDTH}
											autocomplete="off"
											min="0"
										/>
									</Tooltip>
								</div>

								<div>
									<div class="text-xs font-medium text-gray-500 mb-1.5">
										{$i18n.t('Image Compression Height')}
									</div>
									<Tooltip
										content={$i18n.t('The height in pixels to compress images to. Leave empty for no compression.')}
										placement="top-start"
									>
										<input
											class="w-full rounded-lg py-2 px-4 text-sm bg-gray-50 dark:bg-gray-850 dark:text-gray-300 border border-gray-200 dark:border-gray-700 outline-none"
											type="number"
											placeholder={$i18n.t('Leave empty for no compression')}
											bind:value={RAGConfig.FILE_IMAGE_COMPRESSION_HEIGHT}
											autocomplete="off"
											min="0"
										/>
									</Tooltip>
								</div>
							</div>
						</div>
					</div>
				{/if}
			</div>

			<!-- 集成 Integration -->
			<div class="max-w-5xl mx-auto rounded-xl border border-gray-200 dark:border-gray-800">
				<button
					type="button"
					class="w-full flex items-center justify-between px-5 py-4 text-left"
					on:click={() => (expandedSections.integration = !expandedSections.integration)}
				>
					<div class="flex items-center gap-3">
						<svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="1.5" stroke="currentColor" class="size-5 text-gray-500">
							<path stroke-linecap="round" stroke-linejoin="round" d="M13.19 8.688a4.5 4.5 0 0 1 1.242 7.244l-4.5 4.5a4.5 4.5 0 0 1-6.364-6.364l1.757-1.757m13.35-.622 1.757-1.757a4.5 4.5 0 0 0-6.364-6.364l-4.5 4.5a4.5 4.5 0 0 0 1.242 7.244" />
						</svg>
						<span class="text-base font-medium text-gray-900 dark:text-gray-100">{$i18n.t('Integration')}</span>
					</div>
					<div class="transform transition-transform duration-200 {expandedSections.integration ? 'rotate-180' : ''}">
						<ChevronDown className="size-5 text-gray-400" />
					</div>
				</button>

				{#if expandedSections.integration}
					<div transition:slide={{ duration: 200, easing: quintOut }} class="px-5 pb-5 space-y-4">
						<div class="bg-white dark:bg-gray-900 rounded-lg p-4 border border-gray-100 dark:border-gray-800">
							<div class="space-y-3">
								<div class="flex items-center justify-between">
									<div class="text-sm font-medium">{$i18n.t('Google Drive')}</div>
									<Switch bind:state={RAGConfig.ENABLE_GOOGLE_DRIVE_INTEGRATION} />
								</div>
								<div class="flex items-center justify-between">
									<div class="text-sm font-medium">{$i18n.t('OneDrive')}</div>
									<Switch bind:state={RAGConfig.ENABLE_ONEDRIVE_INTEGRATION} />
								</div>
							</div>
						</div>
					</div>
				{/if}
			</div>

			<!-- 危险区域 Danger Zone -->
			<div class="max-w-5xl mx-auto rounded-xl border border-red-200 dark:border-red-900/50">
				<button
					type="button"
					class="w-full flex items-center justify-between px-5 py-4 text-left"
					on:click={() => (expandedSections.danger = !expandedSections.danger)}
				>
					<div class="flex items-center gap-3">
						<svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="1.5" stroke="currentColor" class="size-5 text-red-500">
							<path stroke-linecap="round" stroke-linejoin="round" d="M12 9v3.75m-9.303 3.376c-.866 1.5.217 3.374 1.948 3.374h14.71c1.73 0 2.813-1.874 1.948-3.374L13.949 3.378c-.866-1.5-3.032-1.5-3.898 0L2.697 16.126ZM12 15.75h.007v.008H12v-.008Z" />
						</svg>
						<span class="text-base font-medium text-red-600 dark:text-red-400">{$i18n.t('Danger Zone')}</span>
					</div>
					<div class="transform transition-transform duration-200 {expandedSections.danger ? 'rotate-180' : ''}">
						<ChevronDown className="size-5 text-gray-400" />
					</div>
				</button>

				{#if expandedSections.danger}
					<div transition:slide={{ duration: 200, easing: quintOut }} class="px-5 pb-5 space-y-4">
						<div class="bg-white dark:bg-gray-900 rounded-lg p-4 border border-gray-100 dark:border-gray-800">
							<div class="space-y-3">
								<div class="flex items-center justify-between">
									<div class="text-sm font-medium">{$i18n.t('Reset Upload Directory')}</div>
									<button
										class="px-3.5 py-1.5 font-medium text-xs bg-red-100 hover:bg-red-200 text-red-700 dark:bg-red-900/30 dark:hover:bg-red-900/50 dark:text-red-400 transition rounded-lg"
										type="button"
										on:click={() => { showResetUploadDirConfirm = true; }}
									>
										{$i18n.t('Reset')}
									</button>
								</div>

								<div class="flex items-center justify-between">
									<div class="text-sm font-medium">{$i18n.t('Reset Vector Storage/Knowledge')}</div>
									<button
										class="px-3.5 py-1.5 font-medium text-xs bg-red-100 hover:bg-red-200 text-red-700 dark:bg-red-900/30 dark:hover:bg-red-900/50 dark:text-red-400 transition rounded-lg"
										type="button"
										on:click={() => { showResetConfirm = true; }}
									>
										{$i18n.t('Reset')}
									</button>
								</div>

								<div class="flex items-center justify-between">
									<div class="text-sm font-medium">{$i18n.t('Reindex Knowledge Base Vectors')}</div>
									<button
										class="px-3.5 py-1.5 font-medium text-xs bg-gray-100 hover:bg-gray-200 text-gray-700 dark:bg-gray-850 dark:hover:bg-gray-800 dark:text-gray-100 transition rounded-lg"
										type="button"
										on:click={() => { showReindexConfirm = true; }}
									>
										{$i18n.t('Reindex')}
									</button>
								</div>
							</div>
						</div>
					</div>
				{/if}
			</div>
		</div>
		<div class="flex justify-end pt-3 text-sm font-medium">
			<button
				class="px-3.5 py-1.5 text-sm font-medium bg-black hover:bg-gray-900 text-white dark:bg-white dark:text-black dark:hover:bg-gray-100 transition rounded-full"
				type="submit"
			>
				{$i18n.t('Save')}
			</button>
		</div>
	{:else}
		<div class="flex items-center justify-center h-full">
			<Spinner className="size-5" />
		</div>
	{/if}
</form>

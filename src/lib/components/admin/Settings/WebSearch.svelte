<script lang="ts">
	import type { Writable } from 'svelte/store';
	import { getRAGConfig, updateRAGConfig } from '$lib/apis/retrieval';
	import Switch from '$lib/components/common/Switch.svelte';
	import { slide } from 'svelte/transition';
	import { quintOut } from 'svelte/easing';

	import { ragConfigCache } from '$lib/stores';
	import { onMount, getContext } from 'svelte';
	import SensitiveInput from '$lib/components/common/SensitiveInput.svelte';
	import Tooltip from '$lib/components/common/Tooltip.svelte';
	import ChevronDown from '$lib/components/icons/ChevronDown.svelte';

	const i18n: Writable<any> = getContext('i18n');

	export let saveHandler: Function;

	// 折叠状态
	let expandedSections = {
		webSearch: true,
		loader: false
	};

	let webSearchEngines = [
		'ollama_cloud',
		'perplexity_search',
		'searxng',
		'yacy',
		'google_pse',
		'brave',
		'kagi',
		'mojeek',
		'bocha',
		'serpstack',
		'serper',
		'serply',
		'searchapi',
		'serpapi',
		'duckduckgo',
		'tavily',
		'jina',
		'bing',
		'exa',
		'perplexity',
		'sougou',
		'firecrawl',
		'external'
	];
	let webLoaderEngines = ['playwright', 'firecrawl', 'tavily', 'external'];

	// 搜索引擎中文名称映射
	const engineNames: Record<string, string> = {
		'ollama_cloud': 'Ollama 云搜索',
		'perplexity_search': 'Perplexity 搜索',
		'searxng': 'SearXNG',
		'yacy': 'YaCy',
		'google_pse': 'Google 搜索',
		'brave': 'Brave 搜索',
		'kagi': 'Kagi 搜索',
		'mojeek': 'Mojeek 搜索',
		'bocha': 'Bocha搜索',
		'serpstack': 'Serpstack',
		'serper': 'Serper',
		'serply': 'Serply',
		'searchapi': 'SearchApi',
		'serpapi': 'SerpApi',
		'duckduckgo': 'DuckDuckGo',
		'tavily': 'Tavily',
		'jina': 'Jina 搜索',
		'bing': '必应搜索',
		'exa': 'Exa 搜索',
		'perplexity': 'Perplexity',
		'sougou': '搜狗搜索',
		'firecrawl': 'Firecrawl',
		'external': '外部搜索'
	};

	// 加载器引擎中文名称映射
	const loaderNames: Record<string, string> = {
		'playwright': 'Playwright',
		'firecrawl': 'Firecrawl',
		'tavily': 'Tavily',
		'external': '外部加载器'
	};

	let webConfig: any = null;

	const submitHandler = async () => {
		// Convert domain filter string to array before sending
		if (
			typeof webConfig.WEB_SEARCH_DOMAIN_FILTER_LIST === 'string' &&
			webConfig.WEB_SEARCH_DOMAIN_FILTER_LIST
		) {
			webConfig.WEB_SEARCH_DOMAIN_FILTER_LIST = webConfig.WEB_SEARCH_DOMAIN_FILTER_LIST.split(',')
				.map((domain: string) => domain.trim())
				.filter((domain: string) => domain.length > 0);
		} else if (!Array.isArray(webConfig.WEB_SEARCH_DOMAIN_FILTER_LIST)) {
			webConfig.WEB_SEARCH_DOMAIN_FILTER_LIST = [];
		}

		// Convert Youtube loader language string to array before sending
		if (
			typeof webConfig.YOUTUBE_LOADER_LANGUAGE === 'string' &&
			webConfig.YOUTUBE_LOADER_LANGUAGE
		) {
			webConfig.YOUTUBE_LOADER_LANGUAGE = webConfig.YOUTUBE_LOADER_LANGUAGE.split(',')
				.map((lang: string) => lang.trim())
				.filter((lang: string) => lang.length > 0);
		} else if (!Array.isArray(webConfig.YOUTUBE_LOADER_LANGUAGE)) {
			webConfig.YOUTUBE_LOADER_LANGUAGE = [];
		}

		// Convert numeric timeout values to strings (backend expects strings)
		if (typeof webConfig.FIRECRAWL_TIMEOUT === 'number') {
			webConfig.FIRECRAWL_TIMEOUT = webConfig.FIRECRAWL_TIMEOUT.toString();
		}
		if (typeof webConfig.PLAYWRIGHT_TIMEOUT === 'number') {
			webConfig.PLAYWRIGHT_TIMEOUT = webConfig.PLAYWRIGHT_TIMEOUT.toString();
		}

		await updateRAGConfig(localStorage.token, {
			web: webConfig
		} as any);

		// Clear cache after save to ensure fresh data on next visit
		ragConfigCache.set(null);

		// Convert arrays back to strings for display
		if (Array.isArray(webConfig.WEB_SEARCH_DOMAIN_FILTER_LIST)) {
			webConfig.WEB_SEARCH_DOMAIN_FILTER_LIST = webConfig.WEB_SEARCH_DOMAIN_FILTER_LIST.join(',');
		}
		if (Array.isArray(webConfig.YOUTUBE_LOADER_LANGUAGE)) {
			webConfig.YOUTUBE_LOADER_LANGUAGE = webConfig.YOUTUBE_LOADER_LANGUAGE.join(',');
		}
	};

	onMount(async () => {
		// Use cached RAG config if available
		let res;
		if ($ragConfigCache) {
			res = JSON.parse(JSON.stringify($ragConfigCache)); // Deep clone
		} else {
			res = await getRAGConfig(localStorage.token);
			ragConfigCache.set(res);
		}

		if (res) {
			webConfig = res.web;

			// Convert array back to comma-separated string for display
			if (Array.isArray(webConfig?.WEB_SEARCH_DOMAIN_FILTER_LIST)) {
				webConfig.WEB_SEARCH_DOMAIN_FILTER_LIST = webConfig.WEB_SEARCH_DOMAIN_FILTER_LIST.join(',');
			} else if (!webConfig.WEB_SEARCH_DOMAIN_FILTER_LIST) {
				webConfig.WEB_SEARCH_DOMAIN_FILTER_LIST = '';
			}

			if (Array.isArray(webConfig?.YOUTUBE_LOADER_LANGUAGE)) {
				webConfig.YOUTUBE_LOADER_LANGUAGE = webConfig.YOUTUBE_LOADER_LANGUAGE.join(',');
			} else if (!webConfig.YOUTUBE_LOADER_LANGUAGE) {
				webConfig.YOUTUBE_LOADER_LANGUAGE = '';
			}

			// Convert timeout strings to numbers for number input fields
			if (webConfig.FIRECRAWL_TIMEOUT && typeof webConfig.FIRECRAWL_TIMEOUT === 'string') {
				const parsed = parseInt(webConfig.FIRECRAWL_TIMEOUT);
				if (!isNaN(parsed)) {
					webConfig.FIRECRAWL_TIMEOUT = parsed;
				}
			}
			if (webConfig.PLAYWRIGHT_TIMEOUT && typeof webConfig.PLAYWRIGHT_TIMEOUT === 'string') {
				const parsed = parseInt(webConfig.PLAYWRIGHT_TIMEOUT);
				if (!isNaN(parsed)) {
					webConfig.PLAYWRIGHT_TIMEOUT = parsed;
				}
			}
		}
	});
</script>

<form
	class="flex flex-col h-full justify-between space-y-3 text-sm"
	on:submit|preventDefault={async () => {
		await submitHandler();
		saveHandler();
	}}
>
	<div class="overflow-y-scroll scrollbar-hidden h-full pr-2 space-y-3">
		{#if webConfig}
			<!-- 联网搜索 Web Search -->
			<div class="max-w-5xl mx-auto rounded-xl border border-gray-200 dark:border-gray-800">
				<button
					type="button"
					class="w-full flex items-center justify-between px-5 py-4 text-left"
					on:click={() => (expandedSections.webSearch = !expandedSections.webSearch)}
				>
					<div class="flex items-center gap-3">
						<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" fill="currentColor" class="size-5 text-gray-500">
							<path d="M21.721 12.752a9.711 9.711 0 0 0-.945-5.003 12.754 12.754 0 0 1-4.339 2.708 18.991 18.991 0 0 1-.214 4.772 17.165 17.165 0 0 0 5.498-2.477ZM14.634 15.55a17.324 17.324 0 0 0 .332-4.647c-.952.227-1.945.347-2.966.347-1.021 0-2.014-.12-2.966-.347a17.515 17.515 0 0 0 .332 4.647 17.385 17.385 0 0 0 5.268 0ZM9.772 17.119a18.963 18.963 0 0 0 4.456 0A17.182 17.182 0 0 1 12 21.724a17.18 17.18 0 0 1-2.228-4.605ZM7.777 15.23a18.87 18.87 0 0 1-.214-4.774 12.753 12.753 0 0 1-4.34-2.708 9.711 9.711 0 0 0-.944 5.004 17.165 17.165 0 0 0 5.498 2.477ZM21.356 14.752a9.765 9.765 0 0 1-7.478 6.817 18.64 18.64 0 0 0 1.988-4.718 18.627 18.627 0 0 0 5.49-2.098ZM2.644 14.752c1.682.971 3.53 1.688 5.49 2.099a18.64 18.64 0 0 0 1.988 4.718 9.765 9.765 0 0 1-7.478-6.816ZM13.878 2.43a9.755 9.755 0 0 1 6.116 3.986 11.267 11.267 0 0 1-3.746 2.504 18.63 18.63 0 0 0-2.37-6.49ZM12 2.276a17.152 17.152 0 0 1 2.805 7.121c-.897.23-1.837.353-2.805.353-.968 0-1.908-.122-2.805-.353A17.151 17.151 0 0 1 12 2.276ZM10.122 2.43a18.629 18.629 0 0 0-2.37 6.49 11.266 11.266 0 0 1-3.746-2.504 9.754 9.754 0 0 1 6.116-3.985Z" />
						</svg>
						<span class="text-base font-medium text-gray-900 dark:text-gray-100">{$i18n.t('Web Search')}</span>
					</div>
					<div class="flex items-center gap-3">
						<span role="button" tabindex="0" on:click|stopPropagation on:keydown|stopPropagation>
							<Switch bind:state={webConfig.ENABLE_WEB_SEARCH} />
						</span>
						<div class="transform transition-transform duration-200 {expandedSections.webSearch ? 'rotate-180' : ''}">
							<ChevronDown className="size-5 text-gray-400" />
						</div>
					</div>
				</button>

				{#if expandedSections.webSearch}
					<div transition:slide={{ duration: 200, easing: quintOut }} class="px-5 pb-5 space-y-4">
						<!-- 搜索引擎选择 -->
						<div class="bg-white dark:bg-gray-900 rounded-lg p-4 border border-gray-100 dark:border-gray-800">
							<div class="flex items-center justify-between mb-3">
								<div class="text-sm font-medium">{$i18n.t('Web Search Engine')}</div>
								<select
									class="dark:bg-gray-850 w-fit pr-8 rounded-lg px-2 py-1.5 text-sm bg-gray-50 outline-none border border-gray-200 dark:border-gray-700 text-right cursor-pointer"
									bind:value={webConfig.WEB_SEARCH_ENGINE}
									placeholder={$i18n.t('Select a engine')}
									required
								>
									<option disabled selected value="">{$i18n.t('Select a engine')}</option>
									{#each webSearchEngines as engine}
										<option value={engine}>{engineNames[engine] || engine}</option>
									{/each}
								</select>
							</div>

							{#if webConfig.WEB_SEARCH_ENGINE !== ''}
								<div class="space-y-3 pt-2 border-t border-gray-100 dark:border-gray-800">
									{#if webConfig.WEB_SEARCH_ENGINE === 'ollama_cloud'}
										<div class="flex w-full justify-between items-center">
											<div class="text-xs text-gray-500">{$i18n.t('Ollama Cloud API Key')}</div>
											<div class="w-1/2">
												<SensitiveInput inputClassName="text-right w-full text-sm" placeholder={$i18n.t('Enter Ollama Cloud API Key')} bind:value={webConfig.OLLAMA_CLOUD_WEB_SEARCH_API_KEY} />
											</div>
										</div>
									{:else if webConfig.WEB_SEARCH_ENGINE === 'perplexity_search'}
										<div class="flex w-full justify-between items-center">
											<div class="text-xs text-gray-500">{$i18n.t('Perplexity Search API URL')}</div>
											<div class="w-1/2">
												<input class="w-full text-sm bg-transparent outline-hidden text-right" type="text" placeholder={$i18n.t('Enter Perplexity Search API URL')} bind:value={webConfig.PERPLEXITY_SEARCH_API_URL} autocomplete="off" />
											</div>
										</div>
										<div class="flex w-full justify-between items-center">
											<div class="text-xs text-gray-500">{$i18n.t('Perplexity API Key')}</div>
											<div class="w-1/2">
												<SensitiveInput inputClassName="text-right w-full text-sm" placeholder={$i18n.t('Enter Perplexity API Key')} bind:value={webConfig.PERPLEXITY_API_KEY} />
											</div>
										</div>
									{:else if webConfig.WEB_SEARCH_ENGINE === 'searxng'}
										<div class="flex w-full justify-between items-center">
											<div class="text-xs text-gray-500">{$i18n.t('Searxng Query URL')}</div>
											<div class="w-1/2">
												<input class="w-full text-sm bg-transparent outline-hidden text-right" type="text" placeholder={$i18n.t('Enter Searxng Query URL')} bind:value={webConfig.SEARXNG_QUERY_URL} autocomplete="off" required />
											</div>
										</div>
										<div class="flex w-full justify-between items-center">
											<div class="text-xs text-gray-500">{$i18n.t('Searxng search language (all, en, es, de, fr, etc.)')}</div>
											<div class="w-1/2">
												<input class="w-full text-sm bg-transparent outline-hidden text-right" type="text" placeholder={$i18n.t('Enter Searxng search language')} bind:value={webConfig.SEARXNG_LANGUAGE} autocomplete="off" required />
											</div>
										</div>
									{:else if webConfig.WEB_SEARCH_ENGINE === 'yacy'}
										<div class="flex w-full justify-between items-center">
											<div class="text-xs text-gray-500">{$i18n.t('Yacy Instance URL')}</div>
											<div class="w-1/2">
												<input class="w-full text-sm bg-transparent outline-hidden text-right" type="text" placeholder={$i18n.t('Enter Yacy URL (e.g. http://yacy.example.com:8090)')} bind:value={webConfig.YACY_QUERY_URL} autocomplete="off" />
											</div>
										</div>
										<div class="flex w-full justify-between items-center">
											<div class="text-xs text-gray-500">{$i18n.t('Yacy Username')}</div>
											<div class="w-1/2">
												<input class="w-full text-sm bg-transparent outline-hidden text-right" placeholder={$i18n.t('Enter Yacy Username')} bind:value={webConfig.YACY_USERNAME} required />
											</div>
										</div>
										<div class="flex w-full justify-between items-center">
											<div class="text-xs text-gray-500">{$i18n.t('Yacy Password')}</div>
											<div class="w-1/2">
												<SensitiveInput inputClassName="text-right w-full text-sm" placeholder={$i18n.t('Enter Yacy Password')} bind:value={webConfig.YACY_PASSWORD} />
											</div>
										</div>
									{:else if webConfig.WEB_SEARCH_ENGINE === 'google_pse'}
										<div class="flex w-full justify-between items-center">
											<div class="text-xs text-gray-500">{$i18n.t('Google PSE API Key')}</div>
											<div class="w-1/2">
												<SensitiveInput inputClassName="text-right w-full text-sm" placeholder={$i18n.t('Enter Google PSE API Key')} bind:value={webConfig.GOOGLE_PSE_API_KEY} />
											</div>
										</div>
										<div class="flex w-full justify-between items-center">
											<div class="text-xs text-gray-500">{$i18n.t('Google PSE Engine Id')}</div>
											<div class="w-1/2">
												<input class="w-full text-sm bg-transparent outline-hidden text-right" type="text" placeholder={$i18n.t('Enter Google PSE Engine Id')} bind:value={webConfig.GOOGLE_PSE_ENGINE_ID} autocomplete="off" />
											</div>
										</div>
									{:else if webConfig.WEB_SEARCH_ENGINE === 'brave'}
										<div class="flex w-full justify-between items-center">
											<div class="text-xs text-gray-500">{$i18n.t('Brave Search API Key')}</div>
											<div class="w-1/2">
												<SensitiveInput inputClassName="text-right w-full text-sm" placeholder={$i18n.t('Enter Brave Search API Key')} bind:value={webConfig.BRAVE_SEARCH_API_KEY} />
											</div>
										</div>
									{:else if webConfig.WEB_SEARCH_ENGINE === 'kagi'}
										<div class="flex w-full justify-between items-center">
											<div class="text-xs text-gray-500">{$i18n.t('Kagi Search API Key')}</div>
											<div class="w-1/2">
												<SensitiveInput inputClassName="text-right w-full text-sm" placeholder={$i18n.t('Enter Kagi Search API Key')} bind:value={webConfig.KAGI_SEARCH_API_KEY} />
											</div>
										</div>
									{:else if webConfig.WEB_SEARCH_ENGINE === 'mojeek'}
										<div class="flex w-full justify-between items-center">
											<div class="text-xs text-gray-500">{$i18n.t('Mojeek Search API Key')}</div>
											<div class="w-1/2">
												<SensitiveInput inputClassName="text-right w-full text-sm" placeholder={$i18n.t('Enter Mojeek Search API Key')} bind:value={webConfig.MOJEEK_SEARCH_API_KEY} />
											</div>
										</div>
									{:else if webConfig.WEB_SEARCH_ENGINE === 'bocha'}
										<div class="flex w-full justify-between items-center">
											<div class="text-xs text-gray-500">{$i18n.t('Bocha Search API Key')}</div>
											<div class="w-1/2">
												<SensitiveInput inputClassName="text-right w-full text-sm" placeholder={$i18n.t('Enter Bocha Search API Key')} bind:value={webConfig.BOCHA_SEARCH_API_KEY} />
											</div>
										</div>
									{:else if webConfig.WEB_SEARCH_ENGINE === 'serpstack'}
										<div class="flex w-full justify-between items-center">
											<div class="text-xs text-gray-500">{$i18n.t('Serpstack API Key')}</div>
											<div class="w-1/2">
												<SensitiveInput inputClassName="text-right w-full text-sm" placeholder={$i18n.t('Enter Serpstack API Key')} bind:value={webConfig.SERPSTACK_API_KEY} />
											</div>
										</div>
									{:else if webConfig.WEB_SEARCH_ENGINE === 'serper'}
										<div class="flex w-full justify-between items-center">
											<div class="text-xs text-gray-500">{$i18n.t('Serper API Key')}</div>
											<div class="w-1/2">
												<SensitiveInput inputClassName="text-right w-full text-sm" placeholder={$i18n.t('Enter Serper API Key')} bind:value={webConfig.SERPER_API_KEY} />
											</div>
										</div>
									{:else if webConfig.WEB_SEARCH_ENGINE === 'serply'}
										<div class="flex w-full justify-between items-center">
											<div class="text-xs text-gray-500">{$i18n.t('Serply API Key')}</div>
											<div class="w-1/2">
												<SensitiveInput inputClassName="text-right w-full text-sm" placeholder={$i18n.t('Enter Serply API Key')} bind:value={webConfig.SERPLY_API_KEY} />
											</div>
										</div>
									{:else if webConfig.WEB_SEARCH_ENGINE === 'tavily'}
										<div class="flex w-full justify-between items-center">
											<div class="text-xs text-gray-500">{$i18n.t('Tavily API Key')}</div>
											<div class="w-1/2">
												<SensitiveInput inputClassName="text-right w-full text-sm" placeholder={$i18n.t('Enter Tavily API Key')} bind:value={webConfig.TAVILY_API_KEY} />
											</div>
										</div>
									{:else if webConfig.WEB_SEARCH_ENGINE === 'searchapi'}
										<div class="flex w-full justify-between items-center">
											<div class="text-xs text-gray-500">{$i18n.t('SearchApi API Key')}</div>
											<div class="w-1/2">
												<SensitiveInput inputClassName="text-right w-full text-sm" placeholder={$i18n.t('Enter SearchApi API Key')} bind:value={webConfig.SEARCHAPI_API_KEY} />
											</div>
										</div>
										<div class="flex w-full justify-between items-center">
											<div class="text-xs text-gray-500">{$i18n.t('SearchApi Engine')}</div>
											<div class="w-1/2">
												<input class="w-full text-sm bg-transparent outline-hidden text-right" type="text" placeholder={$i18n.t('Enter SearchApi Engine')} bind:value={webConfig.SEARCHAPI_ENGINE} autocomplete="off" />
											</div>
										</div>
									{:else if webConfig.WEB_SEARCH_ENGINE === 'serpapi'}
										<div class="flex w-full justify-between items-center">
											<div class="text-xs text-gray-500">{$i18n.t('SerpApi API Key')}</div>
											<div class="w-1/2">
												<SensitiveInput inputClassName="text-right w-full text-sm" placeholder={$i18n.t('Enter SerpApi API Key')} bind:value={webConfig.SERPAPI_API_KEY} />
											</div>
										</div>
										<div class="flex w-full justify-between items-center">
											<div class="text-xs text-gray-500">{$i18n.t('SerpApi Engine')}</div>
											<div class="w-1/2">
												<input class="w-full text-sm bg-transparent outline-hidden text-right" type="text" placeholder={$i18n.t('Enter SerpApi Engine')} bind:value={webConfig.SERPAPI_ENGINE} autocomplete="off" />
											</div>
										</div>
									{:else if webConfig.WEB_SEARCH_ENGINE === 'jina'}
										<div class="flex w-full justify-between items-center">
											<div class="text-xs text-gray-500">{$i18n.t('Jina API Base URL')}</div>
											<div class="w-1/2">
												<input class="w-full text-sm bg-transparent outline-hidden text-right" type="text" placeholder={$i18n.t('Enter Jina API Base URL')} bind:value={webConfig.JINA_API_BASE_URL} autocomplete="off" />
											</div>
										</div>
										<div class="flex w-full justify-between items-center">
											<div class="text-xs text-gray-500">{$i18n.t('Jina API Key')}</div>
											<div class="w-1/2">
												<SensitiveInput inputClassName="text-right w-full text-sm" placeholder={$i18n.t('Enter Jina API Key')} bind:value={webConfig.JINA_API_KEY} />
											</div>
										</div>
									{:else if webConfig.WEB_SEARCH_ENGINE === 'bing'}
										<div class="flex w-full justify-between items-center">
											<div class="text-xs text-gray-500">{$i18n.t('Bing Search V7 Endpoint')}</div>
											<div class="w-1/2">
												<input class="w-full text-sm bg-transparent outline-hidden text-right" type="text" placeholder={$i18n.t('Enter Bing Search V7 Endpoint')} bind:value={webConfig.BING_SEARCH_V7_ENDPOINT} autocomplete="off" />
											</div>
										</div>
										<div class="flex w-full justify-between items-center">
											<div class="text-xs text-gray-500">{$i18n.t('Bing Search V7 Subscription Key')}</div>
											<div class="w-1/2">
												<SensitiveInput inputClassName="text-right w-full text-sm" placeholder={$i18n.t('Enter Bing Search V7 Subscription Key')} bind:value={webConfig.BING_SEARCH_V7_SUBSCRIPTION_KEY} />
											</div>
										</div>
									{:else if webConfig.WEB_SEARCH_ENGINE === 'exa'}
										<div class="flex w-full justify-between items-center">
											<div class="text-xs text-gray-500">{$i18n.t('Exa API Key')}</div>
											<div class="w-1/2">
												<SensitiveInput inputClassName="text-right w-full text-sm" placeholder={$i18n.t('Enter Exa API Key')} bind:value={webConfig.EXA_API_KEY} />
											</div>
										</div>
									{:else if webConfig.WEB_SEARCH_ENGINE === 'perplexity'}
										<div class="flex w-full justify-between items-center">
											<div class="text-xs text-gray-500">{$i18n.t('Perplexity API Key')}</div>
											<div class="w-1/2">
												<SensitiveInput inputClassName="text-right w-full text-sm" placeholder={$i18n.t('Enter Perplexity API Key')} bind:value={webConfig.PERPLEXITY_API_KEY} />
											</div>
										</div>
										<div class="flex w-full justify-between items-center">
											<div class="text-xs text-gray-500">{$i18n.t('Perplexity Model')}</div>
											<div class="w-1/2">
												<input list="perplexity-model-list" class="w-full text-sm bg-transparent outline-hidden text-right" bind:value={webConfig.PERPLEXITY_MODEL} />
												<datalist id="perplexity-model-list">
													<option value="sonar">{$i18n.t('Sonar')}</option>
													<option value="sonar-pro">{$i18n.t('Sonar Pro')}</option>
													<option value="sonar-reasoning">{$i18n.t('Sonar Reasoning')}</option>
													<option value="sonar-reasoning-pro">{$i18n.t('Sonar Reasoning Pro')}</option>
													<option value="sonar-deep-research">{$i18n.t('Sonar Deep Research')}</option>
												</datalist>
											</div>
										</div>
										<div class="flex w-full justify-between items-center">
											<div class="text-xs text-gray-500">{$i18n.t('Perplexity Search Context Usage')}</div>
											<div class="w-1/2 flex justify-end">
												<select class="dark:bg-gray-850 w-fit pr-8 rounded-lg px-2 py-1 text-sm bg-gray-50 outline-none border border-gray-200 dark:border-gray-700 text-right cursor-pointer" bind:value={webConfig.PERPLEXITY_SEARCH_CONTEXT_USAGE}>
													<option value="low">{$i18n.t('Low')}</option>
													<option value="medium">{$i18n.t('Medium')}</option>
													<option value="high">{$i18n.t('High')}</option>
												</select>
											</div>
										</div>
									{:else if webConfig.WEB_SEARCH_ENGINE === 'sougou'}
										<div class="flex w-full justify-between items-center">
											<div class="text-xs text-gray-500">{$i18n.t('Sougou Search API sID')}</div>
											<div class="w-1/2">
												<SensitiveInput inputClassName="text-right w-full text-sm" placeholder={$i18n.t('Enter Sougou Search API sID')} bind:value={webConfig.SOUGOU_API_SID} />
											</div>
										</div>
										<div class="flex w-full justify-between items-center">
											<div class="text-xs text-gray-500">{$i18n.t('Sougou Search API SK')}</div>
											<div class="w-1/2">
												<SensitiveInput inputClassName="text-right w-full text-sm" placeholder={$i18n.t('Enter Sougou Search API SK')} bind:value={webConfig.SOUGOU_API_SK} />
											</div>
										</div>
									{:else if webConfig.WEB_SEARCH_ENGINE === 'firecrawl'}
										<div class="flex w-full justify-between items-center">
											<div class="text-xs text-gray-500">{$i18n.t('Firecrawl API Base URL')}</div>
											<div class="w-1/2">
												<input class="w-full text-sm bg-transparent outline-hidden text-right" type="text" placeholder={$i18n.t('Enter Firecrawl API Base URL')} bind:value={webConfig.FIRECRAWL_API_BASE_URL} autocomplete="off" />
											</div>
										</div>
										<div class="flex w-full justify-between items-center">
											<div class="text-xs text-gray-500">{$i18n.t('Firecrawl API Key')}</div>
											<div class="w-1/2">
												<SensitiveInput inputClassName="text-right w-full text-sm" placeholder={$i18n.t('Enter Firecrawl API Key')} bind:value={webConfig.FIRECRAWL_API_KEY} />
											</div>
										</div>
										<div class="flex w-full justify-between items-center">
											<div class="text-xs text-gray-500">{$i18n.t('Firecrawl Timeout (s)')}</div>
											<div class="w-1/2">
												<input class="w-full text-sm bg-transparent outline-hidden text-right" type="number" placeholder={$i18n.t('Enter Firecrawl Timeout')} bind:value={webConfig.FIRECRAWL_TIMEOUT} autocomplete="off" />
											</div>
										</div>
									{:else if webConfig.WEB_SEARCH_ENGINE === 'external'}
										<div class="flex w-full justify-between items-center">
											<div class="text-xs text-gray-500">{$i18n.t('External Web Search URL')}</div>
											<div class="w-1/2">
												<input class="w-full text-sm bg-transparent outline-hidden text-right" type="text" placeholder={$i18n.t('Enter External Web Search URL')} bind:value={webConfig.EXTERNAL_WEB_SEARCH_URL} autocomplete="off" />
											</div>
										</div>
										<div class="flex w-full justify-between items-center">
											<div class="text-xs text-gray-500">{$i18n.t('External Web Search API Key')}</div>
											<div class="w-1/2">
												<SensitiveInput inputClassName="text-right w-full text-sm" placeholder={$i18n.t('Enter External Web Search API Key')} bind:value={webConfig.EXTERNAL_WEB_SEARCH_API_KEY} />
											</div>
										</div>
									{/if}

									{#if webConfig.WEB_SEARCH_ENGINE === 'duckduckgo'}
										<div class="flex w-full justify-between items-center">
											<div class="text-xs text-gray-500">{$i18n.t('DDGS Backend')}</div>
											<div class="w-1/2 flex justify-end">
												<select class="dark:bg-gray-850 w-fit pr-8 rounded-lg px-2 py-1 text-sm bg-gray-50 outline-none border border-gray-200 dark:border-gray-700 text-right cursor-pointer" bind:value={webConfig.DDGS_BACKEND}>
													<option value="auto">{$i18n.t('Auto (Random)')}</option>
													<option value="bing">{$i18n.t('Bing')}</option>
													<option value="brave">{$i18n.t('Brave')}</option>
													<option value="duckduckgo">{$i18n.t('DuckDuckGo')}</option>
													<option value="google">{$i18n.t('Google')}</option>
													<option value="grokipedia">{$i18n.t('Grokipedia')}</option>
													<option value="mojeek">{$i18n.t('Mojeek')}</option>
													<option value="wikipedia">{$i18n.t('Wikipedia')}</option>
													<option value="yahoo">{$i18n.t('Yahoo')}</option>
													<option value="yandex">{$i18n.t('Yandex')}</option>
												</select>
											</div>
										</div>
									{/if}
								</div>
							{/if}
						</div>

						<!-- 搜索设置 -->
						{#if webConfig.ENABLE_WEB_SEARCH}
							<div class="bg-white dark:bg-gray-900 rounded-lg p-4 border border-gray-100 dark:border-gray-800">
								<div class="text-sm font-medium mb-3">{$i18n.t('Search Settings')}</div>
								<div class="space-y-3">
									<div class="flex w-full justify-between items-center">
										<div class="text-xs text-gray-500">{$i18n.t('Search Result Count')}</div>
										<div class="w-1/3">
											<input class="w-full text-sm bg-transparent outline-hidden text-right" placeholder={$i18n.t('Search Result Count')} bind:value={webConfig.WEB_SEARCH_RESULT_COUNT} required />
										</div>
									</div>

									<div class="flex w-full justify-between items-center">
										<div class="text-xs text-gray-500">
											<Tooltip content={$i18n.t('Limit concurrent search queries. 0 = unlimited (default). Set to 1 for sequential execution (recommended for APIs with strict rate limits like Brave free tier).')} placement="top-start">
												{$i18n.t('Concurrent Requests')}
											</Tooltip>
										</div>
										<div class="w-1/3">
											<input class="w-full text-sm bg-transparent outline-hidden text-right" placeholder={$i18n.t('Concurrent Requests')} bind:value={webConfig.WEB_SEARCH_CONCURRENT_REQUESTS} type="number" min="0" />
										</div>
									</div>

									<div class="flex w-full justify-between items-center">
										<div class="text-xs text-gray-500">{$i18n.t('Domain Filter List')}</div>
										<div class="w-1/2">
											<input class="w-full text-sm bg-transparent outline-hidden text-right" placeholder={$i18n.t('Enter domains separated by commas (e.g., example.com,site.org,!excludedsite.com)')} bind:value={webConfig.WEB_SEARCH_DOMAIN_FILTER_LIST} />
										</div>
									</div>
								</div>
							</div>
						{/if}

						<!-- 高级选项 -->
						<div class="bg-white dark:bg-gray-900 rounded-lg p-4 border border-gray-100 dark:border-gray-800">
							<div class="text-sm font-medium mb-3">{$i18n.t('Advanced Options')}</div>
							<div class="space-y-3">
								<div class="flex w-full justify-between items-center">
									<div class="text-xs text-gray-500">
										<Tooltip content={$i18n.t('Full Context Mode')} placement="top-start">
											{$i18n.t('Bypass Embedding and Retrieval')}
										</Tooltip>
									</div>
									<Tooltip content={webConfig.BYPASS_WEB_SEARCH_EMBEDDING_AND_RETRIEVAL ? $i18n.t('Inject the entire content as context for comprehensive processing, this is recommended for complex queries.') : $i18n.t('Default to segmented retrieval for focused and relevant content extraction, this is recommended for most cases.')}>
										<Switch bind:state={webConfig.BYPASS_WEB_SEARCH_EMBEDDING_AND_RETRIEVAL} />
									</Tooltip>
								</div>

								<div class="flex w-full justify-between items-center">
									<div class="text-xs text-gray-500">{$i18n.t('Bypass Web Loader')}</div>
									<Switch bind:state={webConfig.BYPASS_WEB_SEARCH_WEB_LOADER} />
								</div>

								<div class="flex w-full justify-between items-center">
									<div class="text-xs text-gray-500">{$i18n.t('Trust Proxy Environment')}</div>
									<Tooltip content={webConfig.WEB_SEARCH_TRUST_ENV ? $i18n.t('Use proxy designated by http_proxy and https_proxy environment variables to fetch page contents.') : $i18n.t('Use no proxy to fetch page contents.')}>
										<Switch bind:state={webConfig.WEB_SEARCH_TRUST_ENV} />
									</Tooltip>
								</div>
							</div>
						</div>
					</div>
				{/if}
			</div>

			<!-- 网页加载器 Loader -->
			<div class="max-w-5xl mx-auto rounded-xl border border-gray-200 dark:border-gray-800">
				<button
					type="button"
					class="w-full flex items-center justify-between px-5 py-4 text-left"
					on:click={() => (expandedSections.loader = !expandedSections.loader)}
				>
					<div class="flex items-center gap-3">
						<svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="1.5" stroke="currentColor" class="size-5 text-gray-500">
							<path stroke-linecap="round" stroke-linejoin="round" d="M12 9.75v6.75m0 0-3-3m3 3 3-3m-8.25 6a4.5 4.5 0 0 1-1.41-8.775 5.25 5.25 0 0 1 10.233-2.33 3 3 0 0 1 3.758 3.848A3.752 3.752 0 0 1 18 19.5H6.75Z" />
						</svg>
						<span class="text-base font-medium text-gray-900 dark:text-gray-100">{$i18n.t('Web Loader')}</span>
					</div>
					<div class="transform transition-transform duration-200 {expandedSections.loader ? 'rotate-180' : ''}">
						<ChevronDown className="size-5 text-gray-400" />
					</div>
				</button>

				{#if expandedSections.loader}
					<div transition:slide={{ duration: 200, easing: quintOut }} class="px-5 pb-5 space-y-4">
						<!-- 加载器引擎 -->
						<div class="bg-white dark:bg-gray-900 rounded-lg p-4 border border-gray-100 dark:border-gray-800">
							<div class="flex items-center justify-between mb-3">
								<div class="text-sm font-medium">{$i18n.t('Web Loader Engine')}</div>
								<select
									class="dark:bg-gray-850 w-fit pr-8 rounded-lg px-2 py-1.5 text-sm bg-gray-50 outline-none border border-gray-200 dark:border-gray-700 text-right cursor-pointer"
									bind:value={webConfig.WEB_LOADER_ENGINE}
									placeholder={$i18n.t('Select a engine')}
								>
									<option value="">{$i18n.t('Default')}</option>
									{#each webLoaderEngines as engine}
										<option value={engine}>{loaderNames[engine] || engine}</option>
									{/each}
								</select>
							</div>

							{#if webConfig.WEB_LOADER_ENGINE !== ''}
								<div class="space-y-3 pt-2 border-t border-gray-100 dark:border-gray-800">
									{#if webConfig.WEB_LOADER_ENGINE === '' || webConfig.WEB_LOADER_ENGINE === 'safe_web'}
										<div class="flex w-full justify-between items-center">
											<div class="text-xs text-gray-500">{$i18n.t('Timeout')}</div>
											<div class="w-1/3">
												<input class="w-full text-sm bg-transparent outline-hidden text-right" placeholder={$i18n.t('Timeout')} bind:value={webConfig.WEB_LOADER_TIMEOUT} />
											</div>
										</div>
										<div class="flex w-full justify-between items-center">
											<div class="text-xs text-gray-500">{$i18n.t('Verify SSL Certificate')}</div>
											<Switch bind:state={webConfig.ENABLE_WEB_LOADER_SSL_VERIFICATION} />
										</div>
									{:else if webConfig.WEB_LOADER_ENGINE === 'playwright'}
										<div class="flex w-full justify-between items-center">
											<div class="text-xs text-gray-500">{$i18n.t('Playwright WebSocket URL')}</div>
											<div class="w-1/2">
												<input class="w-full text-sm bg-transparent outline-hidden text-right" type="text" placeholder={$i18n.t('Enter Playwright WebSocket URL')} bind:value={webConfig.PLAYWRIGHT_WS_URL} autocomplete="off" />
											</div>
										</div>
										<div class="flex w-full justify-between items-center">
											<div class="text-xs text-gray-500">{$i18n.t('Playwright Timeout (ms)')}</div>
											<div class="w-1/3">
												<input class="w-full text-sm bg-transparent outline-hidden text-right" placeholder={$i18n.t('Enter Playwright Timeout')} bind:value={webConfig.PLAYWRIGHT_TIMEOUT} autocomplete="off" />
											</div>
										</div>
									{:else if webConfig.WEB_LOADER_ENGINE === 'firecrawl' && webConfig.WEB_SEARCH_ENGINE !== 'firecrawl'}
										<div class="flex w-full justify-between items-center">
											<div class="text-xs text-gray-500">{$i18n.t('Firecrawl API Base URL')}</div>
											<div class="w-1/2">
												<input class="w-full text-sm bg-transparent outline-hidden text-right" type="text" placeholder={$i18n.t('Enter Firecrawl API Base URL')} bind:value={webConfig.FIRECRAWL_API_BASE_URL} autocomplete="off" />
											</div>
										</div>
										<div class="flex w-full justify-between items-center">
											<div class="text-xs text-gray-500">{$i18n.t('Firecrawl API Key')}</div>
											<div class="w-1/2">
												<SensitiveInput inputClassName="text-right w-full text-sm" placeholder={$i18n.t('Enter Firecrawl API Key')} bind:value={webConfig.FIRECRAWL_API_KEY} />
											</div>
										</div>
									{:else if webConfig.WEB_LOADER_ENGINE === 'tavily'}
										<div class="flex w-full justify-between items-center">
											<div class="text-xs text-gray-500">{$i18n.t('Tavily Extract Depth')}</div>
											<div class="w-1/3">
												<input class="w-full text-sm bg-transparent outline-hidden text-right" type="text" placeholder={$i18n.t('Enter Tavily Extract Depth')} bind:value={webConfig.TAVILY_EXTRACT_DEPTH} autocomplete="off" />
											</div>
										</div>
										{#if webConfig.WEB_SEARCH_ENGINE !== 'tavily'}
											<div class="flex w-full justify-between items-center">
												<div class="text-xs text-gray-500">{$i18n.t('Tavily API Key')}</div>
												<div class="w-1/2">
													<SensitiveInput inputClassName="text-right w-full text-sm" placeholder={$i18n.t('Enter Tavily API Key')} bind:value={webConfig.TAVILY_API_KEY} />
												</div>
											</div>
										{/if}
									{:else if webConfig.WEB_LOADER_ENGINE === 'external'}
										<div class="flex w-full justify-between items-center">
											<div class="text-xs text-gray-500">{$i18n.t('External Web Loader URL')}</div>
											<div class="w-1/2">
												<input class="w-full text-sm bg-transparent outline-hidden text-right" type="text" placeholder={$i18n.t('Enter External Web Loader URL')} bind:value={webConfig.EXTERNAL_WEB_LOADER_URL} autocomplete="off" />
											</div>
										</div>
										<div class="flex w-full justify-between items-center">
											<div class="text-xs text-gray-500">{$i18n.t('External Web Loader API Key')}</div>
											<div class="w-1/2">
												<SensitiveInput inputClassName="text-right w-full text-sm" placeholder={$i18n.t('Enter External Web Loader API Key')} bind:value={webConfig.EXTERNAL_WEB_LOADER_API_KEY} />
											</div>
										</div>
									{/if}
								</div>
							{/if}
						</div>

						<!-- 通用加载器设置 -->
						<div class="bg-white dark:bg-gray-900 rounded-lg p-4 border border-gray-100 dark:border-gray-800">
							<div class="text-sm font-medium mb-3">{$i18n.t('Loader Settings')}</div>
							<div class="space-y-3">
								<div class="flex w-full justify-between items-center">
									<div class="text-xs text-gray-500">{$i18n.t('Concurrent Requests')}</div>
									<div class="w-1/3">
										<input class="w-full text-sm bg-transparent outline-hidden text-right" placeholder={$i18n.t('Concurrent Requests')} bind:value={webConfig.WEB_LOADER_CONCURRENT_REQUESTS} required />
									</div>
								</div>

								<div class="flex w-full justify-between items-center">
									<div class="text-xs text-gray-500">{$i18n.t('Youtube Language')}</div>
									<div class="w-1/2">
										<input class="w-full text-sm bg-transparent outline-hidden text-right" type="text" placeholder={$i18n.t('Enter language codes')} bind:value={webConfig.YOUTUBE_LOADER_LANGUAGE} autocomplete="off" />
									</div>
								</div>

								<div class="flex w-full justify-between items-center">
									<div class="text-xs text-gray-500">{$i18n.t('Youtube Proxy URL')}</div>
									<div class="w-1/2">
										<input class="w-full text-sm bg-transparent outline-hidden text-right" type="text" placeholder={$i18n.t('Enter proxy URL (e.g. https://user:password@host:port)')} bind:value={webConfig.YOUTUBE_LOADER_PROXY_URL} autocomplete="off" />
									</div>
								</div>
							</div>
						</div>
					</div>
				{/if}
			</div>
		{/if}
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

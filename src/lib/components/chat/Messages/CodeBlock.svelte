<script lang="ts">
	import hljs from 'highlight.js';
	import { loadPyodide } from 'pyodide';
	import mermaid from 'mermaid';

	import { v4 as uuidv4 } from 'uuid';

	import { getContext, getAllContexts, onMount, tick, createEventDispatcher } from 'svelte';
	import { copyToClipboard } from '$lib/utils';

	import 'highlight.js/styles/github-dark.min.css';

	//import { getSingleFileByPath } from '$lib/apis/files';
	import PyodideWorker from '$lib/workers/pyodide.worker?worker';
	import CodeEditor from '$lib/components/common/CodeEditor.svelte';
	import SvgPanZoom from '$lib/components/common/SVGPanZoom.svelte';

	const i18n = getContext('i18n');
	const dispatch = createEventDispatcher();

	export let id = '';
	export let save = false;

	export let token;
	export let lang = '';
	export let code = '';

	let _code = '';
	$: if (code) {
		updateCode();
	}

	const updateCode = () => {
		_code = code;
	};

	let _token = null;

	let mermaidHtml = null;

	let highlightedCode = null;
	let executing = false;

	let stdout = null;
	let stderr = null;
	let result = null;

	let copied = false;
	let saved = false;

	const saveCode = () => {
		saved = true;

		code = _code;
		dispatch('save', code);

		setTimeout(() => {
			saved = false;
		}, 1000);
	};

	const copyCode = async () => {
		copied = true;
		await copyToClipboard(code);

		setTimeout(() => {
			copied = false;
		}, 1000);
	};

	const checkPythonCode = (str) => {
		// Check if the string contains typical Python syntax characters
		const pythonSyntax = [
			'def ',
			'else:',
			'elif ',
			'try:',
			'except:',
			'finally:',
			'yield ',
			'lambda ',
			'assert ',
			'nonlocal ',
			'del ',
			'True',
			'False',
			'None',
			' and ',
			' or ',
			' not ',
			' in ',
			' is ',
			' with '
		];

		for (let syntax of pythonSyntax) {
			if (str.includes(syntax)) {
				return true;
			}
		}

		// If none of the above conditions met, it's probably not Python code
		return false;
	};

	const executePython = async (code) => {
		if (!code.includes('input') && !code.includes('matplotlib')) {
			executePythonAsWorker(code);
		} else {
			result = null;
			stdout = null;
			stderr = null;

			executing = true;

			document.pyodideMplTarget = document.getElementById(`plt-canvas-${id}`);

			let pyodide = await loadPyodide({
				indexURL: '/pyodide/',
				stdout: (text) => {
					console.log('Python output:', text);

					if (stdout) {
						stdout += `${text}\n`;
					} else {
						stdout = `${text}\n`;
					}
				},
				stderr: (text) => {
					console.log('An error occured:', text);
					if (stderr) {
						stderr += `${text}\n`;
					} else {
						stderr = `${text}\n`;
					}
				},
				packages: ['micropip']
			});

			try {
				const micropip = pyodide.pyimport('micropip');

				// await micropip.set_index_urls('https://pypi.org/pypi/{package_name}/json');

				let packages = [
					code.includes('requests') ? 'requests' : null,
					code.includes('bs4') ? 'beautifulsoup4' : null,
					code.includes('numpy') ? 'numpy' : null,
					code.includes('pandas') ? 'pandas' : null,
					code.includes('matplotlib') ? 'matplotlib' : null,
					code.includes('sklearn') ? 'scikit-learn' : null,
					code.includes('scipy') ? 'scipy' : null,
					code.includes('re') ? 'regex' : null,
					code.includes('seaborn') ? 'seaborn' : null
				].filter(Boolean);

				console.log(packages);
				await micropip.install(packages);

				result = await pyodide.runPythonAsync(`from js import prompt
def input(p):
    return prompt(p)
__builtins__.input = input`);

				result = await pyodide.runPython(code);

				if (!result) {
					result = '[NO OUTPUT]';
				}

				console.log(result);
				console.log(stdout);
				console.log(stderr);

				const pltCanvasElement = document.getElementById(`plt-canvas-${id}`);

				if (pltCanvasElement?.innerHTML !== '') {
					pltCanvasElement.classList.add('pt-4');
				}
			} catch (error) {
				console.error('Error:', error);
				stderr = error;
			}

			executing = false;
		}
	};

	const executePythonAsWorker = async (code) => {
		result = null;
		stdout = null;
		stderr = null;

		executing = true;

		let packages = [
			code.includes('requests') ? 'requests' : null,
			code.includes('bs4') ? 'beautifulsoup4' : null,
			code.includes('numpy') ? 'numpy' : null,
			code.includes('pandas') ? 'pandas' : null,
			code.includes('sklearn') ? 'scikit-learn' : null,
			code.includes('scipy') ? 'scipy' : null,
			code.includes('re') ? 'regex' : null,
			code.includes('seaborn') ? 'seaborn' : null
		].filter(Boolean);

		console.log(packages);

		const pyodideWorker = new PyodideWorker();

		pyodideWorker.postMessage({
			id: id,
			code: code,
			packages: packages
		});

		setTimeout(() => {
			if (executing) {
				executing = false;
				stderr = 'Execution Time Limit Exceeded';
				pyodideWorker.terminate();
			}
		}, 60000);

		pyodideWorker.onmessage = (event) => {
			console.log('pyodideWorker.onmessage', event);
			const { id, ...data } = event.data;

			console.log(id, data);

			data['stdout'] && (stdout = data['stdout']);
			data['stderr'] && (stderr = data['stderr']);
			data['result'] && (result = data['result']);

			executing = false;
		};

		pyodideWorker.onerror = (event) => {
			console.log('pyodideWorker.onerror', event);
			executing = false;
		};
	};

	let debounceTimeout;

	const drawMermaidDiagram = async () => {
		try {
			if (await mermaid.parse(code)) {
				const { svg } = await mermaid.render(`mermaid-${uuidv4()}`, code);
				mermaidHtml = svg;
			}
		} catch (error) {
			console.log('Error:', error);
		}
	};

	const render = async () => {
		if (lang === 'mermaid' && (token?.raw ?? '').slice(-4).includes('```')) {
			(async () => {
				await drawMermaidDiagram();
			})();
		}
	};

	$: if (token) {
		if (JSON.stringify(token) !== JSON.stringify(_token)) {
			_token = token;
		}
	}

	$: if (_token) {
		render();
	}

	$: dispatch('code', { lang, code });

	onMount(async () => {
		console.log('codeblock', lang, code);

		if (lang) {
			dispatch('code', { lang, code });
		}
		if (document.documentElement.classList.contains('dark')) {
			mermaid.initialize({
				startOnLoad: true,
				theme: 'dark',
				securityLevel: 'loose'
			});
		} else {
			mermaid.initialize({
				startOnLoad: true,
				theme: 'default',
				securityLevel: 'loose'
			});
		}
	});

	let error = "";
	// Function to fetch the file and trigger download automatically
	const fetchFileContent = async (filePath, options = "") => {
		console.log("triggered call with filepath ", filePath);
	try {
		console.log("Download File ", filePath)
		// Call the API to fetch the file
		//const response = await getSingleFileByPath(localStorage.token, filePath, options);

		// Create a blob URL from the response
		const blobUrl = window.URL.createObjectURL(response);

		// Extract the filename from the filePath (get the last part after the last '/')
		let fileName = filePath.split('/').pop();

		// Append ".xlsx" if options is not an empty string
		if (options !== "") {
			fileName += ".xlsx";
		}

		// Create a temporary link element to trigger download
		const link = document.createElement('a');
		link.href = blobUrl;
		link.setAttribute('download', fileName);  // Set the extracted filename

		// Append the link to the body (it needs to be in the DOM to trigger a click)
		document.body.appendChild(link);
		link.click();

		// Clean up: remove the link after triggering download
		document.body.removeChild(link);

		// Revoke the object URL after the download
		window.URL.revokeObjectURL(blobUrl);

		error = null; // Reset error if successful
	} catch (err) {
		console.error('Error fetching file:', err);
		error = err;
	}
	};

	function convertStringToDict(str) {
	// Step 1: Replace single quotes with double quotes for valid JSON
	const formattedStr = str.replace(/'/g, '"');
	
	// Step 2: Parse the formatted string to a dictionary
	try {
		console.log("Parsing success");
		return JSON.parse(formattedStr);
	} catch (error) {
		console.error("Error parsing JSON:", error);
		return {};
	}
	};

	// key and value need to be in double quotes.
	function convertStringToDictNormal(str) {
	
	// Step 2: Parse the formatted string to a dictionary
	try {
		let dummy_r = JSON.parse(str);
		console.log("Parsing AGENT CODE success");
		return dummy_r;
	} catch (error) {
		console.error("Error parsing Agent CodeJSON:", error);
		return {};
	}
	};

	// Use the function
	let citations = convertStringToDict(code);

	console.log("code", code);
	console.log("lang", lang);

	try {
		let dummy = `{"code_response": "Created charts <a href='http://localhost:8088/explore/?slice_id=757'", "file_name_ipynb": "def"}`
		dummy = `{"code_response": "Created charts <a href='http://localhost:8088/explore/?slice_id=757' target='_blank' style='color: magenta; text-decoration: none;'>Monthly Sales</a>, <a href='http://localhost:8088/explore/?slice_id=758' target='_blank' style='color: magenta; text-decoration: none;'>Sales Flow by Territory</a>.", "file_name_ipynb": "/Users/oliverkoehn/repos/private/chAIda/examples/generation/out/generated_code__85a75618-5ea2-461c-bbb2-9720f5718490.ipynb"}`
		let rrr = JSON.parse(code);
		console.log("rrr", rrr);
	} catch (error) {
		console.error("ERROR -------:", error);
	};

	let agent_code = convertStringToDictNormal(code);


</script>

<!--When we present generated code, it shal be toggled as most users are nto interested in. Foudn no other solution so far insteadt of xoping whoel block-->
{#if lang == "agent_result"}

<div>
	<!-- Render the HTML content -->
	<div>{@html agent_code.code_response}</div>
  
	<!-- Button in a new line -->
	<button on:click={() => fetchFileContent(agent_code.file_name_ipynb)} style="color: magenta;">
	  View generated code
	</button>
  </div>

{:else if lang === 'xlsx__src'}
<details>
	<summary style="color: magenta; font-size: 1.25em; font-weight: bold;">Source</summary>
<div>
	<div class="relative my-2 flex flex-col rounded-lg" dir="ltr">
		{#each Object.keys(citations) as key}
			<details>
				<summary>{key}</summary>
				{#if "fp_full" in citations[key]} 
				<button on:click={() => fetchFileContent(citations[key].fp_full)} style="color: magenta">
					Download Excel file.
				</button>
				{:else}
				<button on:click={() => fetchFileContent(key, "db")} style="color: magenta">
					Download Sample from Database.
				</button>
				{/if}

				<!-- Dynamic table for map__colname2type with inline styling -->
				<table style="width: 100%; border-collapse: collapse; font-family: Arial, sans-serif; background-color: #2e2e2e; color: #d3d3d3;">
					<thead>
						<tr style="background-color: #4a4a4a; color: #ffffff;">
							<th style="padding: 12px 15px; border: 1px solid #5e5e5e;">Column Name</th>
							<th style="padding: 12px 15px; border: 1px solid #5e5e5e;">Data Type</th>
						</tr>
					</thead>
					<tbody>
						{#each Object.entries(citations[key].map__colname2type) as [colname, dtype], index}
							<tr style="background-color: {index % 2 === 0 ? '#3b3b3b' : '#2e2e2e'};">
								<td style="padding: 12px 15px; border: 1px solid #5e5e5e;">{colname}</td>
								<td style="padding: 12px 15px; border: 1px solid #5e5e5e;">{dtype}</td>
							</tr>
						{/each}
					</tbody>
				</table>
			</details>
		{/each}
	</div>
</div>
</details>
{:else if ['xlsx__code', 'db__code'].includes(lang)}
<details>
	<summary style="color: magenta; font-size: 1.25em; font-weight: bold;">{lang === 'xlsx__code' ? 'Code' : lang === 'db__code' ? 'Code (Database)' : ''}</summary>
	<div>
	<div class="relative my-2 flex flex-col rounded-lg" dir="ltr">
		
		<div class="text-text-300 absolute pl-4 py-1.5 text-xs font-medium dark:text-white">
			{lang === 'xlsx__code' ? 'python' : lang === 'db__code' ? 'sql' : ''}
		</div>

		<div
			class="sticky top-8 mb-1 py-1 pr-2.5 flex items-center justify-end z-10 text-xs text-black dark:text-white"
		>
			<div class="flex items-center gap-0.5 translate-y-[1px]">
				{#if checkPythonCode(code)}
					{#if executing}
						<div class="run-code-button bg-none border-none p-1 cursor-not-allowed">Running</div>
					{:else}
						<button
							class="run-code-button bg-none border-none bg-gray-50 hover:bg-gray-100 dark:bg-gray-850 dark:hover:bg-gray-800 transition rounded-md px-1.5 py-0.5"
							on:click={async () => {
								code = _code;
								await tick();
								executePython(code);
							}}>{$i18n.t('Run')}</button
						>
					{/if}
				{/if}

				{#if save}
					<button
						class="save-code-button bg-none border-none bg-gray-50 hover:bg-gray-100 dark:bg-gray-850 dark:hover:bg-gray-800 transition rounded-md px-1.5 py-0.5"
						on:click={saveCode}
					>
						{saved ? $i18n.t('Saved') : $i18n.t('Save')}
					</button>
				{/if}

				<button
					class="copy-code-button bg-none border-none bg-gray-50 hover:bg-gray-100 dark:bg-gray-850 dark:hover:bg-gray-800 transition rounded-md px-1.5 py-0.5"
					on:click={copyCode}>{copied ? $i18n.t('Copied') : $i18n.t('Copy')}</button
				>
			</div>
		</div>

		<div
			class="language-python rounded-t-lg -mt-8 {executing || stdout || stderr || result
				? ''
				: 'rounded-b-lg'} overflow-hidden"
		>
			<div class=" pt-7 bg-gray-50 dark:bg-gray-850"></div>
			<CodeEditor
				value={code}
				{id}
				lang="python"
				on:save={() => {
					saveCode();
				}}
				on:change={(e) => {
					_code = e.detail.value;
				}}
			/>
		</div>

		<div
			id="plt-canvas-{id}"
			class="bg-[#202123] text-white max-w-full overflow-x-auto scrollbar-hidden"
		/>

		{#if executing}
			<div class="bg-[#202123] text-white px-4 py-4 rounded-b-lg">
				<div class=" text-gray-500 text-xs mb-1">STDOUT/STDERR</div>
				<div class="text-sm">Running...</div>
			</div>
		{:else if stdout || stderr || result}
			<div class="bg-[#202123] text-white px-4 py-4 rounded-b-lg">
				<div class=" text-gray-500 text-xs mb-1">STDOUT/STDERR</div>
				<div class="text-sm">{stdout || stderr || result}</div>
			</div>
		{/if}
	</div>
</div>
</details>

{:else}
<div>
	<div class="relative my-2 flex flex-col rounded-lg" dir="ltr">
		
		{#if lang === 'citations'}
		<!-- We dont wanna show stuff in this renderign step. It is calculated in outlet and shown in citations_styled -->
		<div></div>
		{:else if lang === 'xlsx__response'}
		<h3 style="color: magenta;">Response</h3>
		{@html code}
		{:else if lang === 'citations_styled'}
		<h3 style="color: magenta;">Sources</h3>
		{#if citations.web.length > 0}
        {#each citations.web as file}
            <li>
                <a href={file.url} download target="_blank" rel="noopener noreferrer" style="color: magenta">
					View Source {file.name}
				</a>				
            </li>
        {/each}
    	{/if}

		{#if citations.local.length > 0}
        {#each citations.local as file}
            <li>
                <button on:click={() => fetchFileContent(file.fp)} style="color: magenta">
                    Download Source {file.name}
                </button>
            </li>
        {/each}
    	{/if}

		{:else if lang === 'mermaid'}
			{#if mermaidHtml}
				<SvgPanZoom
					className=" border border-gray-50 dark:border-gray-850 rounded-lg max-h-fit overflow-hidden"
					svg={mermaidHtml}
				/>
			{:else}
				<pre class="mermaid">{code}</pre>
			{/if}
		{:else}
			<div class="text-text-300 absolute pl-4 py-1.5 text-xs font-medium dark:text-white">
				{lang}
			</div>

			<div
				class="sticky top-8 mb-1 py-1 pr-2.5 flex items-center justify-end z-10 text-xs text-black dark:text-white"
			>
				<div class="flex items-center gap-0.5 translate-y-[1px]">
					{#if lang.toLowerCase() === 'python' || lang.toLowerCase() === 'py' || (lang === '' && checkPythonCode(code))}
						{#if executing}
							<div class="run-code-button bg-none border-none p-1 cursor-not-allowed">Running</div>
						{:else}
							<button
								class="run-code-button bg-none border-none bg-gray-50 hover:bg-gray-100 dark:bg-gray-850 dark:hover:bg-gray-800 transition rounded-md px-1.5 py-0.5"
								on:click={async () => {
									code = _code;
									await tick();
									executePython(code);
								}}>{$i18n.t('Run')}</button
							>
						{/if}
					{/if}

					{#if save}
						<button
							class="save-code-button bg-none border-none bg-gray-50 hover:bg-gray-100 dark:bg-gray-850 dark:hover:bg-gray-800 transition rounded-md px-1.5 py-0.5"
							on:click={saveCode}
						>
							{saved ? $i18n.t('Saved') : $i18n.t('Save')}
						</button>
					{/if}

					<button
						class="copy-code-button bg-none border-none bg-gray-50 hover:bg-gray-100 dark:bg-gray-850 dark:hover:bg-gray-800 transition rounded-md px-1.5 py-0.5"
						on:click={copyCode}>{copied ? $i18n.t('Copied') : $i18n.t('Copy')}</button
					>
				</div>
			</div>

			<div
				class="language-{lang} rounded-t-lg -mt-8 {executing || stdout || stderr || result
					? ''
					: 'rounded-b-lg'} overflow-hidden"
			>
				<div class=" pt-7 bg-gray-50 dark:bg-gray-850"></div>
				<CodeEditor
					value={code}
					{id}
					{lang}
					on:save={() => {
						saveCode();
					}}
					on:change={(e) => {
						_code = e.detail.value;
					}}
				/>
			</div>

			<div
				id="plt-canvas-{id}"
				class="bg-[#202123] text-white max-w-full overflow-x-auto scrollbar-hidden"
			/>

			{#if executing}
				<div class="bg-[#202123] text-white px-4 py-4 rounded-b-lg">
					<div class=" text-gray-500 text-xs mb-1">STDOUT/STDERR</div>
					<div class="text-sm">Running...</div>
				</div>
			{:else if stdout || stderr || result}
				<div class="bg-[#202123] text-white px-4 py-4 rounded-b-lg">
					<div class=" text-gray-500 text-xs mb-1">STDOUT/STDERR</div>
					<div class="text-sm">{stdout || stderr || result}</div>
				</div>
			{/if}
		{/if}
	</div>
</div>
{/if}
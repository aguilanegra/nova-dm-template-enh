<script>
	export let templateContent = '';

	// State variables
	let componentTypes = [];
	let selectedComponent = null;
	let activeTab = 'components';
	let currentTheme = 'default';
	let template = null;

	// Parse the template when content is available
	$: if (templateContent) {
		parseTemplate(templateContent);
	}

	// Parse the template content
	function parseTemplate(content) {
		// Extract component types
		const typeRegex = /#(?:if|elseif)\( \$ROW\.getChild\('TYPE'\)\.getValue\(\) == '([A-Z_]+)' \)/g;
		const matches = [...content.matchAll(typeRegex)];
		componentTypes = [...new Set(matches.map((match) => match[1]))];

		// Extract themes
		const themeRegex = /#if\( \$theme == '([^']+)' \)/g;
		const themeMatches = [...content.matchAll(themeRegex)];
		const themes = [...new Set(themeMatches.map((match) => match[1]))];

		// Parse components and their properties
		const components = {};
		componentTypes.forEach((type) => {
			const componentRegex = new RegExp(
				`#(?:if|elseif)\\( \\$ROW\\.getChild\\('TYPE'\\)\\.getValue\\(\\) == '${type}' \\)([\\s\\S]*?)(?:#elseif|#else|#end)`,
				'g'
			);
			const componentMatches = [...content.matchAll(componentRegex)];

			if (componentMatches.length > 0) {
				const componentCode = componentMatches[0][1];

				// Extract properties
				const propRegex = /\$ROW\.getChild\('CONTENT'\)\.getChild\('([a-zA-Z0-9_]+)'\)/g;
				const propMatches = [...componentCode.matchAll(propRegex)];
				const properties = [...new Set(propMatches.map((match) => match[1]))];

				// Extract theme-specific properties
				const themeSpecificCode = [];
				themes.forEach((theme) => {
					const themeCodeRegex = new RegExp(
						`#if\\( \\$theme == '${theme}' \\)([\\s\\S]*?)(?:#else|#elseif|#end)`,
						'g'
					);
					const themeCodeMatches = [...componentCode.matchAll(themeCodeRegex)];

					if (themeCodeMatches.length > 0) {
						themeSpecificCode.push({
							theme,
							code: themeCodeMatches[0][1]
						});
					}
				});

				components[type] = {
					properties,
					themeSpecificCode,
					code: componentCode
				};
			}
		});

		// Extract CSS
		const cssRegex = /<style type="text\/css">([\s\S]*?)<\/style>/g;
		const cssMatches = [...content.matchAll(cssRegex)];
		const css = cssMatches.length > 0 ? cssMatches[0][1] : null;

		// Store the parsed template
		template = {
			componentTypes,
			components,
			themes: themes.length > 0 ? themes : ['default', 'AAW'],
			css
		};

		// Select the first component by default
		if (componentTypes.length > 0) {
			selectedComponent = componentTypes[0];
		}
	}

	// Get formatted properties for a component
	function getComponentProperties(componentType) {
		if (!template || !template.components[componentType]) {
			return [];
		}

		return template.components[componentType].properties;
	}

	// Check if a component has theme-specific styling
	function hasThemeSpecific(componentType) {
		if (!template || !template.components[componentType]) {
			return false;
		}

		return template.components[componentType].themeSpecificCode.length > 0;
	}

	// Clean up VTL code for display
	function formatCode(code) {
		return code
			.replace(/#if\([^)]+\)/g, '// if condition')
			.replace(/#elseif\([^)]+\)/g, '// else if condition')
			.replace(/#else/g, '// else')
			.replace(/#end/g, '// end condition')
			.replace(/#set\([^)]+\)/g, '// set variable')
			.replace(/\$ROW\.getChild\('CONTENT'\)\.getChild\('([^']+)'\)\.getValue\(\)/g, '{$1}');
	}

	// Get theme-specific code for a component
	function getThemeSpecificCode(componentType, theme) {
		if (!template || !template.components[componentType]) {
			return null;
		}

		const themeCode = template.components[componentType].themeSpecificCode.find(
			(tc) => tc.theme === theme
		);
		return themeCode ? themeCode.code : null;
	}
</script>

<div class="min-h-screen bg-gray-100">
	<!-- Header -->
	<header class="bg-blue-600 text-white shadow-md">
		<div class="container mx-auto px-4 py-4">
			<h1 class="text-2xl font-bold">Nova Email Template Parser</h1>
			<p class="text-blue-100">Analyze and explore Nova email template components and themes</p>
		</div>
	</header>

	<!-- Navigation Tabs -->
	<div class="border-b bg-white">
		<div class="container mx-auto px-4">
			<div class="flex">
				<button
					class="px-4 py-3 {activeTab === 'components'
						? 'border-b-2 border-blue-500 text-blue-600'
						: 'text-gray-600 hover:text-gray-800'}"
					on:click={() => (activeTab = 'components')}
				>
					Components
				</button>
				<button
					class="px-4 py-3 {activeTab === 'themes'
						? 'border-b-2 border-blue-500 text-blue-600'
						: 'text-gray-600 hover:text-gray-800'}"
					on:click={() => (activeTab = 'themes')}
				>
					Themes
				</button>
				<button
					class="px-4 py-3 {activeTab === 'css'
						? 'border-b-2 border-blue-500 text-blue-600'
						: 'text-gray-600 hover:text-gray-800'}"
					on:click={() => (activeTab = 'css')}
				>
					CSS
				</button>
			</div>
		</div>
	</div>

	<!-- Main Content -->
	<main class="container mx-auto px-4 py-6">
		{#if activeTab === 'components'}
			<div class="grid grid-cols-1 gap-6 md:grid-cols-4">
				<!-- Component List Sidebar -->
				<div class="rounded bg-white p-4 shadow">
					<h2 class="mb-3 text-lg font-semibold">Component Types</h2>
					<div class="space-y-1">
						{#each componentTypes as type}
							<button
								class="w-full rounded px-3 py-2 text-left {selectedComponent === type
									? 'bg-blue-100 text-blue-700'
									: 'hover:bg-gray-100'}"
								on:click={() => (selectedComponent = type)}
							>
								{type}
								{#if hasThemeSpecific(type)}
									<span
										class="ml-2 rounded-full bg-yellow-100 px-1.5 py-0.5 text-xs text-yellow-800"
										>Theme</span
									>
								{/if}
							</button>
						{/each}
					</div>
				</div>

				<!-- Component Details -->
				<div class="md:col-span-3">
					{#if selectedComponent}
						<div class="rounded bg-white p-6 shadow">
							<h2 class="mb-4 text-xl font-semibold">{selectedComponent} Component</h2>

							<!-- Component Properties -->
							<div class="mb-6">
								<h3 class="mb-2 text-lg font-medium">Properties</h3>
								{#if getComponentProperties(selectedComponent).length > 0}
									<div class="grid grid-cols-1 gap-2 md:grid-cols-2 xl:grid-cols-3">
										{#each getComponentProperties(selectedComponent) as prop}
											<div class="rounded bg-gray-50 px-3 py-2 font-mono text-sm">{prop}</div>
										{/each}
									</div>
								{:else}
									<p class="text-gray-500 italic">No properties found for this component</p>
								{/if}
							</div>

							<!-- Theme Specific Variations -->
							{#if hasThemeSpecific(selectedComponent)}
								<div class="mb-6">
									<h3 class="mb-2 text-lg font-medium">Theme Variations</h3>
									<div class="rounded bg-yellow-50 p-4">
										{#each template.components[selectedComponent].themeSpecificCode as themeCode}
											<div class="mb-2">
												<span class="font-semibold">{themeCode.theme}</span> theme has custom styling
											</div>
										{/each}
									</div>
								</div>
							{/if}

							<!-- Component Code Preview -->
							<div>
								<h3 class="mb-2 text-lg font-medium">Template Code</h3>
								<pre
									class="max-h-60 overflow-auto rounded-md bg-gray-800 p-4 text-xs text-gray-100">{formatCode(
										template.components[selectedComponent].code
									)}</pre>
							</div>
						</div>
					{:else}
						<div class="rounded bg-white p-6 text-center shadow">
							<p class="text-gray-500">Select a component from the list to view details</p>
						</div>
					{/if}
				</div>
			</div>
		{:else if activeTab === 'themes'}
			<div class="rounded bg-white p-6 shadow">
				<h2 class="mb-4 text-xl font-semibold">Theme Variables</h2>

				<div class="mb-6">
					<label class="mb-2 block text-sm font-medium text-gray-700">Select Theme</label>
					<select
						class="block w-full rounded-md border border-gray-300 px-3 py-2 focus:ring-2 focus:ring-blue-500 focus:outline-none md:w-64"
						bind:value={currentTheme}
					>
						{#each template.themes as theme}
							<option value={theme}>{theme}</option>
						{/each}
					</select>
				</div>

				<div class="mb-6">
					<h3 class="mb-3 text-lg font-medium">Theme Effects</h3>
					{#if currentTheme === 'AAW'}
						<div class="mb-4 rounded bg-blue-50 p-4">
							<p class="font-medium text-blue-800">AAW Theme Characteristics:</p>
							<ul class="mt-2 list-disc space-y-1 pl-5 text-blue-700">
								<li>Uses 600 font-weight (bold) for headlines</li>
								<li>Custom padding and margins</li>
								<li>#666666 color for certain text elements</li>
								<li>Specific alignment values</li>
							</ul>
						</div>
					{:else}
						<div class="mb-4 rounded bg-gray-50 p-4">
							<p class="font-medium">Default Theme Characteristics:</p>
							<ul class="mt-2 list-disc space-y-1 pl-5 text-gray-700">
								<li>Standard font weights</li>
								<li>Default padding and margins</li>
								<li>Standard color schemes</li>
							</ul>
						</div>
					{/if}
				</div>

				<div>
					<h3 class="mb-3 text-lg font-medium">Components with Theme-Specific Styling</h3>
					<div class="grid grid-cols-1 gap-4 md:grid-cols-2">
						{#each componentTypes.filter((type) => hasThemeSpecific(type)) as type}
							<div class="rounded border border-gray-200 bg-white p-4">
								<h4 class="font-medium">{type}</h4>
								<p class="mt-1 text-sm text-gray-600">
									Has custom styling for:
									{#each template.components[type].themeSpecificCode as themeCode, i}
										{themeCode.theme}{i < template.components[type].themeSpecificCode.length - 1
											? ', '
											: ''}
									{/each}
								</p>
							</div>
						{/each}
					</div>
				</div>
			</div>
		{:else if activeTab === 'css'}
			<div class="rounded bg-white p-6 shadow">
				<h2 class="mb-4 text-xl font-semibold">CSS Styles</h2>

				{#if template.css}
					<pre
						class="max-h-[600px] overflow-auto rounded-md bg-gray-800 p-4 text-xs text-gray-100">{template.css}</pre>
				{:else}
					<p class="text-gray-500 italic">No CSS styles found in the template</p>
				{/if}
			</div>
		{/if}
	</main>
</div>

<style>
	/* Tailwind-like utility classes */
	.container {
		width: 100%;
		max-width: 1280px;
	}
	.mx-auto {
		margin-left: auto;
		margin-right: auto;
	}
	.px-4 {
		padding-left: 1rem;
		padding-right: 1rem;
	}
	.py-4 {
		padding-top: 1rem;
		padding-bottom: 1rem;
	}
	.py-3 {
		padding-top: 0.75rem;
		padding-bottom: 0.75rem;
	}
	.py-6 {
		padding-top: 1.5rem;
		padding-bottom: 1.5rem;
	}
	.p-4 {
		padding: 1rem;
	}
	.p-6 {
		padding: 1.5rem;
	}
	.px-3 {
		padding-left: 0.75rem;
		padding-right: 0.75rem;
	}
	.py-2 {
		padding-top: 0.5rem;
		padding-bottom: 0.5rem;
	}
	.mt-4 {
		margin-top: 1rem;
	}
	.mt-2 {
		margin-top: 0.5rem;
	}
	.mt-1 {
		margin-top: 0.25rem;
	}
	.mb-2 {
		margin-bottom: 0.5rem;
	}
	.mb-3 {
		margin-bottom: 0.75rem;
	}
	.mb-4 {
		margin-bottom: 1rem;
	}
	.mb-6 {
		margin-bottom: 1.5rem;
	}
	.ml-2 {
		margin-left: 0.5rem;
	}
	.flex {
		display: flex;
	}
	.grid {
		display: grid;
	}
	.h-screen {
		height: 100vh;
	}
	.h-10 {
		height: 2.5rem;
	}
	.w-10 {
		width: 2.5rem;
	}
	.w-full {
		width: 100%;
	}
	.space-y-1 > :not([hidden]) ~ :not([hidden]) {
		margin-top: 0.25rem;
	}
	.rounded {
		border-radius: 0.25rem;
	}
	.rounded-md {
		border-radius: 0.375rem;
	}
	.rounded-full {
		border-radius: 9999px;
	}
	.border {
		border-width: 1px;
	}
	.border-4 {
		border-width: 4px;
	}
	.border-b {
		border-bottom-width: 1px;
	}
	.border-b-2 {
		border-bottom-width: 2px;
	}
	.border-blue-500 {
		border-color: rgb(59, 130, 246);
	}
	.border-red-400 {
		border-color: rgb(248, 113, 113);
	}
	.border-gray-200 {
		border-color: rgb(229, 231, 235);
	}
	.border-gray-300 {
		border-color: rgb(209, 213, 219);
	}
	.border-t-transparent {
		border-top-color: transparent;
	}
	.shadow {
		box-shadow:
			0 1px 3px 0 rgba(0, 0, 0, 0.1),
			0 1px 2px 0 rgba(0, 0, 0, 0.06);
	}
	.shadow-md {
		box-shadow:
			0 4px 6px -1px rgba(0, 0, 0, 0.1),
			0 2px 4px -1px rgba(0, 0, 0, 0.06);
	}
	.text-center {
		text-align: center;
	}
	.text-left {
		text-align: left;
	}
	.font-bold {
		font-weight: 700;
	}
	.font-semibold {
		font-weight: 600;
	}
	.font-medium {
		font-weight: 500;
	}
	.font-mono {
		font-family: ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, monospace;
	}
	.text-2xl {
		font-size: 1.5rem;
	}
	.text-xl {
		font-size: 1.25rem;
	}
	.text-lg {
		font-size: 1.125rem;
	}
	.text-sm {
		font-size: 0.875rem;
	}
	.text-xs {
		font-size: 0.75rem;
	}
	.text-gray-500 {
		color: rgb(107, 114, 128);
	}
	.text-gray-600 {
		color: rgb(75, 85, 99);
	}
	.text-gray-700 {
		color: rgb(55, 65, 81);
	}
	.text-gray-800 {
		color: rgb(31, 41, 55);
	}
	.text-gray-100 {
		color: rgb(243, 244, 246);
	}
	.text-blue-500 {
		color: rgb(59, 130, 246);
	}
	.text-blue-600 {
		color: rgb(37, 99, 235);
	}
	.text-blue-700 {
		color: rgb(29, 78, 216);
	}
	.text-blue-800 {
		color: rgb(30, 64, 175);
	}
	.text-blue-100 {
		color: rgb(219, 234, 254);
	}
	.text-red-700 {
		color: rgb(185, 28, 28);
	}
	.text-yellow-800 {
		color: rgb(146, 64, 14);
	}
	.bg-white {
		background-color: white;
	}
	.bg-blue-600 {
		background-color: rgb(37, 99, 235);
	}
	.bg-blue-500 {
		background-color: rgb(59, 130, 246);
	}
	.bg-blue-100 {
		background-color: rgb(219, 234, 254);
	}
	.bg-blue-50 {
		background-color: rgb(239, 246, 255);
	}
	.bg-gray-100 {
		background-color: rgb(243, 244, 246);
	}
	.bg-gray-50 {
		background-color: rgb(249, 250, 251);
	}
	.bg-gray-800 {
		background-color: rgb(31, 41, 55);
	}
	.bg-red-100 {
		background-color: rgb(254, 226, 226);
	}
	.bg-yellow-50 {
		background-color: rgb(254, 252, 232);
	}
	.bg-yellow-100 {
		background-color: rgb(254, 249, 195);
	}
	.min-h-screen {
		min-height: 100vh;
	}
	.max-w-md {
		max-width: 28rem;
	}
	.max-h-60 {
		max-height: 15rem;
	}
	.overflow-auto {
		overflow: auto;
	}
	.italic {
		font-style: italic;
	}
	.items-center {
		align-items: center;
	}
	.justify-center {
		justify-content: center;
	}

	/* Responsive styles */
	@media (min-width: 768px) {
		.md\:grid-cols-4 {
			grid-template-columns: repeat(4, minmax(0, 1fr));
		}
		.md\:grid-cols-2 {
			grid-template-columns: repeat(2, minmax(0, 1fr));
		}
		.md\:col-span-3 {
			grid-column: span 3 / span 3;
		}
		.md\:w-64 {
			width: 16rem;
		}
	}

	@media (min-width: 1280px) {
		.xl\:grid-cols-3 {
			grid-template-columns: repeat(3, minmax(0, 1fr));
		}
	}
</style>

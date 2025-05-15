<script>
	import { onMount } from 'svelte';
	import NovaTemplateParser from '$lib/components/NovaTemplateParser.svelte';

	let templateContent = '';
	let loading = true;
	let error = null;

	onMount(async () => {
		try {
			// Fetch the HTML template file
			const response = await fetch('html.txt');
			if (!response.ok) {
				throw new Error(`Failed to load template file: ${response.statusText}`);
			}

			templateContent = await response.text();
			loading = false;
		} catch (err) {
			console.error('Error loading template:', err);
			error = err.message || 'Failed to load the template file';
			loading = false;
		}
	});
</script>

<svelte:head>
	<title>Nova Email Template Parser</title>
</svelte:head>

{#if loading}
	<div class="loading-container">
		<div class="spinner"></div>
		<p>Loading template file...</p>
	</div>
{:else if error}
	<div class="error-container">
		<div class="error-message">
			<h2>Error</h2>
			<p>{error}</p>
			<button on:click={() => window.location.reload()}>Retry</button>
		</div>
	</div>
{:else}
	<NovaTemplateParser {templateContent} />
{/if}

<style>
	.loading-container,
	.error-container {
		display: flex;
		justify-content: center;
		align-items: center;
		height: 100vh;
	}

	.spinner {
		width: 50px;
		height: 50px;
		border: 5px solid #f3f3f3;
		border-top: 5px solid #3498db;
		border-radius: 50%;
		animation: spin 1s linear infinite;
		margin-bottom: 20px;
	}

	.error-container {
		padding: 1rem;
	}

	.error-message {
		background-color: #fff3f3;
		border: 1px solid #ffcdd2;
		border-radius: 0.5rem;
		padding: 2rem;
		max-width: 500px;
		text-align: center;
	}

	.error-message button {
		background-color: #3498db;
		color: white;
		border: none;
		padding: 0.5rem 1rem;
		border-radius: 0.25rem;
		margin-top: 1rem;
		cursor: pointer;
	}

	@keyframes spin {
		0% {
			transform: rotate(0deg);
		}
		100% {
			transform: rotate(360deg);
		}
	}
</style>

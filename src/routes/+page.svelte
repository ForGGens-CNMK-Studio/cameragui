<script lang="ts">
	import { env } from '$env/dynamic/public';
	import { onMount } from 'svelte';
	import html2canvas from 'html2canvas';

	let screenshotArea: HTMLDivElement;

	async function takeScreenShot() {
		const canvas = await html2canvas(screenshotArea, {
			allowTaint: true,
			useCORS: true,
			removeContainer: true,
			backgroundColor: 'blue'
		});

		const link = document.createElement('a');
		link.download = `data-${Date.now()}.png`;
		link.href = canvas.toDataURL('image/png');
		link.click();
	}

	async function playStream() {
		const response = await fetch(`${env.PUBLIC_API_ROOT}/control?action=play`, {
			method: 'POST'
		});
		const data = await response.json();
		console.log(data);
	}

	async function pauseStream() {
		const response = await fetch(`${env.PUBLIC_API_ROOT}/control?action=pause`, {
			method: 'POST'
		});
		const data = await response.json();
		console.log(data);
	}

	type HeightData = number[];

	let heights = [];

	async function fetchHeights() {
		const response = await fetch(`${env.PUBLIC_API_ROOT}/height_feed`);
		if (!response.body) {
			console.error('Response body is null');
			return;
		}
		const reader = response.body.getReader();
		const decoder = new TextDecoder('utf-8');

		let buffer = '';

		while (true) {
			const { done, value } = await reader.read();
			if (done) break;

			buffer += decoder.decode(value, { stream: true });

			// Split by the boundary
			const parts = buffer.split('--height');

			for (const part of parts) {
				console.log(part);
				const match = part.match(/Content-Type: application\/json\r\n\r\n(.+?)\r\n/);
				if (match) {
					const data = JSON.parse(match[1]);
					console.log(data);
					heights = data; // Update the Svelte field
				}
			}

			// Reset buffer
			buffer = parts[parts.length - 1];
		}
	}

	// Fetch heights once component is mounted
	onMount(() => {
		fetchHeights();
	});
</script>

<div class="container h-full mx-auto flex justify-center items-center">
	<div class="space-y-10 text-center flex flex-col items-center">
		<h2 class="h2">Welcome to ForGGEns.</h2>
		<div class="grid grid-cols-2 items-center justify-center m-16" bind:this={screenshotArea}>
			<figure class="w-[720px]">
				<img src={`${env.PUBLIC_API_ROOT}/video_feed`} width="720" height="480" />
			</figure>
			{#if heights.length > 0}
				<ul>
					{#each heights as height, index}
						<li>Person {index + 1}: {height} cm</li>
					{/each}
				</ul>
			{:else}
				<p>No height data available.</p>
			{/if}
		</div>
		<div class="input-group mt-4 input-group-divider grid-cols-[1fr_1fr_1fr] h-12 text-center">
			<button on:click={playStream} class="variant-filled-primary">Play</button>
			<button on:click={pauseStream} class="variant-filled-secondary">Pause</button>
			<button on:click={takeScreenShot} class="variant-filled-tertiary">Take Screenshot</button>
		</div>
	</div>
</div>

<style lang="postcss">
	figure {
		@apply flex relative flex-col;
	}
</style>

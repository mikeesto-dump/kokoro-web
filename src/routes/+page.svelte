<script lang="ts">
	import { onMount } from 'svelte';

	let worker;
	let audioSource = $state(null);
	let voices: { id: string; name: string }[] = $state([]);
	let inputText = $state('');
	let isLoading = $state(false);
	let selectedVoice = $state('af_heart');

	onMount(async () => {
		worker = new Worker(new URL('../lib/worker.js', import.meta.url), { type: 'module' });

		worker.addEventListener('message', onMessageReceived);
		worker.addEventListener('error', onErrorReceived);
	});

	function onMessageReceived(event) {
		switch (event.data.status) {
			case 'device':
				console.log('Loading model');
				break;
			case 'ready':
				// Voices is actually an object, but we want to convert it to an array
				voices = Object.entries(event.data.voices).map(([id, name]) => ({
					id,
					name
				}));
				console.log('Model loaded');
				break;
			case 'error':
				console.error('Error from worker:', event.data);
				break;
			case 'complete':
				console.log('Audio generated');
				const { audio, text } = event.data;
				audioSource = audio;
				isLoading = false;
				break;
		}
	}

	function onErrorReceived(event) {
		console.error('Error from worker:', event);
		isLoading = false;
	}

	function handleSubmit(event) {
		event.preventDefault();
		isLoading = true;

		worker.postMessage({
			type: 'generate',
			text: inputText,
			voice: selectedVoice
		});
	}

	let canSubmit = $derived(inputText.trim().length > 0 && !isLoading);
</script>

<main class="mx-auto max-w-3xl p-8">
	<h1 class="mb-2 text-center text-4xl text-gray-800">Text-to-Speech</h1>
	<p class="mb-8 text-center text-gray-600">
		Transform your text into natural-sounding speech, right in your browser!
	</p>

	<form onsubmit={handleSubmit} class="flex flex-col gap-6 rounded-lg bg-white p-8 shadow-sm">
		<div class="flex flex-col gap-2">
			<label for="text-input" class="font-medium text-gray-600">Enter your text</label>
			<textarea
				id="text-input"
				bind:value={inputText}
				placeholder="Type something you'd like to hear..."
				rows="4"
				class="w-full rounded-md border border-gray-200 p-3 text-base focus:border-blue-500 focus:ring-2 focus:ring-blue-500/20 focus:outline-none"
			></textarea>
		</div>

		<div class="flex flex-col gap-2">
			<label for="voice-select" class="font-medium text-gray-600">Choose a voice</label>
			<select
				id="voice-select"
				bind:value={selectedVoice}
				class="w-full rounded-md border border-gray-200 p-3 text-base focus:border-blue-500 focus:ring-2 focus:ring-blue-500/20 focus:outline-none"
			>
				{#each voices as voice}
					<option value={voice.id}>{voice.name.name} - {voice.name.gender}</option>
				{/each}
			</select>
		</div>

		<button
			type="submit"
			disabled={!canSubmit}
			class="rounded-md bg-blue-500 px-6 py-3 text-base font-medium text-white transition-colors hover:bg-blue-600 disabled:cursor-not-allowed disabled:bg-gray-400"
		>
			{isLoading ? 'Generating...' : 'Generate Speech'}
		</button>
	</form>

	{#if audioSource}
		<div class="mt-8">
			<audio controls src={audioSource} class="w-full"></audio>
		</div>
	{/if}
</main>

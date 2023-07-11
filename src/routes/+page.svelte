<script>
	import Loader from '$lib/Loader.svelte';
	import { stores } from '$lib/utils/hmr-stores';
	import { goto } from '$app/navigation';

	var textOnInput = 'Haz clic aquí para seleccionar un archivo';
	var logLines = [];
	var objsInLines = [];
	var fixLinesIdentified = [];

	const regularExpressions = [
		/^(\d{2}:\d{2}:\d{2},\d+)/,
		/~(\d+)/,
		/{(\d+):/,
		/:(\d+)}/,
		/}(.+?)\[/,
		/\[(.+?)\(/,
		/\((.*?)\)/,
		/\.(.*?)\]/,
		/\[.*\] ([^]*)$/,
	];

	async function identifyFixLines() {
		fixLinesIdentified = objsInLines.filter((obj) => obj[8].includes('FIX.4.4'));

		var fixMessageArray = [];
		if (fixLinesIdentified.length > 0) {
			fixLinesIdentified.forEach((element) => {
				fixMessageArray.push(element[8]);
			});
			await stores.update((state) => {
				return {
					...state,
					logWithFixMessage: fixMessageArray,
					completeLog: objsInLines,
				};
			});
			goto('/parse');
		} else {
			await stores.update((state) => {
				return { ...state, completeLog: objsInLines };
			});
			goto('/examine');
		}
	}

	async function readFile(event) {
		/* Get the file from event */
		const archivo = event.target.files[0];

		textOnInput = event.target.files[0].name;
		/* Read the file */
		const contenido = await archivo.text();
		/* Get lines in array logLines */
		logLines = contenido.split('\n');

		var actualLine = null;
		var actualobj = {};

		await logLines.forEach((element) => {
			regularExpressions.forEach((item, index) => {
				actualLine = element.match(item);
				if (actualLine) {
					actualobj[`${index}`] = actualLine[1];
				} else {
					actualobj[`${index}`] = 'none';
				}
			});
			objsInLines.push(actualobj);
			actualobj = {};
		});
		await identifyFixLines();
	}
</script>

<main>
	{#if logLines.length == 0}
		<div>
			<img src="/LogoApp.png" alt="logo" style="height: 70px; width:70px" />
		</div>
		<h1>LogViewer</h1>
		<div class="file-upload">
			<label for="file-input">{textOnInput}</label>
			<input id="file-input" type="file" on:change={readFile} accept=".txt, .log" />
		</div>
		<p>Los archivos deben ser .log o .txt</p>
	{:else}
		<Loader />
		<p>Estamos analizando tu archivo</p>
	{/if}
</main>

<style>
	:global(body) {
		font-family: 'Nunito';
		background-color: #02111a;
		color: aliceblue;
		padding: 0px;
		margin: 0px;
	}

	.file-upload {
		position: relative;
		display: flex;
		justify-content: center;
		align-items: center;
		overflow: hidden;
		width: 50%;
		height: 50%;
		border: 5px #07283c dashed;
		text-align: center;
		line-height: 200px;
		cursor: pointer;
		border-radius: 16px;
	}

	input[type='file'] {
		position: absolute;
		top: 0;
		left: 0;
		opacity: 0;
		width: 100%;
		height: 100%;
		cursor: pointer;
	}

	.file-upload label {
		font-size: 1.5rem;
		color: aliceblue;
	}

	.file-upload:hover {
		background-color: #0f3f5e;
		transition: background-color 0.5s ease;
	}

	main {
		display: flex;
		flex-direction: column;
		justify-content: center;
		align-items: center;
		width: 98%;
		height: 98vh;
	}

	@keyframes fade {
		from {
			opacity: 0;
		}
		to {
			opacity: 1;
		}
	}
</style>

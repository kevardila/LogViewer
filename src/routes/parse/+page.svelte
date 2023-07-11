<script>
	import { stores } from '$lib/utils/hmr-stores';

	import { paginate, LightPaginationNav, DarkPaginationNav } from 'svelte-paginate';
	import { onMount } from 'svelte';

	var linesWithFixMessage = [];
	const specificationsCodeByCSV = [];
	var cleanedLinesArray = [];
	var partsOfFixMessages = [];

	var keys = [];
	var values = [];
	var result = [];

	let items = [];
	let currentPage = 1;
	let pageSize = 20;
	$: paginatedItems = paginate({ items, pageSize, currentPage });

	const readSpecificationCodes = async () => {
		const response = await fetch('/specificationsCodes.csv');
		const csvData = await response.text();
		const lines = csvData.split('\n');

		lines.forEach(function (line) {
			const values = line.split(',');
			specificationsCodeByCSV.push(values);
		});
	};

	const extractPartOfFixMessages = () => {
		const regexToSeparePartsOfFixMessages = /\x01(.*?)\x01/g;
		var actualLineSeparation = [];
		cleanedLinesArray.forEach((element) => {
			actualLineSeparation = element.split(regexToSeparePartsOfFixMessages);
			actualLineSeparation.forEach((item) => {
				partsOfFixMessages.push(item);
			});
			actualLineSeparation = [];
		});
	};

	const cleanFixMessages = () => {
		var characterTrigger = false;
		const regexToLineWithOtherInformation = /<([^>]+)>/;
		var matches = null;

		linesWithFixMessage.forEach((item) => {
			characterTrigger = item.includes('{');
			if (characterTrigger) {
				matches = item.match(regexToLineWithOtherInformation);
				if (matches) {
					cleanedLinesArray.push(matches[1]);
				} else {
					console.log('No se encontraron coincidencias.');
				}
			} else {
				cleanedLinesArray.push(item);
			}
			matches = null;
			characterTrigger = false;
		});
	};

	onMount(async () => {
		linesWithFixMessage = $stores.logWithFixMessage;
		await readSpecificationCodes();
		await cleanFixMessages();
		await extractPartOfFixMessages();
		partsOfFixMessages = partsOfFixMessages.filter((item) => item.includes('='));
		keys = partsOfFixMessages.map((item) => item.split('=')[0]);
		values = partsOfFixMessages.map((item) => item.split('=')[1]);
		result = keys.map((key, index) => {
			const [_, name, description] =
				specificationsCodeByCSV.find((code) => code[0] === key) || [];
			return { key, name, description, value: values[index] };
		});
		items = result;
	});

	let searchTerm = '';
	let filtered = false;

	const searching = () => {
		if (!filtered) {
			items = items.filter((item) => {
				return item.key.includes(searchTerm);
			});
			filtered = true;
		} else {
			items = result;
			filtered = false;
			searchTerm = '';
		}
	};
</script>

<div class="container">
	<h1>Parse FIX Message</h1>
	<button>Observar log completo</button>
	<section class="table-configuration-container">
		<div class="pageSize">
			<p>Número de elementos</p>
			<select name="pageSize" id="pageSize" bind:value={pageSize} style="margin-left:10px">
				<option value="10" selected>10</option>
				<option value="30">30</option>
				<option value="50">50</option>
				<option value="100">100</option>
			</select>
		</div>
		<div class="pageSize">
			{#if !filtered}
				<p>Filtrar por Tag</p>
				<input
					class="filter-input"
					type="number"
					bind:value={searchTerm}
					min="1"
					max="9999"
					placeholder="TAG"
				/>
				<button on:click={searching} class="filter-button">Filtrar</button>
			{:else}
				<button on:click={searching}>Restaurar filtros</button>
			{/if}
		</div>
	</section>
	<table>
		<tr>
			<th>TAG</th>
			<th>FieldName</th>
			<th>Valor</th>
			<th>Description</th>
		</tr>
		{#each paginatedItems as item}
			<tr>
				<td>{item.key}</td>
				<td>{item.name}</td>
				<td>{item.value}</td>
				<td>{item.description}</td>
			</tr>
		{/each}
	</table>

	<DarkPaginationNav
		totalItems={items.length}
		{pageSize}
		{currentPage}
		limit={1}
		showStepOptions={true}
		on:setPage={(e) => (currentPage = e.detail.page)}
	/>
</div>

<style>
	.filter-input {
		margin-left: 10px;
		border-top-left-radius: 10px;
		border-bottom-left-radius: 10px;
		border: none;
		height: 40%;
	}

	.filter-button {
		background-color: #00b5e2;
		color: white;
		font-weight: 600;
		border: none;
		border-top-right-radius: 10px;
		border-bottom-right-radius: 10px;
		height: 44%;
		-webkit-box-shadow: -4px 0px 12px -1px rgba(0, 0, 0, 0.47);
		-moz-box-shadow: -4px 0px 12px -1px rgba(0, 0, 0, 0.47);
		box-shadow: -4px 0px 12px -1px rgba(0, 0, 0, 0.47);
	}

	.filter-button:hover {
		background-color: #006485;
	}

	.pageSize {
		display: flex;
		align-items: center;
		width: 100%;
	}

	.table-configuration-container {
		display: flex;
		flex-direction: column;
		align-items: center;
		width: 100%;
		margin-bottom: 10px;
	}

	.container {
		display: flex;
		flex-direction: column;
		justify-content: center;
		align-items: center;
	}
	table {
		border: 1px solid #b3adad;
		border-collapse: collapse;
		padding: 5px;
		margin-bottom: 10px;
	}
	table th {
		border: 1px solid #b3adad;
		padding: 5px;
		background: #02111a;
		color: #00b5e2;
	}
	table td {
		border: 1px solid #b3adad;
		text-align: center;
		padding: 5px;
		background: #02111a;
		color: #c3c8c6;
	}

	/* Chrome, Safari, Edge, Opera */
	input::-webkit-outer-spin-button,
	input::-webkit-inner-spin-button {
		-webkit-appearance: none;
		margin: 0;
	}
</style>

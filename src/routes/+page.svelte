<script>
	import Papa from 'papaparse';

	let expenses = $state([]);
	let income = $state([]);
	let isDragging = $state(false);
	let importAccount = $state('Chase 6302');
	let expensesCopied = $state(false);
	let incomeCopied = $state(false);
	let fileInput;

	// Account options for dropdown
	const accountOptions = ['Chase 6302', 'Chase 1639', 'Chase 8965', 'Amazon', 'Truist'];

	// Parse CSV and split into expenses and income
	function parseCSV(csvText, filename) {
		// Use the import-level account selection
		const account = importAccount;
		
		const parsed = Papa.parse(csvText, {
			header: true,
			skipEmptyLines: true
		});

		const headers = parsed.meta.fields;
		
		parsed.data.forEach(row => {
			// Chase format - only expenses (Sale transactions)
			if (headers.includes('Type')) {
				if (row.Type === 'Sale') {
					// Expense - convert negative to positive
					const amount = Math.abs(parseFloat(row.Amount));
					expenses.push({
						date: row['Transaction Date'],
						amount: `$${amount.toFixed(2)}`,
						description: row.Description,
						account: account,
						originalAmount: amount,
						originalDescription: row.Description
					});
				}
				// Ignore Payment type (card payments, not income)
			} 
			// Truist format
			else if (headers.includes('Transaction Type')) {
				if (row['Transaction Type'] === 'Debit') {
					// Expense - extract from parentheses format like ($100.00)
					let amountStr = row.Amount.replace('$', '').replace('(', '').replace(')', '');
					const amount = Math.abs(parseFloat(amountStr));
									
					// Only add if it's not a payment to another account
					if (!row.Description.includes('EPAY CHASE CREDIT CRD') && !row.Description.includes('PAYMENT AMZ_STORECRD')) {
						expenses.push({
							date: row['Transaction Date'],
							amount: `$${amount.toFixed(2)}`,
							description: row.Description,
							account: account,
							originalAmount: amount,
							originalDescription: row.Description
						});
					}
				} else if (row['Transaction Type'] === 'Credit') {
					// Income - remove $ and commas
					let amountStr = row.Amount.replace('$', '').replace(/,/g, '');
					const amount = parseFloat(amountStr);

					income.push({
						date: row['Transaction Date'],
						amount: `$${amount.toFixed(2)}`,
						description: row.Description,
						category: 'Paycheck'
					});
				}
			}
		});
	}

	function handleDrop(event) {
		event.preventDefault();
		isDragging = false;

		const files = event.dataTransfer.files;
		handleFiles(files);
	}

	function handleFileSelect(event) {
		const files = event.target.files;
		handleFiles(files);
	}

	async function handleFiles(files) {
		// Clear previous data
		expenses = [];
		income = [];

		for (const file of files) {
			if (file.type === 'text/csv' || file.name.endsWith('.csv') || file.name.endsWith('.CSV')) {
				try {
					const text = await file.text();
					parseCSV(text, file.name);
				} catch (error) {
					console.error('Error processing file:', error);
				}
			}
		}
		
		// Sort by date
		sortByDate();
	}
	
	// Parse date from MM/DD/YYYY format
	function parseDate(dateStr) {
		const [month, day, year] = dateStr.split('/');
		return new Date(year, month - 1, day);
	}
	
	// Sort expenses and income by date
	function sortByDate() {
		expenses.sort((a, b) => parseDate(a.date) - parseDate(b.date));
		income.sort((a, b) => parseDate(a.date) - parseDate(b.date));
	}

	function handleDropzoneClick(event) {
		// Don't trigger if clicking on the label (which has its own handler)
		if (event.target.tagName !== 'LABEL' && event.target.tagName !== 'P') {
			fileInput.click();
		}
	}

	function handleDropzoneKeydown(event) {
		if (event.key === 'Enter' || event.key === ' ') {
			event.preventDefault();
			fileInput.click();
		}
	}

	function handleDragOver(event) {
		event.preventDefault();
		isDragging = true;
	}

	function handleDragLeave() {
		isDragging = false;
	}

	function copyToClipboard(text) {
		navigator.clipboard.writeText(text);
	}

	function copyExpensesTable() {
		// Get all rows with accounts (no header)
		const rows = expenses.map(exp => {
			return `${exp.date}\t${exp.amount}\t${exp.description}\t${exp.account}`;
		});
		const text = rows.join('\n');
		copyToClipboard(text);
		showExpensesCopiedMessage();
	}

	function copyIncomeTable() {
		const rows = income.map(inc => {
			return `${inc.date}\t${inc.amount}\t${inc.description}\t${inc.category}`;
		});
		const text = rows.join('\n');
		copyToClipboard(text);
		showIncomeCopiedMessage();
	}
	
	function showExpensesCopiedMessage() {
		expensesCopied = true;
		setTimeout(() => {
			expensesCopied = false;
		}, 2000);
	}

	function showIncomeCopiedMessage() {
		incomeCopied = true;
		setTimeout(() => {
			incomeCopied = false;
		}, 2000);
	}

	// Update all expenses when account changes
	function updateAccount() {
		expenses = expenses.map(exp => ({
			...exp,
			account: importAccount
		}));
	}
</script>

<svelte:head>
	<title>Budget Export Cleaner</title>
</svelte:head>

<div class="container">
	<h1>Budget Export Cleaner</h1>
	<p>Upload CSV files to separate expenses and income</p>

	<div class="import-controls">
		<label for="accountSelect" class="control-label">Account:</label>
		<select bind:value={importAccount} onchange={updateAccount} id="accountSelect" class="account-select">
			{#each accountOptions as option}
				<option value={option}>{option}</option>
			{/each}
		</select>
	</div>

	<div
		class="dropzone"
		class:dragging={isDragging}
		ondrop={handleDrop}
		ondragover={handleDragOver}
		ondragleave={handleDragLeave}
		onclick={handleDropzoneClick}
		onkeydown={handleDropzoneKeydown}
		role="button"
		tabindex="0"
	>
		<input bind:this={fileInput} type="file" accept=".csv" multiple onchange={handleFileSelect} id="fileInput" />
		<label for="fileInput" class="dropzone-label">
			<p>Drop CSV files here or click to select</p>
		</label>
	</div>

	{#if expenses.length > 0 || income.length > 0}
		<div class="tables">
			{#if expenses.length > 0}
				<div class="table-section">
					<h2>Expenses</h2>
					<div class="table-container">
						<table>
							<thead>
								<tr>
									<th>Date</th>
									<th>Amount</th>
									<th>Description</th>
									<th>Account</th>
								</tr>
							</thead>
							<tbody>
								{#each expenses as exp}
									<tr>
										<td>{exp.date}</td>
										<td>{exp.amount}</td>
										<td>{exp.description}</td>
										<td>{exp.account}</td>
									</tr>
								{/each}
							</tbody>
						</table>
					</div>
				<div class="button-container">
					<button onclick={copyExpensesTable} class="copy-button">Copy Table</button>
					{#if expensesCopied}
						<span class="copied-message">Copied!</span>
					{/if}
				</div>
				</div>
			{/if}

			{#if income.length > 0}
				<div class="table-section">
					<h2>Income</h2>
					<div class="table-container">
						<table>
							<thead>
								<tr>
									<th>Date</th>
									<th>Amount</th>
									<th>Description</th>
									<th>Category</th>
								</tr>
							</thead>
							<tbody>
								{#each income as inc}
									<tr>
										<td>{inc.date}</td>
										<td>{inc.amount}</td>
										<td>{inc.description}</td>
										<td>{inc.category}</td>
									</tr>
								{/each}
							</tbody>
						</table>
					</div>
				<div class="button-container">
					<button onclick={copyIncomeTable} class="copy-button">Copy Table</button>
					{#if incomeCopied}
						<span class="copied-message">Copied!</span>
					{/if}
				</div>
				</div>
			{/if}
		</div>
	{/if}
</div>

<style>
	:global(body) {
		font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, 'Helvetica Neue', Arial, sans-serif;
	}

	.container {
		max-width: 1200px;
		margin: 0 auto;
		padding: 2rem;
	}

	h1 {
		color: #ff6600;
		margin-bottom: 0.5rem;
	}

	.import-controls {
		display: flex;
		align-items: center;
		gap: 1rem;
		margin-bottom: 1.5rem;
	}

	.control-label {
		font-weight: 500;
		color: #333;
	}

	.account-select {
		padding: 0.5rem 1rem;
		border: 1px solid #ddd;
		border-radius: 4px;
		font-size: 1rem;
		background-color: white;
		cursor: pointer;
	}

	.dropzone {
		border: 3px dashed #ccc;
		border-radius: 8px;
		padding: 3rem;
		text-align: center;
		cursor: pointer;
		transition: all 0.3s ease;
		background-color: #f9f9f9;
	}

	.dropzone.dragging {
		border-color: #ff6600;
		background-color: #fff5f0;
	}

	#fileInput {
		display: none;
	}

	.dropzone-label {
		cursor: pointer;
	}

	.tables {
		margin-top: 2rem;
		display: flex;
		flex-direction: column;
		gap: 2rem;
	}

	.table-section {
		margin-bottom: 2rem;
	}

	h2 {
		color: #333;
		margin-bottom: 1rem;
	}

	.table-container {
		overflow-x: auto;
		border: 1px solid #ddd;
		border-radius: 4px;
	}

	table {
		width: 100%;
		border-collapse: collapse;
	}

	thead {
		background-color: #f5f5f5;
	}

	th, td {
		padding: 0.75rem;
		text-align: left;
		border-bottom: 1px solid #ddd;
	}

	tbody tr:hover {
		background-color: #f9f9f9;
	}

	.copy-button {
		padding: 0.75rem 1.5rem;
		background-color: #ff6600;
		color: white;
		border: none;
		border-radius: 4px;
		cursor: pointer;
		font-size: 1rem;
		font-weight: 500;
		transition: background-color 0.2s;
	}

	.copy-button:hover {
		background-color: #e55a00;
	}

	.button-container {
		display: flex;
		align-items: center;
		gap: 1rem;
		margin-top: 1rem;
	}

	.copied-message {
		color: #28a745;
		font-weight: 500;
		font-size: 1rem;
	}
</style>

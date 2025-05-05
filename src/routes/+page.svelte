<script>
	import { onMount } from 'svelte';
	import { tick } from 'svelte';
	import { scale, fly } from 'svelte/transition';
	import words from '../lib/Wordle_WordList.json'
	
	let grid = Array.from({ length: 6 }, () => Array(5).fill(''));
	let inputs = Array.from({ length: 6 }, () => Array(5).fill(null));
	let revealedGrid = Array.from({ length: 6 }, () => Array(5).fill(false));
	let colorGrid = Array.from({ length: 6 }, () => Array(5).fill(''));

	let currentRow = 0;
	let currentCol = 0;
	let gameStatus = null;
	let showHelp = false;

	let targetWordObj = null; // { word, meaning }
	let targetWord = '';
	let targetMeaning = '';

	function pickRandomWord() {
		const randomIndex = Math.floor(Math.random() * words.length);
		targetWordObj = words[randomIndex];
		targetWord = targetWordObj.word.toUpperCase();
		targetMeaning = targetWordObj.meaning;
	}

	function resetGame() {
		grid = Array.from({ length: 6 }, () => Array(5).fill(''));
		colorGrid = Array.from({ length: 6 }, () => Array(5).fill(''));
		revealedGrid = Array.from({ length: 6 }, () => Array(5).fill(false));
		currentRow = 0;
		currentCol = 0;
		gameStatus = null;
		pickRandomWord();
		setTimeout(() => {
			focusCell(currentRow, currentCol);
		}, 50);
	}

	function focusCell(row, col) {
		const input = inputs?.[row]?.[col];
		input?.focus();
	}

	onMount(() => {
		pickRandomWord();
		focusCell(currentRow, currentCol);
	});

	function openHelp() {
		showHelp = true;
	}
	function closeHelp() {
		showHelp = false;
	}

	async function handleInput(e, rowIndex, colIndex) {
		if (rowIndex !== currentRow) return;
		const value = e.target.value.toUpperCase();
		if (value.length > 0 && /^[A-Z]$/.test(value)) {
			grid[rowIndex][colIndex] = value;
			if (currentCol < 4) {
				currentCol++;
				focusCell(currentRow, currentCol);
			}
		}
	}

	function handleKeyDown(e, rowIndex, colIndex) {
		if (rowIndex !== currentRow) return;
		if (e.key === 'Backspace') {
			if (grid[rowIndex][colIndex] === '') {
				if (currentCol > 0) {
					currentCol--;
					focusCell(currentRow, currentCol);
				}
			}
		}
		if (e.key === 'Enter') {
			submitGuess();
		}
	}

	function getColor(rowIndex, colIndex) {
		const color = colorGrid[rowIndex]?.[colIndex];
		if (color === 'green') return '#4cff00d7';
		if (color === 'yellow') return '#ffea00'; // Wordle yellow
		if (color === 'gray') return '#787c7e';
		return '#ffffffd7'; // Wordle gray
	}

	async function submitGuess() {
		const guess = grid[currentRow].join('');
		if (guess.length !== 5) {
			alert('Please fill all letters in current row');
			return;
		}

		const targetArray = targetWord.split('');
		let usedIndices = [];
		for (let i = 0; i < 5; i++) {
			if (grid[currentRow][i] === targetArray[i]) {
				colorGrid[currentRow][i] = 'green';
				usedIndices.push(i); // mark as matched
			}
		}
		// Step 2: Check for misplaced letters (Yellow)
		for (let i = 0; i < 5; i++) {
			if (colorGrid[currentRow][i] !== '') continue; // already green
			const index = targetArray.findIndex(
				(letter, idx) => letter === grid[currentRow][i] && !usedIndices.includes(idx)
			);
			if (index !== -1) {
				colorGrid[currentRow][i] = 'yellow';
				usedIndices.push(index);
			} else {
				colorGrid[currentRow][i] = 'gray';
			}
		}

		// ADD THIS LINE to trigger Svelte reactivity
		colorGrid = colorGrid.map((row) => [...row]);
		for (let i = 0; i < 5; i++) {
			await new Promise((resolve) => setTimeout(resolve, 100 * i));
			revealedGrid[currentRow][i] = true;
			revealedGrid = [...revealedGrid];
		}

		if (guess === targetWord) {
			gameStatus = 'Won';
			return;
		}
		if (currentRow < 5) {
			currentRow++;
			currentCol = 0;
			await tick(); // Wait for DOM to update (enable new row)
			focusCell(currentRow, currentCol);
		} else {
			currentRow++;
		}
		if (currentRow >= 6) {
			gameStatus = 'Lost';
			return;
		}
	}
</script>

<div class="game flex flex-col items-center gap-4 p-7 text-white">
	<div class="gameblocks mt-13 flex flex-col gap-2">
		{#each grid as row, rowIndex}
			<div class="rows flex w-full justify-evenly gap-4">
				<!-- a single row repeated 6 times -->
				{#each row as cell, colIndex}
					<div class="cell-container">
						<input
							type="text"
							maxlength="1"
							bind:value={grid[rowIndex][colIndex]}
							bind:this={inputs[rowIndex][colIndex]}
							on:input={(e) => handleInput(e, rowIndex, colIndex)}
							on:keydown={(e) => handleKeyDown(e, rowIndex, colIndex)}
							disabled={rowIndex !== currentRow || revealedGrid[rowIndex][colIndex]}
							class="h-12 w-12 rounded-lg bg-[#ffffffd7] text-center text-2xl font-bold text-black uppercase text-shadow-lg {revealedGrid[
								rowIndex
							][colIndex]
								? 'hidden'
								: ''}"
						/>
						<div
							class="cell {revealedGrid[rowIndex][colIndex]
								? 'flipped'
								: ''} flex h-12 w-12 items-center justify-center rounded-lg text-center text-2xl font-bold uppercase"
							style="background-color:{getColor(rowIndex, colIndex)};
							color:{colorGrid[rowIndex][colIndex] === 'gray' ? 'white' : 'black'};"
						>
							{grid[rowIndex][colIndex]}
						</div>
					</div>
				{/each}
			</div>
		{/each}
	</div>
	{#if gameStatus}
		<div class="mt-4 flex flex-col items-center gap-2">
			<p>You {gameStatus}</p>
			<p>The Word was "{targetWord}"</p>
			<p class="italic text-sm text-gray-300">{targetMeaning}</p>
		</div>
		<button
			on:click={resetGame}
			class="mt-4 h-10 w-full max-w-md rounded-lg bg-[#fffffffa] text-black"
		>
			Play More
		</button>
	{/if}
	<button
		on:click={openHelp}
		class="mb-4 h-10 w-full max-w-md rounded-lg bg-[#fffffffa] text-black"
	>
		How to Play
	</button>
	{#if showHelp}
		<div class="modal-backdrop" on:click={closeHelp}></div>
		<div class="modal">
			<h2 class="font-bold text-2xl">How To Play</h2>
			<ul class="flex flex-col gap-2">
				<li>📝 Guess the 5-letter word in 6 tries.</li>
				<li>✅ Each guess must be a valid 5-letter word.</li>
				<li>🎨
					After each guess, the color of the tiles will change to show how close your guess was to
					the word.
				</li>
				<li class="text-center flex gap-2 flex-col items-center md:flex-row  ">
					<div class="flex h-12 w-12 items-center justify-center rounded-lg text-center text-2xl font-bold uppercase bg-[#4cff00d7]" >G</div>
					<span style="color:#4cff00d7;">Green</span>: Correct letter, correct spot.</li>
				<li class="text-center flex gap-2 flex-col items-center md:flex-row ">
					<div class="flex h-12 w-12 items-center justify-center rounded-lg text-center text-2xl font-bold uppercase bg-[#ffea00]">Y</div>
					<span style="color:#ffea00;">Yellow</span>: Correct letter, wrong spot.</li>
				<li class="text-center flex gap-2 flex-col items-center md:flex-row ">
					<div class="flex h-12 w-12 items-center justify-center rounded-lg text-center text-2xl font-bold uppercase bg-[#787c7e]">B</div>
					<span style="color:#787c7e;">Gray</span>: Letter not in the word.</li>
			</ul>
			<button on:click={closeHelp} class="absolute right-[1rem] top-[1rem] cursor-pointer">X</button>
		</div>
	{/if}
</div>

<style>
	.cell-container {
		transform-style: preserve-3d;
		position: relative;
		width: 3rem;
		height: 3rem;
		perspective: 1000px;
	}

	.cell {
		position: absolute;
		width: 100%;
		height: 100%;
		backface-visibility: hidden;
		transform: rotateX(180deg);
		transition: transform 0.3s cubic-bezier(0.4, 0, 0.2, 1);
		will-change: transform;
	}
	input {
		position: absolute;
		width: 100%;
		height: 100%;
		backface-visibility: hidden;
		transform: rotateX(0deg);
		transition: transform 0.3s cubic-bezier(0.4, 0, 0.2, 1);
		will-change: transform;
	}
	.cell.flipped {
		transform: rotateX(0deg);
	}
	input.hidden {
		transform: rotateX(-180deg);
	}
	.modal-backdrop {
	position: fixed;
	top: 0; left: 0; right: 0; bottom: 0;
	background: rgba(0,0,0,0.5);
	z-index: 10;
}
	.modal {
		display: flex;
		flex-direction: column;
		gap: 16px;
	background: rgba(42, 41, 41, 0.8);
	backdrop-filter: blur(50px);
		
		position: relative;
	position: fixed;
	top: 50%; left: 50%;
	transform: translate(-50%, -50%);
	/* background: white; */
	color: rgb(255, 255, 255);
	padding: 2rem;
	border-radius: 1rem;
	z-index: 11;
	}
</style>

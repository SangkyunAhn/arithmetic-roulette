<script>
	import { onMount } from 'svelte';
	import { flip } from 'svelte/animate';
	import { fade, scale } from 'svelte/transition';

	let sets = $state([
		{ 
			id: crypto.randomUUID(), 
			name: '', 
			cards: Array(9).fill('?'), 
			spinValues: Array(9).fill(''),
			result: null, 
			rank: null,
			rankColor: '',
			revealed: Array(9).fill(false),
			spinning: Array(9).fill(false)
		}
	]);
	let isRunning = $state(false);
	let isFinished = $state(false);

	function addSet() {
		sets.push({ 
			id: crypto.randomUUID(), 
			name: '', 
			cards: Array(9).fill('?'), 
			spinValues: Array(9).fill(''),
			result: null, 
			rank: null,
			rankColor: '',
			revealed: Array(9).fill(false),
			spinning: Array(9).fill(false)
		});
	}

	function removeSet(id) {
		sets = sets.filter((s) => s.id !== id);
	}

	async function startGame() {
		if (isRunning) return;
		isRunning = true;
		isFinished = false;

		// Initialize sets for game start without losing input focus/data
		for (let s of sets) {
			s.cards = Array(9).fill('?');
			s.spinValues = Array(9).fill('');
			s.result = null;
			s.rank = null;
			s.rankColor = '';
			s.revealed = Array(9).fill(false);
			s.spinning = Array(9).fill(false);
		}

		const ops = ['+', '-', '×', '÷'];
		
		let currentSlot = 0;
		let spinInterval = setInterval(() => {
			if (!isRunning) return;
			for (let s of sets) {
				if (s.spinning[currentSlot]) {
					if (currentSlot % 2 === 0) {
						s.spinValues[currentSlot] = Math.floor(Math.random() * 9) + 1;
					} else {
						s.spinValues[currentSlot] = ops[Math.floor(Math.random() * ops.length)];
					}
				}
			}
		}, 50);

		for (let i = 0; i < 9; i++) {
			currentSlot = i;
			// Reveal and start spinning the current slot for all players
			for (let s of sets) {
				s.revealed[i] = true;
				s.spinning[i] = true;
			}
			
			await new Promise((r) => setTimeout(r, 3000));
			
			// Stop spinning and decide final value
			for (let s of sets) {
				s.spinning[i] = false;
				if (i % 2 === 0) {
					s.cards[i] = Math.floor(Math.random() * 9) + 1; // 1~9
				} else {
					s.cards[i] = ops[Math.floor(Math.random() * ops.length)];
				}

				// Calculate partial result
				// If index is operator (odd), evaluate up to previous number. Else evaluate up to current number.
				let partialCards = s.cards.slice(0, i % 2 === 0 ? i + 1 : i);
				if (partialCards.length > 0) {
					let expression = partialCards
						.map((c) => {
							if (c === '×') return '*';
							if (c === '÷') return '/';
							return c;
						})
						.join('');

					try {
						s.result = new Function('return ' + expression)();
					} catch (e) {
						// fallback
					}
				}
			}
		}
		
		clearInterval(spinInterval);

		// Calculate rankings based on final results
		const sortedUniqueResults = [...new Set(sets.map((s) => s.result))].sort((a, b) => b - a);
		
		// Generate random colors for each rank
		let rankColors = {};
		for (let i = 0; i < sortedUniqueResults.length; i++) {
			// Random bright color
			rankColors[i + 1] = `hsl(${Math.floor(Math.random() * 360)}, 80%, 55%)`;
		}

		for (let s of sets) {
			s.rank = sortedUniqueResults.indexOf(s.result) + 1;
			s.rankColor = rankColors[s.rank];
		}

		isFinished = true;
		isRunning = false;
	}

	function formatResult(val) {
		if (val === null) return '';
		if (!isFinite(val)) return '∞';
		if (Number.isInteger(val)) return val;
		return parseFloat(val.toFixed(2));
	}

	function getOrdinal(n) {
		if (n % 10 === 1 && n % 100 !== 11) return n + "st";
		if (n % 10 === 2 && n % 100 !== 12) return n + "nd";
		if (n % 10 === 3 && n % 100 !== 13) return n + "rd";
		return n + "th";
	}
</script>

<svelte:head>
	<link rel="preconnect" href="https://fonts.googleapis.com" />
	<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin />
	<link
		href="https://fonts.googleapis.com/css2?family=Outfit:wght@300;400;600;800&family=Space+Mono:wght@400;700&display=swap"
		rel="stylesheet"
	/>
</svelte:head>

<main class="app-container">
	<header>
		<h1>Arithmetic <span>Roulette</span></h1>
		<p class="subtitle">Will you draw the lucky numbers?</p>
	</header>

	<div class="controls">
		<button class="btn btn-add" onclick={addSet} disabled={isRunning}>
			<svg width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><line x1="12" y1="5" x2="12" y2="19"></line><line x1="5" y1="12" x2="19" y2="12"></line></svg>
			Add Player
		</button>
		<button class="btn btn-start {isRunning ? 'running' : ''}" onclick={startGame} disabled={isRunning || sets.length === 0}>
			{#if isRunning}
				<div class="spinner"></div> Running...
			{:else}
				<svg width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><polygon points="5 3 19 12 5 21 5 3"></polygon></svg>
				Start Game
			{/if}
		</button>
	</div>

	<div class="sets-container">
		{#each sets as set (set.id)}
			<div class="player-row" transition:fade|local>
				<div class="player-info">
					{#if set.rank && isFinished}
						<div class="rank-badge" style="background: {set.rankColor}; box-shadow: 0 4px 15px {set.rankColor}80;" in:scale>
							{getOrdinal(set.rank)}
						</div>
					{/if}
					<input
						type="text"
						placeholder="Player Name"
						bind:value={set.name}
						disabled={isRunning}
						class="name-input"
					/>
					{#if sets.length > 1 && !isRunning}
						<button class="btn-remove" onclick={() => removeSet(set.id)} aria-label="Remove player">
							&times;
						</button>
					{/if}
				</div>

				<div class="cards-wrapper">
					{#each set.cards as card, i}
						<div class="card {i % 2 === 0 ? 'number' : 'operator'} {set.revealed[i] ? 'revealed' : ''}">
							<div class="card-inner">
								<div class="card-front">?</div>
								<div class="card-back {set.spinning[i] ? 'spinning-bg' : ''}">
									<span class={set.spinning[i] ? 'slot-spin-text' : ''}>
										{set.spinning[i] ? set.spinValues[i] : card}
									</span>
								</div>
							</div>
						</div>
					{/each}
					
					<div class="result-box {set.result !== null ? 'show' : ''}">
						<div class="equals">=</div>
						<div class="result-value">
							{#if set.result !== null}
								{formatResult(set.result)}
							{:else}
								?
							{/if}
						</div>
					</div>
				</div>
			</div>
		{/each}
	</div>

	<footer class="copyright">
		By Sangkyun Ahn
	</footer>
</main>

<style>
	:global(body) {
		margin: 0;
		padding: 0;
		background: #0f111a;
		color: #e2e8f0;
		font-family: 'Outfit', sans-serif;
		min-height: 100vh;
		display: flex;
		flex-direction: column;
		overflow-x: hidden;
	}

	/* Animated background blobs */
	:global(body)::before,
	:global(body)::after {
		content: '';
		position: fixed;
		width: 600px;
		height: 600px;
		border-radius: 50%;
		filter: blur(120px);
		z-index: -1;
		animation: float 20s infinite ease-in-out;
		opacity: 0.5;
	}

	:global(body)::before {
		background: rgba(99, 102, 241, 0.4);
		top: -200px;
		left: -200px;
	}

	:global(body)::after {
		background: rgba(236, 72, 153, 0.3);
		bottom: -200px;
		right: -200px;
		animation-delay: -10s;
	}

	@keyframes float {
		0%, 100% { transform: translate(0, 0); }
		33% { transform: translate(30px, -50px); }
		66% { transform: translate(-20px, 40px); }
	}

	.app-container {
		max-width: 1200px;
		margin: 0 auto;
		padding: 3rem 1.5rem;
		width: 100%;
		box-sizing: border-box;
	}

	header {
		text-align: center;
		margin-bottom: 3rem;
	}

	h1 {
		font-size: 3.5rem;
		font-weight: 800;
		margin: 0 0 0.5rem 0;
		letter-spacing: -1px;
	}

	h1 span {
		background: linear-gradient(135deg, #6366f1 0%, #ec4899 100%);
		-webkit-background-clip: text;
		-webkit-text-fill-color: transparent;
	}

	.subtitle {
		font-size: 1.2rem;
		color: #94a3b8;
		margin: 0;
		font-weight: 300;
	}

	.controls {
		display: flex;
		justify-content: center;
		gap: 1rem;
		margin-bottom: 3rem;
	}

	.btn {
		display: inline-flex;
		align-items: center;
		gap: 0.5rem;
		padding: 0.8rem 1.5rem;
		border-radius: 999px;
		font-family: 'Outfit', sans-serif;
		font-size: 1.1rem;
		font-weight: 600;
		cursor: pointer;
		transition: all 0.2s cubic-bezier(0.4, 0, 0.2, 1);
		border: none;
		outline: none;
	}

	.btn:disabled {
		opacity: 0.5;
		cursor: not-allowed;
		transform: none !important;
		box-shadow: none !important;
	}

	.btn-add {
		background: rgba(30, 41, 59, 0.8);
		color: white;
		border: 1px solid rgba(255, 255, 255, 0.1);
		backdrop-filter: blur(10px);
	}

	.btn-add:hover:not(:disabled) {
		background: rgba(40, 53, 75, 0.9);
		transform: translateY(-2px);
		box-shadow: 0 10px 20px rgba(0, 0, 0, 0.2);
	}

	.btn-start {
		background: linear-gradient(135deg, #6366f1 0%, #8b5cf6 100%);
		color: white;
		box-shadow: 0 4px 15px rgba(99, 102, 241, 0.4);
	}

	.btn-start:hover:not(:disabled) {
		transform: translateY(-2px);
		box-shadow: 0 10px 25px rgba(99, 102, 241, 0.6);
	}

	.btn-start.running {
		background: #334155;
		box-shadow: none;
	}

	.spinner {
		width: 18px;
		height: 18px;
		border: 2px solid rgba(255,255,255,0.3);
		border-radius: 50%;
		border-top-color: white;
		animation: spin 1s ease-in-out infinite;
	}

	@keyframes spin {
		to { transform: rotate(360deg); }
	}

	.sets-container {
		display: flex;
		flex-direction: column;
		gap: 1.5rem;
	}

	.player-row {
		background: rgba(15, 23, 42, 0.6);
		border: 1px solid rgba(255, 255, 255, 0.05);
		border-radius: 20px;
		padding: 1.5rem;
		display: flex;
		align-items: center;
		gap: 2rem;
		backdrop-filter: blur(12px);
		box-shadow: 0 8px 32px rgba(0, 0, 0, 0.2);
		transition: transform 0.3s ease, box-shadow 0.3s ease;
	}
	
	.player-row:hover {
		transform: translateY(-2px);
		box-shadow: 0 12px 40px rgba(0, 0, 0, 0.3);
		border-color: rgba(255, 255, 255, 0.1);
	}

	.player-info {
		display: flex;
		align-items: center;
		gap: 1rem;
		flex: 0 0 250px;
		max-width: 250px;
		position: relative;
	}

	.rank-badge {
		position: absolute;
		top: -30px;
		left: 0;
		padding: 0.35rem 0.85rem;
		border-radius: 999px;
		font-weight: 800;
		font-size: 0.9rem;
		text-transform: uppercase;
		letter-spacing: 1px;
		color: white;
		z-index: 10;
		text-shadow: 0 1px 2px rgba(0,0,0,0.5);
	}

	.name-input {
		background: rgba(0, 0, 0, 0.2);
		border: 1px solid rgba(255, 255, 255, 0.1);
		color: white;
		padding: 0.8rem 1rem;
		border-radius: 12px;
		font-family: 'Outfit', sans-serif;
		font-size: 1.1rem;
		font-weight: 600;
		width: 100%;
		outline: none;
		transition: all 0.2s;
	}

	.name-input:focus {
		border-color: #6366f1;
		background: rgba(0, 0, 0, 0.4);
		box-shadow: 0 0 0 2px rgba(99, 102, 241, 0.2);
	}

	.name-input:disabled {
		opacity: 0.7;
	}

	.btn-remove {
		background: none;
		border: none;
		color: #ef4444;
		font-size: 1.5rem;
		cursor: pointer;
		padding: 0 0.5rem;
		opacity: 0.5;
		transition: all 0.2s;
	}

	.btn-remove:hover {
		opacity: 1;
		transform: scale(1.1);
	}

	.cards-wrapper {
		display: flex;
		align-items: center;
		gap: 0.75rem;
		flex: 1;
		justify-content: flex-start;
		flex-wrap: nowrap;
	}

	.card {
		width: 60px;
		height: 80px;
		perspective: 1000px;
		flex-shrink: 0;
	}

	.card.operator {
		width: 40px;
		height: 40px;
		border-radius: 50%;
	}

	.card-inner {
		position: relative;
		width: 100%;
		height: 100%;
		text-align: center;
		transition: transform 0.6s cubic-bezier(0.4, 0, 0.2, 1);
		transform-style: preserve-3d;
	}

	.card.revealed .card-inner {
		transform: rotateY(180deg);
	}

	.card-front, .card-back {
		position: absolute;
		width: 100%;
		height: 100%;
		-webkit-backface-visibility: hidden;
		backface-visibility: hidden;
		display: flex;
		align-items: center;
		justify-content: center;
		font-family: 'Space Mono', monospace;
		font-size: 1.8rem;
		font-weight: 700;
		border-radius: 12px;
		box-shadow: 0 4px 10px rgba(0,0,0,0.2);
	}

	.card.operator .card-front,
	.card.operator .card-back {
		border-radius: 50%;
		font-size: 1.4rem;
	}

	.card-front {
		background: linear-gradient(135deg, #1e293b 0%, #0f172a 100%);
		border: 1px solid rgba(255, 255, 255, 0.1);
		color: #475569;
	}

	.card-back {
		transform: rotateY(180deg);
	}

	.card.number .card-back {
		background: linear-gradient(135deg, #3b82f6 0%, #2563eb 100%);
		color: white;
		border: 1px solid rgba(96, 165, 250, 0.3);
	}

	.card.operator .card-back {
		background: linear-gradient(135deg, #ec4899 0%, #be185d 100%);
		color: white;
		border: 1px solid rgba(244, 114, 182, 0.3);
	}

	.slot-spin-text {
		display: inline-block;
		animation: slotSpin 0.08s infinite alternate;
		filter: blur(0.5px);
	}

	@keyframes slotSpin {
		0% { transform: translateY(-4px); }
		100% { transform: translateY(4px); }
	}

	.spinning-bg {
		box-shadow: inset 0 0 15px rgba(255, 255, 255, 0.6) !important;
		filter: brightness(1.2);
	}

	.result-box {
		display: flex;
		align-items: center;
		gap: 0.75rem;
		margin-left: auto;
		opacity: 0.3;
		transition: all 0.5s ease;
		transform: translateX(-10px);
		width: 140px;
		justify-content: flex-end;
	}

	.result-box.show {
		opacity: 1;
		transform: translateX(0);
	}

	.equals {
		font-size: 1.5rem;
		font-weight: 700;
		color: #94a3b8;
	}

	.result-value {
		background: rgba(16, 185, 129, 0.1);
		border: 1px solid rgba(16, 185, 129, 0.3);
		color: #34d399;
		padding: 0.5rem 1rem;
		border-radius: 12px;
		font-family: 'Space Mono', monospace;
		font-size: 1.8rem;
		font-weight: 700;
		min-width: 80px;
		text-align: center;
		box-shadow: 0 0 20px rgba(16, 185, 129, 0.1);
		transition: all 0.3s ease;
	}

	@media (max-width: 1024px) {
		.player-row {
			flex-direction: column;
			align-items: stretch;
		}
		
		.player-info {
			margin-bottom: 1rem;
			flex: auto;
			max-width: none;
		}
		
		.cards-wrapper {
			justify-content: center;
			flex-wrap: wrap;
		}

		.result-box {
			margin-left: 0;
			width: auto;
			justify-content: center;
			margin-top: 1rem;
		}
	}
	
	@media (max-width: 600px) {
		.card {
			width: 45px;
			height: 65px;
		}
		
		.card-front, .card-back {
			font-size: 1.4rem;
		}
		
		.result-value {
			font-size: 1.4rem;
			padding: 0.4rem 0.8rem;
		}
		
		h1 {
			font-size: 2.5rem;
		}
	}
	.copyright {
		text-align: center;
		margin-top: 4rem;
		padding-top: 2rem;
		border-top: 1px solid rgba(255, 255, 255, 0.1);
		color: #64748b;
		font-size: 0.9rem;
		font-weight: 300;
		letter-spacing: 1px;
	}
</style>

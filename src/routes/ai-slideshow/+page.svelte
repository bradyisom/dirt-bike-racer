<script lang="ts">
	let currentSlide = $state(0);
	let visibleBullets = $state(0);

	const slides = [
		{
			title: "What is AI?",
			subtitle: "It's not what you think...",
			content: [
				"Software that learns patterns instead of following rules",
				"Like autocomplete on your phone, but way more powerful",
				"It's basically really good at guessing what comes next"
			],
			icon: "🤖"
		},
		{
			title: "Movie AI vs Real AI",
			subtitle: "Spoiler: Robots aren't taking over",
			content: [
				"Movies: Evil robots with feelings plotting revenge",
				"Reality: Fancy math that predicts text",
				"AI doesn't want anything — it's not alive",
				"Skynet isn't happening (sorry to disappoint)"
			],
			icon: "🎬"
		},
		{
			title: "How Does ChatGPT Work?",
			subtitle: "The world's best autocomplete",
			content: [
				"It's an LLM (Large Language Model) — trained on tons of text",
				"Learned patterns in how humans write",
				"Predicts the next word, over and over",
				"That's it. That's the secret."
			],
			icon: "🧠"
		},
		{
			title: "What AI is Good At",
			subtitle: "Actually pretty cool stuff",
			content: [
				"Writing code and explaining how it works",
				"Answering questions about almost anything",
				"Translating languages instantly",
				"Helping you brainstorm ideas"
			],
			icon: "💪"
		},
		{
			title: "What AI is Bad At",
			subtitle: "It's not perfect",
			content: [
				"Sometimes just makes stuff up (called \"hallucinating\")",
				"Can't actually think or understand anything",
				"Doesn't know what happened after it was trained",
				"No common sense — just pattern matching"
			],
			icon: "🤷"
		},
		{
			title: "AI is a Tool",
			subtitle: "Like a super-powered calculator for words",
			content: [
				"You're still the one in charge",
				"Best results = You + AI working together",
				"Don't trust everything it says — verify!",
				"The people who learn to use AI will have superpowers"
			],
			icon: "🛠️"
		}
	];

	function advance() {
		const totalBullets = slides[currentSlide].content.length;
		if (visibleBullets < totalBullets) {
			visibleBullets++;
		} else if (currentSlide < slides.length - 1) {
			currentSlide++;
			visibleBullets = 0;
		}
	}

	function goBack() {
		if (visibleBullets > 0) {
			visibleBullets--;
		} else if (currentSlide > 0) {
			currentSlide--;
			visibleBullets = slides[currentSlide].content.length;
		}
	}

	function goToSlide(index: number) {
		currentSlide = index;
		visibleBullets = 0;
	}

	function handleKeydown(event: KeyboardEvent) {
		if (event.key === 'ArrowRight' || event.key === ' ' || event.key === 'Enter') {
			event.preventDefault();
			advance();
		}
		if (event.key === 'ArrowLeft') {
			goBack();
		}
	}
</script>

<svelte:window onkeydown={handleKeydown} />

<div class="slideshow">
	<div class="slide">
		<div class="slide-icon">{slides[currentSlide].icon}</div>
		<h1>{slides[currentSlide].title}</h1>
		<h2>{slides[currentSlide].subtitle}</h2>
		<ul>
			{#each slides[currentSlide].content as point, index}
				{#if index < visibleBullets}
					<li class="visible">{point}</li>
				{/if}
			{/each}
		</ul>
	</div>

	<div class="footer">
		<div class="dots">
			{#each slides as _, index}
				<button
					class="dot"
					class:active={currentSlide === index}
					onclick={() => goToSlide(index)}
					aria-label="Go to slide {index + 1}"
				></button>
			{/each}
		</div>

		{#if currentSlide === slides.length - 1 && visibleBullets === slides[currentSlide].content.length}
			<a href="/" class="home-link">Back to Home</a>
		{:else}
			<p class="hint">Press Space, Enter, or → to reveal bullet points</p>
		{/if}
	</div>
</div>

<style>
	.slideshow {
		height: 100vh;
		display: flex;
		flex-direction: column;
		align-items: center;
		justify-content: center;
		padding: 1rem;
		box-sizing: border-box;
		overflow: hidden;
		background: linear-gradient(135deg, #1a1a2e 0%, #16213e 50%, #0f3460 100%);
		color: white;
		font-family: system-ui, -apple-system, sans-serif;
	}

	.slide {
		max-width: 800px;
		text-align: center;
		animation: fadeIn 0.3s ease-in-out;
		margin-bottom: 6rem;
	}

	@keyframes fadeIn {
		from { opacity: 0; transform: translateY(10px); }
		to { opacity: 1; transform: translateY(0); }
	}

	.slide-icon {
		font-size: 3.5rem;
		margin-bottom: 0.75rem;
	}

	h1 {
		font-size: 2.25rem;
		margin: 0 0 0.4rem 0;
		color: #e94560;
	}

	h2 {
		font-size: 1.4rem;
		margin: 0 0 1.25rem 0;
		color: #a0a0a0;
		font-weight: 400;
	}

	ul {
		list-style: none;
		padding: 0;
		margin: 0;
		text-align: left;
	}

	li {
		font-size: 1.2rem;
		padding: 0.5rem 0;
		padding-left: 1.5rem;
		position: relative;
		line-height: 1.4;
	}

	li::before {
		content: "→";
		position: absolute;
		left: 0;
		color: #e94560;
	}

	li.visible {
		animation: bulletIn 0.3s ease-out;
	}

	@keyframes bulletIn {
		from { opacity: 0; transform: translateX(-20px); }
		to { opacity: 1; transform: translateX(0); }
	}

	button {
		background: #e94560;
		color: white;
		border: none;
		padding: 0.75rem 1.5rem;
		font-size: 1rem;
		border-radius: 8px;
		cursor: pointer;
		transition: all 0.2s;
	}

	button:hover:not(:disabled) {
		background: #ff6b6b;
		transform: scale(1.05);
	}

	button:disabled {
		background: #444;
		cursor: not-allowed;
	}

	.footer {
		position: absolute;
		bottom: 2rem;
		display: flex;
		flex-direction: column;
		align-items: center;
		gap: 0.75rem;
	}

	.dots {
		display: flex;
		gap: 0.5rem;
	}

	.dot {
		width: 12px;
		height: 12px;
		border-radius: 50%;
		background: #444;
		padding: 0;
		transition: all 0.2s;
	}

	.dot:hover {
		background: #666;
	}

	.dot.active {
		background: #e94560;
		transform: scale(1.2);
	}

	.hint {
		margin-top: 1rem;
		color: #666;
		font-size: 0.875rem;
	}

	.home-link {
		margin-top: 1rem;
		color: #e94560;
		font-size: 1rem;
		text-decoration: none;
		padding: 0.5rem 1rem;
		border: 2px solid #e94560;
		border-radius: 8px;
		transition: all 0.2s;
	}

	.home-link:hover {
		background: #e94560;
		color: white;
	}
</style>

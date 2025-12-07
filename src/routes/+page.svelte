<script>
	import { onMount, onDestroy } from 'svelte';

	let mounted = false;
	let downloadUrl = 'https://github.com/PxPerfectMike/signals-site/releases/latest/download/signals-demo-win64.zip';

	// Audio state
	let audio;
	let audioContext;
	let analyser;
	let dataArray;
	let isMuted = true;
	let isPlaying = false;
	let audioInitialized = false;

	// Reactive values for visuals
	let bass = 0;
	let mids = 0;
	let highs = 0;
	let energy = 0;

	// Particles
	let particles = [];
	const MAX_PARTICLES = 50;

	// Animation frame
	let animationFrame;

	function initAudio() {
		if (audioInitialized) return;

		audio = new Audio('/song14.wav');
		audio.loop = true;
		audio.volume = 0.7;

		audioContext = new (window.AudioContext || window.webkitAudioContext)();
		analyser = audioContext.createAnalyser();
		analyser.fftSize = 256;

		const source = audioContext.createMediaElementSource(audio);
		source.connect(analyser);
		analyser.connect(audioContext.destination);

		dataArray = new Uint8Array(analyser.frequencyBinCount);
		audioInitialized = true;
	}

	function toggleMute() {
		if (!audioInitialized) {
			initAudio();
		}

		if (audioContext.state === 'suspended') {
			audioContext.resume();
		}

		isMuted = !isMuted;

		if (!isMuted) {
			audio.play();
			isPlaying = true;
		} else {
			audio.pause();
			isPlaying = false;
		}
	}

	function updateAudioData() {
		if (!analyser || !isPlaying) {
			bass = 0;
			mids = 0;
			highs = 0;
			energy = 0;
			return;
		}

		analyser.getByteFrequencyData(dataArray);

		// Calculate frequency bands
		const bufferLength = dataArray.length;
		let bassSum = 0, midsSum = 0, highsSum = 0;

		// Bass: 0-10% of frequencies
		for (let i = 0; i < bufferLength * 0.1; i++) {
			bassSum += dataArray[i];
		}
		bass = bassSum / (bufferLength * 0.1) / 255;

		// Mids: 10-50% of frequencies
		for (let i = Math.floor(bufferLength * 0.1); i < bufferLength * 0.5; i++) {
			midsSum += dataArray[i];
		}
		mids = midsSum / (bufferLength * 0.4) / 255;

		// Highs: 50-100% of frequencies
		for (let i = Math.floor(bufferLength * 0.5); i < bufferLength; i++) {
			highsSum += dataArray[i];
		}
		highs = highsSum / (bufferLength * 0.5) / 255;

		// Overall energy
		energy = (bass * 0.5 + mids * 0.3 + highs * 0.2);
	}

	function spawnParticle() {
		if (particles.length >= MAX_PARTICLES) return;
		if (!isPlaying || energy < 0.3) return;

		particles = [...particles, {
			id: Math.random(),
			x: Math.random() * 100,
			y: 100 + Math.random() * 20,
			vx: (Math.random() - 0.5) * 2,
			vy: -2 - Math.random() * 3,
			size: 4 + bass * 10,
			hue: Math.random() * 60 + 160, // Cyan to purple range
			life: 1
		}];
	}

	function updateParticles(dt) {
		particles = particles
			.map(p => ({
				...p,
				x: p.x + p.vx * dt * 30,
				y: p.y + p.vy * dt * 30,
				vy: p.vy + 0.5 * dt * 30, // Gravity
				life: p.life - dt * 0.5
			}))
			.filter(p => p.life > 0 && p.y < 120);
	}

	let lastTime = 0;
	function animate(time) {
		const dt = Math.min((time - lastTime) / 1000, 0.1);
		lastTime = time;

		updateAudioData();
		updateParticles(dt);

		// Spawn particles on beats
		if (Math.random() < bass * 0.5) {
			spawnParticle();
		}

		animationFrame = requestAnimationFrame(animate);
	}

	onMount(() => {
		mounted = true;
		lastTime = performance.now();
		animationFrame = requestAnimationFrame(animate);
	});

	onDestroy(() => {
		if (animationFrame) {
			cancelAnimationFrame(animationFrame);
		}
		if (audio) {
			audio.pause();
			audio.src = '';
		}
		if (audioContext) {
			audioContext.close();
		}
	});
</script>

<svelte:head>
	<title>SIGNALS - A Music-Driven Survival Game</title>
	<meta name="description" content="SIGNALS is a music-reactive survival game where you dodge enemies and survive the beat. Download the free demo now!" />
</svelte:head>

<!-- Music-reactive background layer -->
<div
	class="reactive-bg"
	style="--bass: {bass}; --mids: {mids}; --highs: {highs}; --energy: {energy};"
></div>

<!-- Particle layer -->
<div class="particles">
	{#each particles as particle (particle.id)}
		<div
			class="particle"
			style="
				left: {particle.x}%;
				top: {particle.y}%;
				width: {particle.size}px;
				height: {particle.size}px;
				background: hsl({particle.hue}, 100%, 60%);
				opacity: {particle.life};
				box-shadow: 0 0 {particle.size * 2}px hsl({particle.hue}, 100%, 50%);
			"
		></div>
	{/each}
</div>

<!-- Floating geometric shapes -->
<div class="shapes" class:mounted class:playing={isPlaying} style="--bass: {bass}; --energy: {energy};">
	<div class="shape circle"></div>
	<div class="shape square"></div>
	<div class="shape triangle"></div>
	<div class="shape circle small"></div>
	<div class="shape square small"></div>
	<div class="shape triangle small"></div>
</div>

<!-- Beat rings -->
<div class="beat-rings" style="--bass: {bass}; --energy: {energy};">
	<div class="beat-ring"></div>
	<div class="beat-ring delay-1"></div>
	<div class="beat-ring delay-2"></div>
</div>

<!-- MUTE BUTTON - Always visible -->
<button class="mute-btn" class:muted={isMuted} on:click={toggleMute}>
	{#if isMuted}
		<svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
			<polygon points="11 5 6 9 2 9 2 15 6 15 11 19 11 5"></polygon>
			<line x1="23" y1="9" x2="17" y2="15"></line>
			<line x1="17" y1="9" x2="23" y2="15"></line>
		</svg>
		<span>Play Music</span>
	{:else}
		<svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
			<polygon points="11 5 6 9 2 9 2 15 6 15 11 19 11 5"></polygon>
			<path d="M19.07 4.93a10 10 0 0 1 0 14.14M15.54 8.46a5 5 0 0 1 0 7.07"></path>
		</svg>
		<span>Mute</span>
	{/if}
</button>

<!-- Visualizer bars at bottom -->
{#if isPlaying}
<div class="visualizer">
	{#each Array(32) as _, i}
		<div
			class="viz-bar"
			style="height: {Math.max(4, (dataArray ? dataArray[i * 4] / 255 : 0) * 100)}%"
		></div>
	{/each}
</div>
{/if}

<!-- Hero Section -->
<section class="hero">
	<div class="hero-content">
		<h1 class="title" class:mounted class:playing={isPlaying} style="--bass: {bass}; --energy: {energy};">
			<span class="letter">S</span><span class="letter">I</span><span class="letter">G</span><span class="letter">N</span><span class="letter">A</span><span class="letter">L</span><span class="letter">S</span>
		</h1>
		<p class="tagline" class:mounted>A Music-Driven Survival Experience</p>

		<div class="hero-description" class:mounted>
			<p>
				Dive into a neon-soaked world where music is your guide and survival is your goal.
				Every beat pulses through the environment, every enemy moves to the rhythm.
				<strong>Survive the song to win.</strong>
			</p>
		</div>

		<div class="cta-buttons" class:mounted>
			<a href={downloadUrl} class="btn btn-primary" class:pulse={isPlaying && bass > 0.5} download>
				<svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
					<path d="M21 15v4a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2v-4"></path>
					<polyline points="7 10 12 15 17 10"></polyline>
					<line x1="12" y1="15" x2="12" y2="3"></line>
				</svg>
				Download Demo
			</a>
			<span class="download-info">Windows 64-bit | ~150MB</span>
		</div>
	</div>

	<!-- Animated rings -->
	<div class="rings" style="--bass: {bass};">
		<div class="ring"></div>
		<div class="ring"></div>
		<div class="ring"></div>
	</div>
</section>

<!-- Features Section -->
<section class="features">
	<h2 class="section-title">Features</h2>

	<div class="features-grid">
		<div class="feature-card" class:glow={isPlaying && bass > 0.4}>
			<div class="feature-icon">
				<svg xmlns="http://www.w3.org/2000/svg" width="48" height="48" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.5">
					<path d="M9 18V5l12-2v13"></path>
					<circle cx="6" cy="18" r="3"></circle>
					<circle cx="18" cy="16" r="3"></circle>
				</svg>
			</div>
			<h3>Music-Reactive World</h3>
			<p>Every visual element pulses with the beat. The hex grid, enemies, and effects all react to the music in real-time.</p>
		</div>

		<div class="feature-card" class:glow={isPlaying && mids > 0.4}>
			<div class="feature-icon magenta">
				<svg xmlns="http://www.w3.org/2000/svg" width="48" height="48" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.5">
					<polygon points="12 2 15.09 8.26 22 9.27 17 14.14 18.18 21.02 12 17.77 5.82 21.02 7 14.14 2 9.27 8.91 8.26 12 2"></polygon>
				</svg>
			</div>
			<h3>Survival Gameplay</h3>
			<p>Auto-aim lets you focus on movement. Dodge waves of music-synced enemies and survive until the song ends.</p>
		</div>

		<div class="feature-card" class:glow={isPlaying && highs > 0.4}>
			<div class="feature-icon purple">
				<svg xmlns="http://www.w3.org/2000/svg" width="48" height="48" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.5">
					<path d="M18 3a3 3 0 0 0-3 3v12a3 3 0 0 0 3 3 3 3 0 0 0 3-3 3 3 0 0 0-3-3H6a3 3 0 0 0-3 3 3 3 0 0 0 3 3 3 3 0 0 0 3-3V6a3 3 0 0 0-3-3 3 3 0 0 0-3 3 3 3 0 0 0 3 3h12a3 3 0 0 0 3-3 3 3 0 0 0-3-3z"></path>
				</svg>
			</div>
			<h3>Psychedelic Visuals</h3>
			<p>Shader-powered backgrounds, glowing particle effects, and a vibrant neon aesthetic create an immersive experience.</p>
		</div>
	</div>
</section>

<!-- How to Play Section -->
<section class="how-to-play">
	<h2 class="section-title">How to Play</h2>

	<div class="steps">
		<div class="step">
			<div class="step-number">1</div>
			<div class="step-content">
				<h3>Download & Extract</h3>
				<p>Download the demo zip file and extract it anywhere on your computer.</p>
			</div>
		</div>

		<div class="step">
			<div class="step-number">2</div>
			<div class="step-content">
				<h3>Run the Game</h3>
				<p>Double-click <code>signals-demo.exe</code> to launch. No installation required!</p>
			</div>
		</div>

		<div class="step">
			<div class="step-number">3</div>
			<div class="step-content">
				<h3>Survive</h3>
				<p>Use <strong>WASD</strong> or <strong>Arrow Keys</strong> to move. Auto-aim handles shooting. Survive the song!</p>
			</div>
		</div>
	</div>
</section>

<!-- Footer -->
<footer>
	<div class="footer-content">
		<p class="made-with">Made with LÖVE</p>
		<p class="copyright">SIGNALS Demo &copy; 2024</p>
	</div>
</footer>

<style>
	/* Music-reactive background */
	.reactive-bg {
		position: fixed;
		top: 0;
		left: 0;
		width: 100%;
		height: 100%;
		pointer-events: none;
		z-index: -3;
		background: radial-gradient(
			ellipse at 50% 50%,
			rgba(0, 255, 204, calc(0.1 + var(--bass, 0) * 0.3)) 0%,
			rgba(255, 0, 170, calc(0.05 + var(--mids, 0) * 0.2)) 30%,
			rgba(136, 68, 255, calc(0.05 + var(--highs, 0) * 0.15)) 60%,
			transparent 100%
		);
		transition: background 0.1s ease;
	}

	/* Particles */
	.particles {
		position: fixed;
		top: 0;
		left: 0;
		width: 100%;
		height: 100%;
		pointer-events: none;
		z-index: 10;
		overflow: hidden;
	}

	.particle {
		position: absolute;
		border-radius: 50%;
		transform: translate(-50%, -50%);
	}

	/* Beat rings */
	.beat-rings {
		position: fixed;
		top: 50%;
		left: 50%;
		transform: translate(-50%, -50%);
		pointer-events: none;
		z-index: -1;
	}

	.beat-ring {
		position: absolute;
		border: 2px solid var(--primary-cyan);
		border-radius: 50%;
		opacity: calc(0.1 + var(--bass, 0) * 0.4);
		width: calc(300px + var(--bass, 0) * 200px);
		height: calc(300px + var(--bass, 0) * 200px);
		top: 50%;
		left: 50%;
		transform: translate(-50%, -50%);
		animation: beatPulse 2s ease-out infinite;
	}

	.beat-ring.delay-1 {
		animation-delay: 0.3s;
		border-color: var(--primary-magenta);
	}

	.beat-ring.delay-2 {
		animation-delay: 0.6s;
		border-color: var(--primary-purple);
	}

	@keyframes beatPulse {
		0% {
			transform: translate(-50%, -50%) scale(0.5);
			opacity: 0.6;
		}
		100% {
			transform: translate(-50%, -50%) scale(2);
			opacity: 0;
		}
	}

	/* Mute button */
	.mute-btn {
		position: fixed;
		top: 20px;
		right: 20px;
		z-index: 1000;
		display: flex;
		align-items: center;
		gap: 8px;
		padding: 12px 20px;
		background: rgba(0, 255, 204, 0.15);
		border: 2px solid var(--primary-cyan);
		border-radius: 50px;
		color: var(--primary-cyan);
		font-family: var(--font-display);
		font-size: 0.9rem;
		font-weight: 600;
		letter-spacing: 0.1em;
		cursor: pointer;
		transition: all 0.3s ease;
		backdrop-filter: blur(10px);
	}

	.mute-btn:hover {
		background: rgba(0, 255, 204, 0.3);
		box-shadow: 0 0 30px rgba(0, 255, 204, 0.5);
		transform: scale(1.05);
	}

	.mute-btn.muted {
		animation: pulseGlow 2s ease-in-out infinite;
	}

	@keyframes pulseGlow {
		0%, 100% {
			box-shadow: 0 0 20px rgba(0, 255, 204, 0.3);
		}
		50% {
			box-shadow: 0 0 40px rgba(0, 255, 204, 0.6), 0 0 60px rgba(255, 0, 170, 0.3);
		}
	}

	/* Visualizer */
	.visualizer {
		position: fixed;
		bottom: 0;
		left: 0;
		width: 100%;
		height: 80px;
		display: flex;
		align-items: flex-end;
		justify-content: center;
		gap: 4px;
		padding: 0 10%;
		pointer-events: none;
		z-index: 5;
	}

	.viz-bar {
		flex: 1;
		max-width: 20px;
		background: linear-gradient(to top, var(--primary-cyan), var(--primary-magenta));
		border-radius: 2px 2px 0 0;
		transition: height 0.05s ease;
		opacity: 0.7;
	}

	/* Floating shapes - music reactive */
	.shapes {
		position: fixed;
		top: 0;
		left: 0;
		width: 100%;
		height: 100%;
		pointer-events: none;
		z-index: 0;
		opacity: 0;
		transition: opacity 1s ease;
	}

	.shapes.mounted {
		opacity: 1;
	}

	.shapes.playing .shape {
		animation-duration: 3s;
	}

	.shape {
		position: absolute;
		opacity: calc(0.15 + var(--energy, 0) * 0.3);
		animation: float 20s ease-in-out infinite;
		transition: opacity 0.3s ease;
	}

	.shape.circle {
		width: calc(300px + var(--bass, 0) * 100px);
		height: calc(300px + var(--bass, 0) * 100px);
		border: 2px solid var(--primary-cyan);
		border-radius: 50%;
		top: 10%;
		left: 5%;
		animation-delay: 0s;
		transition: width 0.1s, height 0.1s;
	}

	.shape.square {
		width: 200px;
		height: 200px;
		border: 2px solid var(--primary-magenta);
		top: 60%;
		right: 10%;
		animation-delay: -5s;
		animation-duration: 25s;
	}

	.shape.triangle {
		width: 0;
		height: 0;
		border-left: 100px solid transparent;
		border-right: 100px solid transparent;
		border-bottom: 173px solid transparent;
		border-bottom-color: var(--primary-purple);
		opacity: 0.1;
		top: 30%;
		right: 20%;
		animation-delay: -10s;
		animation-duration: 30s;
	}

	.shape.small {
		transform: scale(0.4);
	}

	.shape.small.circle {
		top: 70%;
		left: 15%;
		animation-delay: -7s;
	}

	.shape.small.square {
		top: 20%;
		right: 30%;
		animation-delay: -12s;
	}

	.shape.small.triangle {
		top: 80%;
		right: 40%;
		animation-delay: -3s;
	}

	@keyframes float {
		0%, 100% {
			transform: translate(0, 0) rotate(0deg);
		}
		25% {
			transform: translate(30px, -30px) rotate(5deg);
		}
		50% {
			transform: translate(-20px, 20px) rotate(-5deg);
		}
		75% {
			transform: translate(20px, 10px) rotate(3deg);
		}
	}

	/* Hero Section */
	.hero {
		min-height: 100vh;
		display: flex;
		align-items: center;
		justify-content: center;
		position: relative;
		padding: 2rem;
		overflow: hidden;
	}

	.hero-content {
		text-align: center;
		z-index: 1;
		max-width: 800px;
	}

	/* TITLE - Using game font */
	.title {
		font-family: var(--font-game);
		font-size: clamp(4rem, 15vw, 10rem);
		font-weight: normal;
		margin-bottom: 1rem;
		letter-spacing: 0.15em;
		display: flex;
		justify-content: center;
		gap: 0.02em;
	}

	.letter {
		display: inline-block;
		opacity: 0;
		transform: translateY(50px) scale(calc(1 + var(--bass, 0) * 0.1));
		animation: letterReveal 0.8s ease forwards;
		text-shadow:
			0 0 10px var(--primary-cyan),
			0 0 20px var(--primary-cyan),
			0 0 40px var(--primary-cyan),
			0 0 80px var(--primary-blue);
		transition: transform 0.1s ease, text-shadow 0.1s ease;
	}

	.title.playing .letter {
		text-shadow:
			0 0 calc(10px + var(--bass, 0) * 20px) var(--primary-cyan),
			0 0 calc(20px + var(--bass, 0) * 40px) var(--primary-cyan),
			0 0 calc(40px + var(--energy, 0) * 60px) var(--primary-magenta),
			0 0 calc(80px + var(--energy, 0) * 80px) var(--primary-blue);
	}

	.mounted .letter:nth-child(1) { animation-delay: 0.1s; }
	.mounted .letter:nth-child(2) { animation-delay: 0.2s; }
	.mounted .letter:nth-child(3) { animation-delay: 0.3s; }
	.mounted .letter:nth-child(4) { animation-delay: 0.4s; }
	.mounted .letter:nth-child(5) { animation-delay: 0.5s; }
	.mounted .letter:nth-child(6) { animation-delay: 0.6s; }
	.mounted .letter:nth-child(7) { animation-delay: 0.7s; }

	@keyframes letterReveal {
		to {
			opacity: 1;
			transform: translateY(0);
		}
	}

	.tagline {
		font-size: 1.5rem;
		color: var(--text-dim);
		margin-bottom: 2rem;
		letter-spacing: 0.3em;
		text-transform: uppercase;
		opacity: 0;
		transform: translateY(20px);
		transition: all 0.8s ease 0.8s;
	}

	.tagline.mounted {
		opacity: 1;
		transform: translateY(0);
	}

	.hero-description {
		max-width: 600px;
		margin: 0 auto 3rem;
		opacity: 0;
		transform: translateY(20px);
		transition: all 0.8s ease 1s;
	}

	.hero-description.mounted {
		opacity: 1;
		transform: translateY(0);
	}

	.hero-description p {
		font-size: 1.2rem;
		line-height: 1.8;
	}

	.hero-description strong {
		color: var(--primary-cyan);
	}

	.cta-buttons {
		display: flex;
		flex-direction: column;
		align-items: center;
		gap: 1rem;
		opacity: 0;
		transform: translateY(20px);
		transition: all 0.8s ease 1.2s;
	}

	.cta-buttons.mounted {
		opacity: 1;
		transform: translateY(0);
	}

	.btn.pulse {
		animation: btnPulse 0.15s ease;
	}

	@keyframes btnPulse {
		0% { transform: scale(1); }
		50% { transform: scale(1.05); }
		100% { transform: scale(1); }
	}

	.download-info {
		font-size: 0.9rem;
		color: var(--text-dim);
	}

	/* Animated rings */
	.rings {
		position: absolute;
		top: 50%;
		left: 50%;
		transform: translate(-50%, -50%);
		pointer-events: none;
	}

	.ring {
		position: absolute;
		border: 1px solid var(--primary-cyan);
		border-radius: 50%;
		opacity: calc(0.1 + var(--bass, 0) * 0.2);
		animation: ringPulse 4s ease-out infinite;
	}

	.ring:nth-child(1) {
		width: 400px;
		height: 400px;
		top: -200px;
		left: -200px;
		animation-delay: 0s;
	}

	.ring:nth-child(2) {
		width: 600px;
		height: 600px;
		top: -300px;
		left: -300px;
		animation-delay: 1s;
	}

	.ring:nth-child(3) {
		width: 800px;
		height: 800px;
		top: -400px;
		left: -400px;
		animation-delay: 2s;
	}

	@keyframes ringPulse {
		0% {
			transform: scale(0.8);
			opacity: 0.2;
		}
		100% {
			transform: scale(1.5);
			opacity: 0;
		}
	}

	/* Features Section */
	.features {
		padding: 6rem 2rem;
		position: relative;
		z-index: 1;
	}

	.section-title {
		text-align: center;
		font-size: 2.5rem;
		margin-bottom: 4rem;
		text-shadow: var(--glow-cyan);
	}

	.features-grid {
		display: grid;
		grid-template-columns: repeat(auto-fit, minmax(280px, 1fr));
		gap: 2rem;
		max-width: 1200px;
		margin: 0 auto;
	}

	.feature-card {
		background: rgba(255, 255, 255, 0.02);
		border: 1px solid rgba(0, 255, 204, 0.2);
		border-radius: 8px;
		padding: 2rem;
		text-align: center;
		transition: all 0.3s ease;
	}

	.feature-card:hover {
		background: rgba(0, 255, 204, 0.05);
		border-color: var(--primary-cyan);
		transform: translateY(-5px);
		box-shadow: 0 10px 40px rgba(0, 255, 204, 0.1);
	}

	.feature-card.glow {
		border-color: var(--primary-cyan);
		box-shadow: 0 0 30px rgba(0, 255, 204, 0.3);
	}

	.feature-icon {
		color: var(--primary-cyan);
		margin-bottom: 1.5rem;
	}

	.feature-icon.magenta {
		color: var(--primary-magenta);
	}

	.feature-icon.purple {
		color: var(--primary-purple);
	}

	.feature-card h3 {
		font-size: 1.3rem;
		margin-bottom: 1rem;
	}

	.feature-card p {
		font-size: 1rem;
	}

	/* How to Play Section */
	.how-to-play {
		padding: 6rem 2rem;
		background: rgba(0, 0, 0, 0.3);
		position: relative;
		z-index: 1;
	}

	.steps {
		display: flex;
		flex-direction: column;
		gap: 2rem;
		max-width: 600px;
		margin: 0 auto;
	}

	.step {
		display: flex;
		align-items: flex-start;
		gap: 1.5rem;
	}

	.step-number {
		flex-shrink: 0;
		width: 50px;
		height: 50px;
		display: flex;
		align-items: center;
		justify-content: center;
		background: linear-gradient(135deg, var(--primary-cyan), var(--primary-blue));
		color: var(--bg-dark);
		font-family: var(--font-display);
		font-size: 1.5rem;
		font-weight: 700;
		border-radius: 50%;
	}

	.step-content h3 {
		font-size: 1.2rem;
		margin-bottom: 0.5rem;
	}

	.step-content p {
		font-size: 1rem;
	}

	.step-content code {
		background: rgba(0, 255, 204, 0.2);
		padding: 0.2em 0.5em;
		border-radius: 4px;
		font-family: monospace;
		color: var(--primary-cyan);
	}

	.step-content strong {
		color: var(--text-white);
	}

	/* Footer */
	footer {
		padding: 3rem 2rem;
		text-align: center;
		border-top: 1px solid rgba(0, 255, 204, 0.1);
		position: relative;
		z-index: 1;
	}

	.made-with {
		font-size: 0.9rem;
		color: var(--text-dim);
		margin-bottom: 0.5rem;
	}

	.copyright {
		font-size: 0.8rem;
		color: rgba(136, 153, 170, 0.5);
	}

	/* Responsive */
	@media (max-width: 768px) {
		.title {
			letter-spacing: 0.1em;
		}

		.tagline {
			font-size: 1rem;
			letter-spacing: 0.2em;
		}

		.features-grid {
			grid-template-columns: 1fr;
		}

		.step {
			flex-direction: column;
			text-align: center;
			align-items: center;
		}

		.mute-btn {
			top: 10px;
			right: 10px;
			padding: 10px 15px;
			font-size: 0.8rem;
		}

		.mute-btn span {
			display: none;
		}

		.visualizer {
			height: 50px;
		}
	}
</style>

<script>
	import { onMount, onDestroy } from 'svelte';

	let mounted = false;
	let downloadUrl = 'https://github.com/PxPerfectMike/signals-site/releases/latest/download/signals-demo-win64.zip';

	// Intro state - starts with intro screen
	let showIntro = true;
	let transitioning = false;

	// Download popup state
	let showDownloadPopup = false;

	function handleDownloadClick() {
		showDownloadPopup = true;
	}

	function closeDownloadPopup() {
		showDownloadPopup = false;
	}

	// Audio state
	let audio;
	let audioContext;
	let analyser;
	let dataArray;
	let isMuted = false;
	let isPlaying = false;
	let audioInitialized = false;

	// Reactive values for visuals - STRONGER reactions
	let bass = 0;
	let mids = 0;
	let highs = 0;
	let energy = 0;
	let beatPulse = 0;
	let lastBass = 0;

	// Particles
	let particles = [];
	const MAX_PARTICLES = 80;

	// Animation frame
	let animationFrame;

	function initAudio() {
		if (audioInitialized) return;

		audio = new Audio('/song14.wav');
		audio.loop = true;
		audio.volume = 0.7;
		audio.crossOrigin = 'anonymous';

		audioContext = new (window.AudioContext || window.webkitAudioContext)();
		analyser = audioContext.createAnalyser();
		analyser.fftSize = 256;
		analyser.smoothingTimeConstant = 0.3;

		const source = audioContext.createMediaElementSource(audio);
		source.connect(analyser);
		analyser.connect(audioContext.destination);

		dataArray = new Uint8Array(analyser.frequencyBinCount);
		audioInitialized = true;
	}

	async function enterSite() {
		if (transitioning) return;
		transitioning = true;

		// Initialize and start audio
		if (!audioInitialized) {
			initAudio();
		}

		try {
			if (audioContext.state === 'suspended') {
				await audioContext.resume();
			}
			await audio.play();
			isPlaying = true;
			isMuted = false;
		} catch (e) {
			console.log('Audio play failed:', e);
		}

		// Transition to main site
		showIntro = false;
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
			// Decay values when not playing
			bass *= 0.9;
			mids *= 0.9;
			highs *= 0.9;
			energy *= 0.9;
			beatPulse *= 0.85;
			return;
		}

		analyser.getByteFrequencyData(dataArray);

		// Calculate frequency bands with STRONGER response
		const bufferLength = dataArray.length;
		let bassSum = 0, midsSum = 0, highsSum = 0;

		// Bass: 0-10% of frequencies (sub bass and bass)
		const bassEnd = Math.floor(bufferLength * 0.1);
		for (let i = 0; i < bassEnd; i++) {
			bassSum += dataArray[i];
		}
		const rawBass = bassSum / bassEnd / 255;

		// Mids: 10-50% of frequencies
		const midsStart = bassEnd;
		const midsEnd = Math.floor(bufferLength * 0.5);
		for (let i = midsStart; i < midsEnd; i++) {
			midsSum += dataArray[i];
		}
		const rawMids = midsSum / (midsEnd - midsStart) / 255;

		// Highs: 50-100% of frequencies
		for (let i = midsEnd; i < bufferLength; i++) {
			highsSum += dataArray[i];
		}
		const rawHighs = highsSum / (bufferLength - midsEnd) / 255;

		// Apply with smoothing but keep it punchy
		bass = bass * 0.3 + rawBass * 0.7;
		mids = mids * 0.4 + rawMids * 0.6;
		highs = highs * 0.5 + rawHighs * 0.5;

		// Overall energy
		energy = (bass * 0.5 + mids * 0.3 + highs * 0.2);

		// Beat detection - trigger on bass spikes
		if (bass > lastBass + 0.15 && bass > 0.4) {
			beatPulse = 1;
			// Spawn burst of particles on beat
			for (let i = 0; i < 5; i++) {
				spawnParticle(true);
			}
		} else {
			beatPulse *= 0.85;
		}
		lastBass = bass;
	}

	function spawnParticle(onBeat = false) {
		if (particles.length >= MAX_PARTICLES) {
			// Remove oldest particle
			particles = particles.slice(1);
		}

		const baseSpeed = onBeat ? 4 : 2;
		const size = onBeat ? (8 + bass * 15) : (4 + bass * 8);

		particles = [...particles, {
			id: Math.random(),
			x: 10 + Math.random() * 80,
			y: 105,
			vx: (Math.random() - 0.5) * 3,
			vy: -baseSpeed - Math.random() * 3,
			size: size,
			hue: Math.random() * 60 + 160,
			life: 1,
			onBeat: onBeat
		}];
	}

	function updateParticles(dt) {
		particles = particles
			.map(p => ({
				...p,
				x: p.x + p.vx * dt * 40,
				y: p.y + p.vy * dt * 40,
				vy: p.vy + 0.15 * dt * 40,
				life: p.life - dt * 0.4
			}))
			.filter(p => p.life > 0 && p.y < 120 && p.y > -10);
	}

	let lastTime = 0;
	function animate(time) {
		const dt = Math.min((time - lastTime) / 1000, 0.1);
		lastTime = time;

		updateAudioData();
		updateParticles(dt);

		// Spawn particles randomly based on energy
		if (isPlaying && Math.random() < energy * 0.3) {
			spawnParticle(false);
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

<!-- INTRO SCREEN -->
{#if showIntro}
<div class="intro-screen" class:mounted>
	<h1 class="intro-title">
		<span class="letter">S</span><span class="letter">I</span><span class="letter">G</span><span class="letter">N</span><span class="letter">A</span><span class="letter">L</span><span class="letter">S</span>
	</h1>

	<button class="enter-btn" class:mounted on:click={enterSite}>
		<span class="btn-text">Feel the Rhythm</span>
		<div class="btn-glow"></div>
	</button>
</div>
{:else}
<!-- MAIN SITE (after intro) -->

<!-- Music-reactive background layer -->
<div
	class="reactive-bg"
	class:playing={isPlaying}
	style="
		--bass: {bass};
		--mids: {mids};
		--highs: {highs};
		--energy: {energy};
		--beat: {beatPulse};
	"
></div>

<!-- Flash overlay on beats -->
{#if beatPulse > 0.5}
<div class="beat-flash" style="opacity: {beatPulse * 0.3};"></div>
{/if}

<!-- Particle layer -->
<div class="particles">
	{#each particles as particle (particle.id)}
		<div
			class="particle"
			class:on-beat={particle.onBeat}
			style="
				left: {particle.x}%;
				top: {particle.y}%;
				width: {particle.size}px;
				height: {particle.size}px;
				background: hsl({particle.hue}, 100%, {particle.onBeat ? 70 : 60}%);
				opacity: {particle.life};
				box-shadow: 0 0 {particle.size * 2}px hsl({particle.hue}, 100%, 50%),
				            0 0 {particle.size * 4}px hsl({particle.hue}, 100%, 30%);
			"
		></div>
	{/each}
</div>

<!-- Floating geometric shapes -->
<div
	class="shapes"
	class:playing={isPlaying}
	style="--bass: {bass}; --energy: {energy}; --beat: {beatPulse};"
>
	<div class="shape circle"></div>
	<div class="shape square"></div>
	<div class="shape triangle"></div>
	<div class="shape circle small"></div>
	<div class="shape square small"></div>
	<div class="shape triangle small"></div>
</div>

<!-- Beat rings -->
<div class="beat-rings" class:playing={isPlaying} style="--bass: {bass}; --energy: {energy}; --beat: {beatPulse};">
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

<!-- Hero Section -->
<section class="hero">
	<div class="hero-content">
		<h1
			class="title"
			class:playing={isPlaying}
			style="--bass: {bass}; --energy: {energy}; --beat: {beatPulse};"
		>
			<span class="letter">S</span><span class="letter">I</span><span class="letter">G</span><span class="letter">N</span><span class="letter">A</span><span class="letter">L</span><span class="letter">S</span>
		</h1>
		<p class="tagline">A Music-Driven Survival Experience</p>

		<div class="hero-description">
			<p class="desc-line">Dive into a neon-soaked world where music is your guide and survival is your goal.</p>
			<p class="desc-line">Every beat pulses through the environment, every enemy moves to the rhythm.</p>
			<p class="desc-cta">Survive the song to win.</p>
		</div>

		<div class="cta-buttons">
			<a
				href={downloadUrl}
				class="btn btn-primary"
				class:pulse={isPlaying && beatPulse > 0.5}
				style="transform: scale({1 + beatPulse * 0.08});"
				download
				on:click={handleDownloadClick}
			>
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
	<div class="rings" style="--bass: {bass}; --beat: {beatPulse};">
		<div class="ring"></div>
		<div class="ring"></div>
		<div class="ring"></div>
	</div>
</section>

<!-- Features Section -->
<section class="features">
	<h2 class="section-title">Features</h2>

	<div class="features-grid">
		<div class="feature-card" class:glow={isPlaying && bass > 0.35} style="transform: scale({1 + bass * 0.05});">
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

		<div class="feature-card" class:glow={isPlaying && mids > 0.35} style="transform: scale({1 + mids * 0.05});">
			<div class="feature-icon magenta">
				<svg xmlns="http://www.w3.org/2000/svg" width="48" height="48" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.5">
					<polygon points="12 2 15.09 8.26 22 9.27 17 14.14 18.18 21.02 12 17.77 5.82 21.02 7 14.14 2 9.27 8.91 8.26 12 2"></polygon>
				</svg>
			</div>
			<h3>Survival Gameplay</h3>
			<p>Auto-aim lets you focus on movement. Dodge waves of music-synced enemies and survive until the song ends.</p>
		</div>

		<div class="feature-card" class:glow={isPlaying && highs > 0.35} style="transform: scale({1 + highs * 0.05});">
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

<!-- Download Instructions Popup -->
{#if showDownloadPopup}
<div class="download-popup-overlay" on:click={closeDownloadPopup}>
	<div class="download-popup" on:click|stopPropagation>
		<button class="popup-close" on:click={closeDownloadPopup}>&times;</button>

		<h3>Almost there!</h3>

		<div class="popup-steps">
			<div class="popup-step">
				<span class="step-num">1</span>
				<span class="step-text"><strong>Extract</strong> the downloaded .zip file</span>
			</div>
			<div class="popup-step">
				<span class="step-num">2</span>
				<span class="step-text">Windows may warn you — click <strong>"More info"</strong></span>
			</div>
			<div class="popup-step">
				<span class="step-num">3</span>
				<span class="step-text">Then click <strong>"Run anyway"</strong></span>
			</div>
		</div>

		<p class="popup-note">The game is safe — Windows just doesn't recognize it yet!</p>

		<button class="popup-got-it" on:click={closeDownloadPopup}>Got it!</button>
	</div>
</div>
{/if}
{/if}

<style>
	/* ==================== INTRO SCREEN ==================== */
	.intro-screen {
		position: fixed;
		top: 0;
		left: 0;
		width: 100%;
		height: 100%;
		display: flex;
		flex-direction: column;
		align-items: center;
		justify-content: center;
		background: var(--bg-darker);
		z-index: 9999;
	}

	.intro-title {
		font-family: var(--font-game);
		font-size: clamp(5rem, 20vw, 14rem);
		font-weight: normal;
		letter-spacing: 0.2em;
		display: flex;
		justify-content: center;
		gap: 0.02em;
		margin-bottom: 4rem;
	}

	.intro-title .letter {
		display: inline-block;
		opacity: 0;
		transform: translateY(80px);
		text-shadow:
			0 0 20px var(--primary-cyan),
			0 0 40px var(--primary-cyan),
			0 0 60px var(--primary-cyan),
			0 0 100px var(--primary-blue);
	}

	.intro-screen.mounted .intro-title .letter {
		animation: introLetterReveal 1s ease forwards;
	}

	.intro-screen.mounted .intro-title .letter:nth-child(1) { animation-delay: 0.1s; }
	.intro-screen.mounted .intro-title .letter:nth-child(2) { animation-delay: 0.2s; }
	.intro-screen.mounted .intro-title .letter:nth-child(3) { animation-delay: 0.3s; }
	.intro-screen.mounted .intro-title .letter:nth-child(4) { animation-delay: 0.4s; }
	.intro-screen.mounted .intro-title .letter:nth-child(5) { animation-delay: 0.5s; }
	.intro-screen.mounted .intro-title .letter:nth-child(6) { animation-delay: 0.6s; }
	.intro-screen.mounted .intro-title .letter:nth-child(7) { animation-delay: 0.7s; }

	@keyframes introLetterReveal {
		to {
			opacity: 1;
			transform: translateY(0);
		}
	}

	/* Enter button */
	.enter-btn {
		position: relative;
		padding: 1.2rem 3rem;
		font-family: var(--font-display);
		font-size: 1.3rem;
		font-weight: 600;
		letter-spacing: 0.2em;
		text-transform: uppercase;
		background: transparent;
		border: 2px solid var(--primary-cyan);
		color: var(--primary-cyan);
		cursor: pointer;
		overflow: hidden;
		opacity: 0;
		transform: translateY(30px);
		transition: all 0.4s ease;
	}

	.enter-btn.mounted {
		animation: btnFadeIn 0.8s ease 1.2s forwards;
	}

	@keyframes btnFadeIn {
		to {
			opacity: 1;
			transform: translateY(0);
		}
	}

	.enter-btn:hover {
		background: rgba(0, 255, 204, 0.15);
		box-shadow:
			0 0 30px rgba(0, 255, 204, 0.5),
			0 0 60px rgba(0, 255, 204, 0.3),
			inset 0 0 30px rgba(0, 255, 204, 0.1);
		transform: translateY(-3px);
		color: var(--text-white);
	}

	.enter-btn:hover .btn-text {
		text-shadow: 0 0 20px var(--primary-cyan);
	}

	.btn-glow {
		position: absolute;
		top: 50%;
		left: 50%;
		width: 0;
		height: 0;
		background: radial-gradient(circle, rgba(0, 255, 204, 0.4) 0%, transparent 70%);
		border-radius: 50%;
		transform: translate(-50%, -50%);
		transition: all 0.4s ease;
		pointer-events: none;
	}

	.enter-btn:hover .btn-glow {
		width: 300px;
		height: 300px;
	}

	/* Subtle ambient glow pulse on intro screen */
	.intro-screen::before {
		content: '';
		position: absolute;
		top: 50%;
		left: 50%;
		width: 600px;
		height: 600px;
		background: radial-gradient(circle, rgba(0, 255, 204, 0.08) 0%, transparent 70%);
		transform: translate(-50%, -50%);
		animation: ambientPulse 4s ease-in-out infinite;
	}

	@keyframes ambientPulse {
		0%, 100% { opacity: 0.3; transform: translate(-50%, -50%) scale(0.8); }
		50% { opacity: 0.6; transform: translate(-50%, -50%) scale(1.2); }
	}

	/* ==================== MAIN SITE STYLES ==================== */

	/* Music-reactive background - STRONGER */
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
			rgba(0, 255, 204, 0.08) 0%,
			rgba(255, 0, 170, 0.05) 30%,
			rgba(136, 68, 255, 0.03) 60%,
			transparent 100%
		);
		transition: all 0.05s ease;
	}

	.reactive-bg.playing {
		background: radial-gradient(
			ellipse at 50% 50%,
			rgba(0, 255, 204, calc(0.15 + var(--bass) * 0.5)) 0%,
			rgba(255, 0, 170, calc(0.08 + var(--mids) * 0.4)) 30%,
			rgba(136, 68, 255, calc(0.05 + var(--highs) * 0.3)) 60%,
			transparent 100%
		);
	}

	/* Beat flash overlay */
	.beat-flash {
		position: fixed;
		top: 0;
		left: 0;
		width: 100%;
		height: 100%;
		background: radial-gradient(ellipse at center, rgba(0, 255, 204, 0.4) 0%, transparent 70%);
		pointer-events: none;
		z-index: 100;
	}

	/* Particles - bigger glow */
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

	.particle.on-beat {
		animation: particlePop 0.3s ease-out;
	}

	@keyframes particlePop {
		0% { transform: translate(-50%, -50%) scale(0.5); }
		50% { transform: translate(-50%, -50%) scale(1.5); }
		100% { transform: translate(-50%, -50%) scale(1); }
	}

	/* Beat rings - MORE DRAMATIC */
	.beat-rings {
		position: fixed;
		top: 50%;
		left: 50%;
		transform: translate(-50%, -50%);
		pointer-events: none;
		z-index: -1;
		opacity: 0.3;
	}

	.beat-rings.playing {
		opacity: 1;
	}

	.beat-ring {
		position: absolute;
		border: 3px solid var(--primary-cyan);
		border-radius: 50%;
		width: 200px;
		height: 200px;
		top: 50%;
		left: 50%;
		transform: translate(-50%, -50%) scale(calc(0.5 + var(--bass) * 1.5 + var(--beat) * 0.5));
		opacity: calc(0.2 + var(--beat) * 0.6);
		transition: transform 0.05s ease, opacity 0.05s ease;
	}

	.beat-ring.delay-1 {
		border-color: var(--primary-magenta);
		width: 350px;
		height: 350px;
		border-width: 2px;
	}

	.beat-ring.delay-2 {
		border-color: var(--primary-purple);
		width: 500px;
		height: 500px;
		border-width: 1px;
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

	/* Floating shapes - MORE REACTIVE */
	.shapes {
		position: fixed;
		top: 0;
		left: 0;
		width: 100%;
		height: 100%;
		pointer-events: none;
		z-index: 0;
		opacity: 1;
	}

	.shape {
		position: absolute;
		opacity: 0.15;
		animation: float 20s ease-in-out infinite;
		transition: all 0.1s ease;
	}

	.shapes.playing .shape {
		opacity: calc(0.2 + var(--energy) * 0.4);
	}

	.shapes.playing .shape.circle {
		width: calc(300px + var(--bass) * 150px + var(--beat) * 100px);
		height: calc(300px + var(--bass) * 150px + var(--beat) * 100px);
		border-width: calc(2px + var(--beat) * 3px);
		box-shadow: 0 0 calc(20px + var(--beat) * 40px) rgba(0, 255, 204, calc(var(--beat) * 0.5));
	}

	.shape.circle {
		width: 300px;
		height: 300px;
		border: 2px solid var(--primary-cyan);
		border-radius: 50%;
		top: 10%;
		left: 5%;
		animation-delay: 0s;
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

	.shapes.playing .shape.square {
		transform: rotate(calc(var(--mids) * 45deg)) scale(calc(1 + var(--mids) * 0.3));
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
		animation: fadeInUp 0.8s ease;
	}

	@keyframes fadeInUp {
		from {
			opacity: 0;
			transform: translateY(30px);
		}
		to {
			opacity: 1;
			transform: translateY(0);
		}
	}

	.hero-content {
		text-align: center;
		z-index: 1;
		max-width: 800px;
	}

	/* TITLE - Using game font - SUPER REACTIVE */
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

	.title .letter {
		display: inline-block;
		text-shadow:
			0 0 10px var(--primary-cyan),
			0 0 20px var(--primary-cyan),
			0 0 40px var(--primary-cyan),
			0 0 80px var(--primary-blue);
		transition: text-shadow 0.05s ease, transform 0.05s ease;
	}

	.title.playing .letter {
		transform: translateY(calc(var(--beat) * -10px)) scale(calc(1 + var(--beat) * 0.15));
		text-shadow:
			0 0 calc(15px + var(--bass) * 30px) var(--primary-cyan),
			0 0 calc(30px + var(--bass) * 50px) var(--primary-cyan),
			0 0 calc(60px + var(--energy) * 80px) var(--primary-magenta),
			0 0 calc(100px + var(--energy) * 100px) var(--primary-blue);
	}

	.tagline {
		font-size: 1.5rem;
		color: var(--text-dim);
		margin-bottom: 2rem;
		letter-spacing: 0.3em;
		text-transform: uppercase;
		animation: fadeIn 0.8s ease 0.2s backwards;
	}

	@keyframes fadeIn {
		from { opacity: 0; }
		to { opacity: 1; }
	}

	/* Hero description - REFORMATTED */
	.hero-description {
		max-width: 700px;
		margin: 0 auto 3rem;
		animation: fadeIn 0.8s ease 0.4s backwards;
	}

	.desc-line {
		font-size: 1.15rem;
		line-height: 1.6;
		margin-bottom: 0.3rem;
		color: var(--text-dim);
	}

	.desc-cta {
		font-size: 1.3rem;
		font-weight: 600;
		color: var(--primary-cyan);
		margin-top: 1rem;
		text-shadow: 0 0 20px rgba(0, 255, 204, 0.5);
	}

	.cta-buttons {
		display: flex;
		flex-direction: column;
		align-items: center;
		gap: 1rem;
		animation: fadeIn 0.8s ease 0.6s backwards;
	}

	.btn.pulse {
		box-shadow: 0 0 40px rgba(0, 255, 204, 0.6), 0 0 80px rgba(255, 0, 170, 0.3);
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
		opacity: calc(0.1 + var(--bass) * 0.3 + var(--beat) * 0.3);
		animation: ringPulse 4s ease-out infinite;
		transition: opacity 0.1s ease;
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
		transition: all 0.1s ease;
	}

	.feature-card:hover {
		background: rgba(0, 255, 204, 0.05);
		border-color: var(--primary-cyan);
		box-shadow: 0 10px 40px rgba(0, 255, 204, 0.1);
	}

	.feature-card.glow {
		border-color: var(--primary-cyan);
		box-shadow: 0 0 40px rgba(0, 255, 204, 0.4), 0 0 80px rgba(0, 255, 204, 0.2);
		background: rgba(0, 255, 204, 0.08);
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

	/* Download Popup */
	.download-popup-overlay {
		position: fixed;
		top: 0;
		left: 0;
		width: 100%;
		height: 100%;
		background: rgba(0, 0, 0, 0.85);
		display: flex;
		align-items: center;
		justify-content: center;
		z-index: 9999;
		animation: fadeIn 0.2s ease;
	}

	.download-popup {
		background: linear-gradient(135deg, rgba(2, 2, 8, 0.98) 0%, rgba(10, 10, 20, 0.98) 100%);
		border: 2px solid var(--primary-cyan);
		border-radius: 16px;
		padding: 2rem 2.5rem;
		max-width: 400px;
		width: 90%;
		position: relative;
		box-shadow:
			0 0 40px rgba(0, 255, 204, 0.3),
			0 0 80px rgba(0, 255, 204, 0.1),
			inset 0 0 60px rgba(0, 255, 204, 0.05);
		animation: popIn 0.3s ease;
	}

	@keyframes popIn {
		from {
			transform: scale(0.9);
			opacity: 0;
		}
		to {
			transform: scale(1);
			opacity: 1;
		}
	}

	.popup-close {
		position: absolute;
		top: 12px;
		right: 16px;
		background: none;
		border: none;
		color: var(--text-dim);
		font-size: 2rem;
		cursor: pointer;
		transition: color 0.2s ease;
		line-height: 1;
	}

	.popup-close:hover {
		color: var(--primary-cyan);
	}

	.download-popup h3 {
		font-family: var(--font-display);
		font-size: 1.5rem;
		color: var(--primary-cyan);
		margin-bottom: 1.5rem;
		text-align: center;
		text-shadow: 0 0 20px rgba(0, 255, 204, 0.5);
	}

	.popup-steps {
		display: flex;
		flex-direction: column;
		gap: 1rem;
		margin-bottom: 1.5rem;
	}

	.popup-step {
		display: flex;
		align-items: center;
		gap: 1rem;
	}

	.step-num {
		flex-shrink: 0;
		width: 28px;
		height: 28px;
		display: flex;
		align-items: center;
		justify-content: center;
		background: var(--primary-cyan);
		color: var(--bg-dark);
		font-family: var(--font-display);
		font-weight: 700;
		font-size: 0.9rem;
		border-radius: 50%;
	}

	.step-text {
		font-family: var(--font-body);
		font-size: 1.05rem;
		color: var(--text-dim);
	}

	.step-text strong {
		color: var(--text-white);
	}

	.popup-note {
		font-size: 0.85rem;
		color: var(--text-dim);
		text-align: center;
		margin-bottom: 1.5rem;
		opacity: 0.8;
	}

	.popup-got-it {
		display: block;
		width: 100%;
		padding: 0.9rem;
		font-family: var(--font-display);
		font-size: 1rem;
		font-weight: 600;
		letter-spacing: 0.1em;
		text-transform: uppercase;
		background: linear-gradient(135deg, var(--primary-cyan), var(--primary-blue));
		border: none;
		border-radius: 8px;
		color: var(--bg-dark);
		cursor: pointer;
		transition: all 0.3s ease;
	}

	.popup-got-it:hover {
		background: linear-gradient(135deg, var(--primary-magenta), var(--primary-purple));
		color: var(--text-white);
		box-shadow: 0 0 30px rgba(255, 0, 170, 0.5);
	}

	/* Responsive */
	@media (max-width: 768px) {
		.intro-title {
			letter-spacing: 0.1em;
		}

		.enter-btn {
			padding: 1rem 2rem;
			font-size: 1rem;
		}

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

		.desc-line {
			font-size: 1rem;
		}

		.desc-cta {
			font-size: 1.1rem;
		}
	}
</style>

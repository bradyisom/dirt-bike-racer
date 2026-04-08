<script lang="ts">
	import { onMount, onDestroy } from 'svelte';

	let canvas: HTMLCanvasElement;
	let ctx: CanvasRenderingContext2D;
	let animationId: number;
	let scrollOffset = 0;
	let currentScrollSpeed = 8;
	const baseScrollSpeed = 8;
	const maxScrollSpeed = 18;
	const speedIncreaseRate = 0.002; // Speed increase per frame

	// Ground parameters
	const groundHeightRatio = 0.35; // Ground takes up 35% of the screen

	// Bike sprite
	let bikeImage!: HTMLImageElement;
	let bikeImageLoaded = false;
	const bikeScale = 0.35; // Scale down the 288x288 sprite
	const bikeX = 150; // Fixed X position on left side

	// Bike physics
	let bikeY = 0; // Will be set on mount
	let velocityY = 0;
	const gravity = 1.2;
	const jumpForce = -22;
	let isOnGround = true;
	let bikeRotation = 0; // Rotation in radians
	const maxWheelieAngle = -0.4; // Tilt back when going up (negative = nose up)
	const maxDiveAngle = 0.3; // Tilt forward when coming down (positive = nose down)
	const rotationSmoothing = 0.1; // How smoothly the bike rotates
	const backflipSpeed = 0.25; // Rotation speed for backflip
	let isDoingBackflip = false;
	let wasDoingBackflip = false; // Track if we just exited a backflip
	let holdingLeft = false;

	// Game state
	let gameState: 'start' | 'playing' | 'gameover' = 'start';
	let score = 0;
	let titleFlashTimer = 0;

	// Obstacle types
	type ObstacleType = 'rock' | 'tire' | 'crate' | 'ramp';
	interface Obstacle {
		x: number;
		type: ObstacleType;
		width: number;
		height: number;
	}

	// Ramps/hills
	interface Ramp {
		x: number;
		width: number;
		height: number;
	}

	let obstacles: Obstacle[] = [];
	let ramps: Ramp[] = [];
	const minObstacleSpacing = 400;
	const maxObstacleSpacing = 700;

	function getGroundY(): number {
		return canvas.height * (1 - groundHeightRatio);
	}

	function getBikeHeight(): number {
		return bikeImageLoaded ? bikeImage.height * bikeScale : 100;
	}

	function getBikeGroundY(): number {
		// The bike's Y position when sitting on the ground
		return getGroundY() - getBikeHeight() / 2;
	}

	function handleKeyDown(e: KeyboardEvent) {
		if (e.code === 'Space' || e.code === 'ArrowUp') {
			e.preventDefault();
			if (gameState === 'start') {
				startGame();
			} else if (gameState === 'gameover') {
				restartGame();
			} else if (isOnGround) {
				velocityY = jumpForce;
				isOnGround = false;
			}
		}
		if (e.code === 'ArrowLeft') {
			holdingLeft = true;
			if (!isOnGround) {
				isDoingBackflip = true;
			}
		}
	}

	function startGame() {
		gameState = 'playing';
		spawnInitialObstacles();
	}

	function handleKeyUp(e: KeyboardEvent) {
		if (e.code === 'ArrowLeft') {
			holdingLeft = false;
		}
	}

	function restartGame() {
		gameState = 'playing';
		score = 0;
		currentScrollSpeed = baseScrollSpeed;
		obstacles = [];
		ramps = [];
		bikeY = getBikeGroundY();
		velocityY = 0;
		isOnGround = true;
		bikeRotation = 0;
		isDoingBackflip = false;
		scrollOffset = 0;
		spawnInitialObstacles();
	}

	function spawnInitialObstacles() {
		let x = canvas.width + 200;
		for (let i = 0; i < 5; i++) {
			spawnObstacleAt(x);
			x += minObstacleSpacing + Math.random() * (maxObstacleSpacing - minObstacleSpacing);
		}
	}

	function spawnObstacleAt(x: number) {
		const types: ObstacleType[] = ['rock', 'tire', 'crate'];
		const type = types[Math.floor(Math.random() * types.length)];

		let width: number;
		let height: number;
		switch (type) {
			case 'rock':
				width = 40 + Math.random() * 20;
				height = 30 + Math.random() * 15;
				break;
			case 'tire':
				width = 35;
				height = 35;
				break;
			case 'crate':
			default:
				width = 40;
				height = 40;
				break;
		}

		obstacles.push({ x, type, width, height });

		// Occasionally add a ramp before the obstacle
		if (Math.random() < 0.3) {
			ramps.push({
				x: x - 150 - Math.random() * 50,
				width: 100 + Math.random() * 50,
				height: 35 + Math.random() * 25
			});
		}
	}

	function updateObstacles() {
		// Move obstacles left
		for (const obstacle of obstacles) {
			obstacle.x -= currentScrollSpeed;
		}
		for (const ramp of ramps) {
			ramp.x -= currentScrollSpeed;
		}

		// Remove off-screen obstacles and add score
		const beforeCount = obstacles.length;
		obstacles = obstacles.filter(o => o.x > -100);
		score += (beforeCount - obstacles.length);

		ramps = ramps.filter(r => r.x > -200);

		// Spawn new obstacles
		const rightmostObstacle = obstacles.length > 0 ? Math.max(...obstacles.map(o => o.x)) : 0;
		if (rightmostObstacle < canvas.width + 200) {
			const spacing = minObstacleSpacing + Math.random() * (maxObstacleSpacing - minObstacleSpacing);
			spawnObstacleAt(rightmostObstacle + spacing);
		}
	}

	function drawObstacle(obstacle: Obstacle) {
		const groundY = getGroundY();

		switch (obstacle.type) {
			case 'rock':
				// Draw a rock
				ctx.fillStyle = '#555';
				ctx.beginPath();
				ctx.ellipse(obstacle.x, groundY - obstacle.height / 2, obstacle.width / 2, obstacle.height / 2, 0, 0, Math.PI * 2);
				ctx.fill();
				// Highlight
				ctx.fillStyle = '#777';
				ctx.beginPath();
				ctx.ellipse(obstacle.x - obstacle.width * 0.15, groundY - obstacle.height * 0.6, obstacle.width * 0.25, obstacle.height * 0.2, -0.3, 0, Math.PI * 2);
				ctx.fill();
				break;

			case 'tire':
				// Draw a tire
				ctx.fillStyle = '#1a1a1a';
				ctx.beginPath();
				ctx.arc(obstacle.x, groundY - obstacle.height / 2, obstacle.width / 2, 0, Math.PI * 2);
				ctx.fill();
				// Inner circle
				ctx.fillStyle = '#333';
				ctx.beginPath();
				ctx.arc(obstacle.x, groundY - obstacle.height / 2, obstacle.width * 0.3, 0, Math.PI * 2);
				ctx.fill();
				// Hub
				ctx.fillStyle = '#666';
				ctx.beginPath();
				ctx.arc(obstacle.x, groundY - obstacle.height / 2, obstacle.width * 0.15, 0, Math.PI * 2);
				ctx.fill();
				break;

			case 'crate':
				// Draw a wooden crate
				ctx.fillStyle = '#8B4513';
				ctx.fillRect(obstacle.x - obstacle.width / 2, groundY - obstacle.height, obstacle.width, obstacle.height);
				// Crate lines
				ctx.strokeStyle = '#5D3A1A';
				ctx.lineWidth = 2;
				ctx.strokeRect(obstacle.x - obstacle.width / 2, groundY - obstacle.height, obstacle.width, obstacle.height);
				// Cross pattern
				ctx.beginPath();
				ctx.moveTo(obstacle.x - obstacle.width / 2, groundY - obstacle.height);
				ctx.lineTo(obstacle.x + obstacle.width / 2, groundY);
				ctx.moveTo(obstacle.x + obstacle.width / 2, groundY - obstacle.height);
				ctx.lineTo(obstacle.x - obstacle.width / 2, groundY);
				ctx.stroke();
				break;
		}
	}

	function drawRamp(ramp: Ramp) {
		const groundY = getGroundY();

		// Draw ramp as a triangle/slope
		ctx.fillStyle = '#A0522D';
		ctx.beginPath();
		ctx.moveTo(ramp.x, groundY);
		ctx.lineTo(ramp.x + ramp.width, groundY);
		ctx.lineTo(ramp.x + ramp.width, groundY - ramp.height);
		ctx.closePath();
		ctx.fill();

		// Ramp surface line
		ctx.strokeStyle = '#8B4513';
		ctx.lineWidth = 3;
		ctx.beginPath();
		ctx.moveTo(ramp.x, groundY);
		ctx.lineTo(ramp.x + ramp.width, groundY - ramp.height);
		ctx.stroke();
	}

	function drawObstacles() {
		for (const ramp of ramps) {
			drawRamp(ramp);
		}
		for (const obstacle of obstacles) {
			drawObstacle(obstacle);
		}
	}

	function getBikeHitbox() {
		const width = bikeImageLoaded ? bikeImage.width * bikeScale * 0.6 : 60;
		const height = bikeImageLoaded ? bikeImage.height * bikeScale * 0.5 : 50;
		return {
			x: bikeX - width / 2,
			y: bikeY - height / 2,
			width,
			height
		};
	}

	function checkCollisions() {
		const bike = getBikeHitbox();
		const groundY = getGroundY();

		for (const obstacle of obstacles) {
			const obstacleBox = {
				x: obstacle.x - obstacle.width / 2,
				y: groundY - obstacle.height,
				width: obstacle.width,
				height: obstacle.height
			};

			// AABB collision detection
			if (
				bike.x < obstacleBox.x + obstacleBox.width &&
				bike.x + bike.width > obstacleBox.x &&
				bike.y < obstacleBox.y + obstacleBox.height &&
				bike.y + bike.height > obstacleBox.y
			) {
				gameState = 'gameover';
				return;
			}
		}
	}

	function drawStartScreen() {
		// Dark background with gradient
		const gradient = ctx.createLinearGradient(0, 0, 0, canvas.height);
		gradient.addColorStop(0, '#1a0a2e');
		gradient.addColorStop(0.5, '#16213e');
		gradient.addColorStop(1, '#0f0f23');
		ctx.fillStyle = gradient;
		ctx.fillRect(0, 0, canvas.width, canvas.height);

		// Animated flash effect for title
		titleFlashTimer += 0.05;
		const flash = Math.sin(titleFlashTimer * 3) * 0.3 + 0.7;

		// Draw retro grid lines at bottom (80s style)
		ctx.strokeStyle = `rgba(255, 0, 255, 0.3)`;
		ctx.lineWidth = 2;
		const gridY = canvas.height * 0.7;
		for (let i = 0; i < 20; i++) {
			const x = (i / 20) * canvas.width;
			ctx.beginPath();
			ctx.moveTo(x, gridY);
			ctx.lineTo(canvas.width / 2, canvas.height);
			ctx.stroke();
		}
		// Horizontal grid lines
		for (let i = 0; i < 8; i++) {
			const y = gridY + (i / 8) * (canvas.height - gridY);
			ctx.beginPath();
			ctx.moveTo(0, y);
			ctx.lineTo(canvas.width, y);
			ctx.stroke();
		}

		// Title shadow (multiple layers for glow effect)
		ctx.textAlign = 'center';
		ctx.font = 'bold 80px Arial';

		// Cyan glow
		ctx.fillStyle = `rgba(0, 255, 255, ${flash * 0.5})`;
		ctx.fillText('DIRT BIKE RACER', canvas.width / 2 + 4, canvas.height / 3 + 4);

		// Magenta glow
		ctx.fillStyle = `rgba(255, 0, 255, ${flash * 0.5})`;
		ctx.fillText('DIRT BIKE RACER', canvas.width / 2 - 4, canvas.height / 3 - 4);

		// Main title in yellow/gold
		ctx.fillStyle = `rgb(${255 * flash}, ${220 * flash}, 0)`;
		ctx.fillText('DIRT BIKE RACER', canvas.width / 2, canvas.height / 3);

		// Subtitle
		ctx.font = 'bold 24px Arial';
		ctx.fillStyle = '#00ffff';
		ctx.fillText('EXTREME EDITION', canvas.width / 2, canvas.height / 3 + 50);

		// Flashing "Press SPACE to Start"
		const blink = Math.sin(titleFlashTimer * 5) > 0;
		if (blink) {
			ctx.font = 'bold 32px Arial';
			ctx.fillStyle = '#ffffff';
			ctx.fillText('PRESS SPACE TO START', canvas.width / 2, canvas.height / 2 + 50);
		}

		// Controls info
		ctx.font = '20px Arial';
		ctx.fillStyle = '#888888';
		ctx.fillText('SPACE / UP - Jump', canvas.width / 2, canvas.height * 0.75);
		ctx.fillText('LEFT ARROW - Backflip', canvas.width / 2, canvas.height * 0.75 + 30);

		// High score style decoration
		ctx.font = 'bold 18px Arial';
		ctx.fillStyle = '#ff00ff';
		ctx.fillText('© 2024 ARCADE CLASSICS', canvas.width / 2, canvas.height - 40);
	}

	function drawGameOver() {
		// Dark overlay with gradient
		const gradient = ctx.createLinearGradient(0, 0, 0, canvas.height);
		gradient.addColorStop(0, 'rgba(26, 10, 46, 0.85)');
		gradient.addColorStop(0.5, 'rgba(22, 33, 62, 0.85)');
		gradient.addColorStop(1, 'rgba(15, 15, 35, 0.85)');
		ctx.fillStyle = gradient;
		ctx.fillRect(0, 0, canvas.width, canvas.height);

		// Animated flash effect
		titleFlashTimer += 0.05;
		const flash = Math.sin(titleFlashTimer * 3) * 0.3 + 0.7;

		// Game over text with glow effect
		ctx.textAlign = 'center';
		ctx.font = 'bold 80px Arial';

		// Red glow layers
		ctx.fillStyle = `rgba(255, 0, 0, ${flash * 0.4})`;
		ctx.fillText('GAME OVER', canvas.width / 2 + 4, canvas.height / 2 - 60 + 4);
		ctx.fillStyle = `rgba(255, 100, 0, ${flash * 0.4})`;
		ctx.fillText('GAME OVER', canvas.width / 2 - 4, canvas.height / 2 - 60 - 4);

		// Main text in red/orange
		ctx.fillStyle = `rgb(${255 * flash}, ${50 * flash}, 0)`;
		ctx.fillText('GAME OVER', canvas.width / 2, canvas.height / 2 - 60);

		// Score with cyan glow
		ctx.font = 'bold 48px Arial';
		ctx.fillStyle = `rgba(0, 255, 255, ${flash * 0.5})`;
		ctx.fillText(`SCORE: ${score}`, canvas.width / 2 + 2, canvas.height / 2 + 20 + 2);
		ctx.fillStyle = '#00ffff';
		ctx.fillText(`SCORE: ${score}`, canvas.width / 2, canvas.height / 2 + 20);

		// Flashing restart instruction
		const blink = Math.sin(titleFlashTimer * 5) > 0;
		if (blink) {
			ctx.font = 'bold 28px Arial';
			ctx.fillStyle = '#ffffff';
			ctx.fillText('PRESS SPACE TO CONTINUE', canvas.width / 2, canvas.height / 2 + 100);
		}

		// Decorative lines
		ctx.strokeStyle = `rgba(255, 0, 255, ${flash * 0.5})`;
		ctx.lineWidth = 3;
		const lineY = canvas.height / 2 + 140;
		ctx.beginPath();
		ctx.moveTo(canvas.width / 2 - 200, lineY);
		ctx.lineTo(canvas.width / 2 + 200, lineY);
		ctx.stroke();

		// Insert coin style text
		ctx.font = 'bold 16px Arial';
		ctx.fillStyle = '#ff00ff';
		ctx.fillText('INSERT COIN', canvas.width / 2, canvas.height - 60);
	}

	function drawScore() {
		ctx.fillStyle = '#ffffff';
		ctx.strokeStyle = '#000000';
		ctx.lineWidth = 3;
		ctx.font = 'bold 24px Arial';
		ctx.textAlign = 'left';
		const text = `Score: ${score}`;
		ctx.strokeText(text, 20, 40);
		ctx.fillText(text, 20, 40);

		// Draw speedometer in top right
		drawSpeedometer();
	}

	function drawSpeedometer() {
		const centerX = canvas.width - 80;
		const centerY = 80;
		const radius = 55;

		// Speedometer background (dark semi-circle)
		ctx.beginPath();
		ctx.arc(centerX, centerY, radius, Math.PI, 0, false);
		ctx.lineTo(centerX + radius, centerY);
		ctx.arc(centerX, centerY, radius - 15, 0, Math.PI, true);
		ctx.closePath();
		ctx.fillStyle = 'rgba(0, 0, 0, 0.7)';
		ctx.fill();
		ctx.strokeStyle = '#333';
		ctx.lineWidth = 2;
		ctx.stroke();

		// Speed tick marks
		const minSpeed = baseScrollSpeed;
		const maxSpeed = maxScrollSpeed;
		const numTicks = 6;
		for (let i = 0; i <= numTicks; i++) {
			const angle = Math.PI + (i / numTicks) * Math.PI;
			const innerR = radius - 12;
			const outerR = radius - 4;
			const x1 = centerX + Math.cos(angle) * innerR;
			const y1 = centerY + Math.sin(angle) * innerR;
			const x2 = centerX + Math.cos(angle) * outerR;
			const y2 = centerY + Math.sin(angle) * outerR;

			ctx.beginPath();
			ctx.moveTo(x1, y1);
			ctx.lineTo(x2, y2);
			ctx.strokeStyle = '#888';
			ctx.lineWidth = 2;
			ctx.stroke();
		}

		// Color gradient arc showing speed zones
		const arcWidth = 8;
		const arcRadius = radius - 8;

		// Green zone (low speed)
		ctx.beginPath();
		ctx.arc(centerX, centerY, arcRadius, Math.PI, Math.PI + Math.PI * 0.33, false);
		ctx.strokeStyle = '#00ff00';
		ctx.lineWidth = arcWidth;
		ctx.stroke();

		// Yellow zone (medium speed)
		ctx.beginPath();
		ctx.arc(centerX, centerY, arcRadius, Math.PI + Math.PI * 0.33, Math.PI + Math.PI * 0.66, false);
		ctx.strokeStyle = '#ffff00';
		ctx.stroke();

		// Red zone (high speed)
		ctx.beginPath();
		ctx.arc(centerX, centerY, arcRadius, Math.PI + Math.PI * 0.66, Math.PI * 2, false);
		ctx.strokeStyle = '#ff0000';
		ctx.stroke();

		// Needle
		const speedPercent = (currentScrollSpeed - minSpeed) / (maxSpeed - minSpeed);
		const needleAngle = Math.PI + speedPercent * Math.PI;
		const needleLength = radius - 20;

		ctx.beginPath();
		ctx.moveTo(centerX, centerY);
		ctx.lineTo(
			centerX + Math.cos(needleAngle) * needleLength,
			centerY + Math.sin(needleAngle) * needleLength
		);
		ctx.strokeStyle = '#ff3333';
		ctx.lineWidth = 3;
		ctx.stroke();

		// Needle center cap
		ctx.beginPath();
		ctx.arc(centerX, centerY, 6, 0, Math.PI * 2);
		ctx.fillStyle = '#cc0000';
		ctx.fill();
		ctx.strokeStyle = '#880000';
		ctx.lineWidth = 2;
		ctx.stroke();

		// Speed text below
		ctx.font = 'bold 14px Arial';
		ctx.textAlign = 'center';
		ctx.fillStyle = '#ffffff';
		ctx.fillText(`${Math.round(currentScrollSpeed * 10)} MPH`, centerX, centerY + 20);
	}

	function resizeCanvas() {
		canvas.width = window.innerWidth;
		canvas.height = window.innerHeight;
		// Reset bike position on resize if on ground
		if (isOnGround) {
			bikeY = getBikeGroundY();
		}
	}

	function drawSky() {
		// Create gradient for sky (blue gradient from top to horizon)
		const skyGradient = ctx.createLinearGradient(0, 0, 0, canvas.height * (1 - groundHeightRatio));
		skyGradient.addColorStop(0, '#87CEEB'); // Light sky blue at top
		skyGradient.addColorStop(0.7, '#B0E0E6'); // Powder blue near horizon
		skyGradient.addColorStop(1, '#F4E4C1'); // Sandy haze at horizon

		ctx.fillStyle = skyGradient;
		ctx.fillRect(0, 0, canvas.width, canvas.height * (1 - groundHeightRatio));
	}

	function drawGround() {
		const groundY = canvas.height * (1 - groundHeightRatio);
		const groundHeight = canvas.height * groundHeightRatio;

		// Create gradient for ground (brown dirt/desert)
		const groundGradient = ctx.createLinearGradient(0, groundY, 0, canvas.height);
		groundGradient.addColorStop(0, '#D2691E'); // Chocolate brown at top
		groundGradient.addColorStop(0.3, '#CD853F'); // Peru (lighter brown)
		groundGradient.addColorStop(1, '#8B4513'); // Saddle brown at bottom

		ctx.fillStyle = groundGradient;
		ctx.fillRect(0, groundY, canvas.width, groundHeight);

		// Draw scrolling ground details (dirt lines/texture)
		drawGroundDetails(groundY, groundHeight);
	}

	function drawGroundDetails(groundY: number, groundHeight: number) {
		ctx.strokeStyle = '#A0522D'; // Sienna color for dirt lines
		ctx.lineWidth = 2;

		// Draw horizontal lines that scroll to simulate movement
		const lineSpacing = 40;
		const numLines = Math.ceil(canvas.width / lineSpacing) + 2;

		for (let i = 0; i < numLines; i++) {
			const x = (i * lineSpacing - (scrollOffset % lineSpacing));

			// Draw varied ground detail lines
			ctx.beginPath();
			ctx.moveTo(x, groundY + 10);
			ctx.lineTo(x - 20, groundY + 25);
			ctx.stroke();

			// Add some dust/pebble marks
			ctx.fillStyle = '#8B7355';
			ctx.beginPath();
			ctx.arc(x + 10, groundY + groundHeight * 0.3, 3, 0, Math.PI * 2);
			ctx.fill();

			ctx.fillStyle = '#A08060';
			ctx.beginPath();
			ctx.arc(x - 5, groundY + groundHeight * 0.5, 2, 0, Math.PI * 2);
			ctx.fill();
		}

		// Draw some larger rocks that scroll
		const rockSpacing = 200;
		const numRocks = Math.ceil(canvas.width / rockSpacing) + 2;

		for (let i = 0; i < numRocks; i++) {
			const x = (i * rockSpacing - (scrollOffset % rockSpacing)) + 50;
			drawRock(x, groundY + groundHeight * 0.6);
		}
	}

	function drawRock(x: number, y: number) {
		ctx.fillStyle = '#696969'; // Dim gray
		ctx.beginPath();
		ctx.ellipse(x, y, 12, 8, 0, 0, Math.PI * 2);
		ctx.fill();

		// Rock highlight
		ctx.fillStyle = '#808080';
		ctx.beginPath();
		ctx.ellipse(x - 3, y - 2, 6, 4, -0.3, 0, Math.PI * 2);
		ctx.fill();
	}

	function drawDistantMountains() {
		const horizonY = canvas.height * (1 - groundHeightRatio);

		// Draw distant mountains/hills
		ctx.fillStyle = '#C4A777'; // Tan/sandy color for distant hills

		const hillSpacing = 300;
		const numHills = Math.ceil(canvas.width / hillSpacing) + 2;
		const parallaxOffset = scrollOffset * 0.3; // Slower scroll for parallax effect

		for (let i = 0; i < numHills; i++) {
			const x = (i * hillSpacing - (parallaxOffset % hillSpacing));
			const hillWidth = 200 + (i % 3) * 50;
			const hillHeight = 40 + (i % 2) * 20;

			ctx.beginPath();
			ctx.moveTo(x - hillWidth / 2, horizonY);
			ctx.quadraticCurveTo(x, horizonY - hillHeight, x + hillWidth / 2, horizonY);
			ctx.fill();
		}
	}

	function drawBike() {
		if (!bikeImageLoaded) return;

		const drawWidth = bikeImage.width * bikeScale;
		const drawHeight = bikeImage.height * bikeScale;

		// Save context, apply rotation, draw, restore
		ctx.save();
		ctx.translate(bikeX, bikeY);
		ctx.rotate(bikeRotation);
		ctx.drawImage(
			bikeImage,
			-drawWidth / 2,
			-drawHeight / 2,
			drawWidth,
			drawHeight
		);

		// Draw rider on top of bike
		drawRider();

		ctx.restore();
	}

	function drawRider() {
		// Rider position relative to bike center (already transformed)
		const riderX = -5; // Slightly behind center
		const riderY = -35; // Above the bike

		// Lean angle based on velocity (rider leans back when going up, forward when coming down)
		let leanAngle = 0;
		if (!isOnGround) {
			if (isDoingBackflip) {
				leanAngle = -0.3; // Tucked during backflip
			} else if (velocityY < 0) {
				leanAngle = -0.2; // Lean back going up
			} else {
				leanAngle = 0.15; // Lean forward coming down
			}
		}

		ctx.save();
		ctx.translate(riderX, riderY);
		ctx.rotate(leanAngle);

		// Body (torso) - racing suit
		ctx.fillStyle = '#cc0000'; // Red racing suit
		ctx.beginPath();
		ctx.ellipse(0, 0, 8, 14, 0, 0, Math.PI * 2);
		ctx.fill();
		ctx.strokeStyle = '#990000';
		ctx.lineWidth = 1;
		ctx.stroke();

		// Racing stripe on suit
		ctx.fillStyle = '#ffffff';
		ctx.fillRect(-2, -10, 4, 20);

		// Head with helmet
		ctx.fillStyle = '#222222'; // Dark helmet
		ctx.beginPath();
		ctx.arc(0, -18, 10, 0, Math.PI * 2);
		ctx.fill();

		// Helmet visor
		ctx.fillStyle = '#333399'; // Blue-tinted visor
		ctx.beginPath();
		ctx.ellipse(4, -18, 6, 5, 0, -0.5, 1.5);
		ctx.fill();

		// Visor shine
		ctx.fillStyle = 'rgba(255, 255, 255, 0.3)';
		ctx.beginPath();
		ctx.ellipse(6, -20, 2, 3, 0.3, 0, Math.PI * 2);
		ctx.fill();

		// Arms holding handlebars
		ctx.strokeStyle = '#cc0000';
		ctx.lineWidth = 5;
		ctx.lineCap = 'round';

		// Left arm
		ctx.beginPath();
		ctx.moveTo(-6, -5);
		ctx.quadraticCurveTo(-15, 5, -12, 18);
		ctx.stroke();

		// Right arm
		ctx.beginPath();
		ctx.moveTo(6, -5);
		ctx.quadraticCurveTo(18, 8, 15, 20);
		ctx.stroke();

		// Gloves
		ctx.fillStyle = '#111111';
		ctx.beginPath();
		ctx.arc(-12, 18, 4, 0, Math.PI * 2);
		ctx.fill();
		ctx.beginPath();
		ctx.arc(15, 20, 4, 0, Math.PI * 2);
		ctx.fill();

		// Legs
		ctx.strokeStyle = '#cc0000';
		ctx.lineWidth = 6;

		// Left leg
		ctx.beginPath();
		ctx.moveTo(-4, 10);
		ctx.quadraticCurveTo(-10, 22, -8, 32);
		ctx.stroke();

		// Right leg
		ctx.beginPath();
		ctx.moveTo(4, 10);
		ctx.quadraticCurveTo(12, 24, 10, 34);
		ctx.stroke();

		// Boots
		ctx.fillStyle = '#111111';
		ctx.beginPath();
		ctx.ellipse(-8, 34, 5, 4, 0.2, 0, Math.PI * 2);
		ctx.fill();
		ctx.beginPath();
		ctx.ellipse(10, 36, 5, 4, -0.2, 0, Math.PI * 2);
		ctx.fill();

		ctx.restore();
	}

	function getRampHeightAtX(x: number): number {
		for (const ramp of ramps) {
			// Check if x is on this ramp
			if (x >= ramp.x && x <= ramp.x + ramp.width) {
				// Calculate height at this x position (linear interpolation)
				const progress = (x - ramp.x) / ramp.width;
				return ramp.height * progress;
			}
		}
		return 0;
	}

	function updateBike() {
		// Apply gravity
		velocityY += gravity;
		bikeY += velocityY;

		// Check ground/ramp collision
		const baseGroundY = getGroundY();
		const rampHeight = getRampHeightAtX(bikeX);
		const effectiveGroundY = baseGroundY - rampHeight - getBikeHeight() / 2;

		if (bikeY >= effectiveGroundY) {
			bikeY = effectiveGroundY;

			// If on a ramp and moving, launch the bike!
			if (rampHeight > 0 && isOnGround) {
				// Use same jump force as spacebar
				velocityY = jumpForce;
				isOnGround = false;
			} else if (rampHeight === 0) {
				velocityY = 0;
				isOnGround = true;
			}
		} else {
			isOnGround = false;
		}

		// Update bike rotation based on velocity
		if (isOnGround) {
			// On ground: stop backflip and snap to level
			if (isDoingBackflip || wasDoingBackflip) {
				// Landing from a backflip - snap rotation to level immediately
				bikeRotation = 0;
				isDoingBackflip = false;
				wasDoingBackflip = false;
			}

			let targetRotation: number;
			if (rampHeight > 0) {
				// Tilt to match ramp angle
				const ramp = ramps.find(r => bikeX >= r.x && bikeX <= r.x + r.width);
				if (ramp) {
					targetRotation = -Math.atan2(ramp.height, ramp.width);
				} else {
					targetRotation = 0;
				}
			} else {
				targetRotation = 0;
			}
			// Smoothly interpolate to target rotation
			bikeRotation += (targetRotation - bikeRotation) * rotationSmoothing;
		} else {
			// In air
			if (isDoingBackflip || holdingLeft) {
				// Do a backflip! (rotate backwards)
				isDoingBackflip = true;
				wasDoingBackflip = true;
				bikeRotation -= backflipSpeed;
			} else {
				// Just exited backflip - normalize rotation and continue forward
				if (wasDoingBackflip) {
					// Normalize to nearest upright position (multiples of 2*PI)
					bikeRotation = bikeRotation % (Math.PI * 2);
					if (bikeRotation < -Math.PI) bikeRotation += Math.PI * 2;
					if (bikeRotation > Math.PI) bikeRotation -= Math.PI * 2;
					wasDoingBackflip = false;
				}

				// Normal air rotation based on velocity
				let targetRotation: number;
				if (velocityY < 0) {
					// Going up - wheelie (nose up)
					targetRotation = maxWheelieAngle * Math.min(1, -velocityY / 20);
				} else {
					// Coming down - nose dive
					targetRotation = maxDiveAngle * Math.min(1, velocityY / 20);
				}
				// Smoothly interpolate to target rotation
				bikeRotation += (targetRotation - bikeRotation) * rotationSmoothing;
			}
		}
	}

	function draw() {
		// Clear canvas
		ctx.clearRect(0, 0, canvas.width, canvas.height);

		if (gameState === 'start') {
			drawStartScreen();
		} else {
			// Draw scene layers (back to front)
			drawSky();
			drawDistantMountains();
			drawGround();
			drawObstacles();
			drawBike();
			drawScore();

			if (gameState === 'gameover') {
				drawGameOver();
			}
		}
	}

	function gameLoop() {
		if (gameState === 'playing') {
			updateBike();
			updateObstacles();
			checkCollisions();

			// Increase speed gradually
			if (currentScrollSpeed < maxScrollSpeed) {
				currentScrollSpeed += speedIncreaseRate;
			}

			// Update scroll offset for next frame
			scrollOffset += currentScrollSpeed;
		} else if (gameState === 'start') {
			// Update title flash timer even on start screen
			titleFlashTimer += 0.016;
		}

		draw();
		animationId = requestAnimationFrame(gameLoop);
	}

	onMount(() => {
		const context = canvas.getContext('2d');
		if (!context) {
			console.error('Failed to get canvas context');
			return;
		}
		ctx = context;

		// Load bike sprite
		bikeImage = new Image();
		bikeImage.onload = () => {
			bikeImageLoaded = true;
			bikeY = getBikeGroundY();
		};
		bikeImage.src = '/dirt-bike.png';

		resizeCanvas();

		window.addEventListener('resize', resizeCanvas);
		window.addEventListener('keydown', handleKeyDown);
		window.addEventListener('keyup', handleKeyUp);

		// Start the game loop
		gameLoop();
	});

	onDestroy(() => {
		if (typeof window === 'undefined') return;
		if (animationId) {
			cancelAnimationFrame(animationId);
		}
		window.removeEventListener('resize', resizeCanvas);
		window.removeEventListener('keydown', handleKeyDown);
		window.removeEventListener('keyup', handleKeyUp);
	});
</script>

<svelte:head>
	<title>Dirt Bike Racer</title>
</svelte:head>

<canvas bind:this={canvas}></canvas>

<style>
	:global(html, body) {
		margin: 0;
		padding: 0;
		overflow: hidden;
		width: 100%;
		height: 100%;
	}

	canvas {
		display: block;
		width: 100vw;
		height: 100vh;
	}
</style>

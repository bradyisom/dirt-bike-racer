Dirt Bike Racer - Cheat Sheet
Setup Commands

npx sv create dirt-bike-racer --template minimal --types ts
cd dirt-bike-racer
npm install
npm run dev
Preview: Cmd+Shift+P → "Simple Browser: Show" → http://localhost:5173

Prompts to Copy/Paste

1. Canvas (8 min)

Create a fullscreen HTML5 canvas game component in src/routes/+page.svelte. The canvas should fill the entire browser window, have a scrolling dirt/desert background with brown ground and blue sky, use requestAnimationFrame for smooth animation, and the ground should scroll right to left to simulate movement.

2. Dirt Bike (10 min)

Add a dirt bike to the game. Draw it using simple shapes (rectangles for body, circles for wheels). Position it on the left side sitting on the ground. Add gravity so it falls when in the air. Add keyboard controls: Up arrow or Space to jump. The bike should have a smooth jump arc and land back on the ground.

3. Obstacles (10 min)

Add obstacles and terrain: Generate random ramps/hills that scroll right to left. Add rock obstacles the player must jump over. Increase scroll speed gradually over time. Detect collisions - if the bike hits a rock, trigger game over. Show "GAME OVER" text and "Press Space to Restart".

4. Polish (7 min)

Add polish: Score counter in top-left that increases over time. Start screen with title "DIRT BIKE RACER" in retro arcade style. Show "Press SPACE to Start". Use bold colors like an 80s arcade game.

5. Deploy (5 min)

Configure this SvelteKit app for static deployment to GitHub Pages. Update config files and create a GitHub Actions workflow to auto-deploy on push to main.

Deploy Commands

npm install -D @sveltejs/adapter-static
npm run build
git add .
git commit -m "Dirt Bike Racer game"
git branch -M main
git push -u origin main
GitHub → Settings → Pages → Source: GitHub Actions

Quick Fixes
Issue Prompt
Too fast/slow "Make the scroll speed slower"
Jump wrong "Make the jump higher and floatier"
Bike too small "Make the bike twice as big"
Error "I'm getting this error: [paste]. Fix it"
Talking Points

- One command = full web app
- Plain English → working code
- Iterate fast, ask for changes
- Debug by pasting errors
- Live on internet in minutes

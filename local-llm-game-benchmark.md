# Local LLM UI & Game Logic Benchmark: Gravity Shift Runner

A standardized single-prompt benchmark designed to evaluate local Large Language Models (LLMs) on both **UI/UX design** and **complex game logic**. Requiring a single standalone `.html` file prevents models from hallucinating external libraries or breaking imports, allowing you to instantly double-click and test every model's output side-by-side.

---

## 1. Benchmark Prompt

Feed this exact prompt to every local LLM using identical generation settings (recommended: `temperature = 0.2` or `0.3` for fair comparison). Unlike standard clones (Snake, Tetris, Pong) that models often memorize from training data, **Gravity Shift Runner** requires genuine spatial math, collision detection, and UI state management.

```text
Create a complete, playable 2D browser game in a single standalone HTML file (combining HTML, CSS, and vanilla JavaScript with no external libraries or image assets). 

Game Concept: 'Gravity Shift Runner'
- Core Mechanic: A square player character automatically runs from left to right across a horizontally scrolling level. Pressing `Space` or clicking flips gravity instantaneously, pulling the player toward either the floor or the ceiling.
- Level Generation: Procedurally generate platforms and spike hazards on both the floor and ceiling. Ensure gaps are mathematically jumpable/flippable based on player speed and gravity pull so impossible levels never spawn.
- UI & Polish: Include a clean, modern dark-mode UI overlay featuring a Start Screen, a live HUD (Current Score, High Score saved in `localStorage`, and a visual cooldown/charge meter for gravity flips), and a Game Over screen with a one-click 'Restart' button that resets all state cleanly without reloading the page.
- Visuals & Juice: Render everything on an HTML5 `<canvas>`. Add a fading motion trail behind the player, screen shake on collision, and particle bursts when landing on a surface.

Write the  complete, valid HTML code inside a single code block into ./index.html
```

---

## 2. How to Automate Running All Local Models

Instead of manually copy-pasting the prompt into a chat interface for each model, automate the test against your local inference server (**Ollama**, **LM Studio**, or **llama.cpp**).

1. **Start Your Local API:** Ensure your local server is active (e.g., `ollama serve`) and verify the exact model tags you want to test (`ollama list`).
2. **Run a Batch Script:** Use a Python script with the `requests` library to loop through your list of model names, send the prompt to your local endpoint (`http://localhost:11434/api/generate` for Ollama or port `1234` for LM Studio), strip the markdown ````html` fences from the response, and save each output as `<model_name>.html`.
3. **Test Side-by-Side:** Open all generated `.html` files in separate browser tabs to inspect, play, and score them simultaneously.

---

## 3. Evaluation Rubric (25 Points Total)

Grade each model's output out of **25 points** (5 points per category) to objectively rank your local models:

| Category | Max Score | Evaluation Criteria |
| :--- | :---: | :--- |
| **Zero-Shot Execution** | 5 pts | Does the file run immediately with zero browser console errors, or did the model leave placeholder comments like `// add collision logic here`? |
| **Physics & Playability** | 5 pts | Is gravity smooth using `requestAnimationFrame` with delta-time, and does the procedural generator reliably prevent impossible walls or gaps? |
| **Collision Accuracy** | 5 pts | Do hitboxes match the rendered spikes and platforms, or does the player die mid-air or clip through floors and ceilings? |
| **State & Reset Hygiene** | 5 pts | Does clicking "Restart" reset speed, score, timers, and obstacles cleanly, or do old obstacles remain and accelerate the game endlessly? |
| **Visual & UI Craft** | 5 pts | Are the particle effects, motion trail, dark-mode overlays, and HUD readable, properly aligned, and responsive to window resizing? |

### Scoring Guide
* **22–25 Points:** Production-capable local model; strong spatial reasoning, clean state management, and polished UI.
* **16–21 Points:** Playable baseline; minor physics quirks, hitbox offsets, or reset bugs.
* **Below 16 Points:** Struggles with multi-constraint logic; likely suffers from syntax errors, broken game loops, or unplayable level generation.

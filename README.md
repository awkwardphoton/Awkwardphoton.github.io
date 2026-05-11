
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>The Perfect Word Search</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <style>
        @import url('https://fonts.googleapis.com/css2?family=JetBrains+Mono:wght@700&family=Inter:wght@400;600&display=swap');

        :root {
            --cell-size: clamp(20px, 5.5vw, 40px);
        }

        body {
            font-family: 'Inter', sans-serif;
            background-color: #ffffff;
            touch-action: none;
            color: #000;
        }

        /* Basic Aesthetic */
        .grid-container {
            display: grid;
            gap: 1px;
            background-color: #000;
            border: 2px solid #000;
            user-select: none;
            touch-action: none;
        }

        .cell {
            width: var(--cell-size);
            height: var(--cell-size);
            background-color: white;
            display: flex;
            align-items: center;
            justify-content: center;
            font-family: 'JetBrains+Mono', monospace;
            font-size: calc(var(--cell-size) * 0.65);
            font-weight: 700;
            cursor: crosshair;
        }

        /* Selection Lines */
        #selection-layer {
            position: absolute;
            top: 0;
            left: 0;
            pointer-events: none;
            z-index: 10;
        }

        .found-word-line {
            position: absolute;
            background-color: rgba(0, 0, 0, 0.1);
            border: 1px solid #000;
            border-radius: 999px;
            pointer-events: none;
            transform-origin: center left;
            z-index: 5;
        }

        .current-selection-line {
            position: absolute;
            background-color: rgba(0, 0, 0, 0.05);
            border: 1px dashed #000;
            border-radius: 999px;
            pointer-events: none;
            transform-origin: center left;
            z-index: 6;
        }

        .word-item.found {
            text-decoration: line-through;
            opacity: 0.3;
        }

        /* Immersive Mode Logic */
        .immersive-active #config-panel,
        .immersive-active header:not(.immersive-header) {
            display: none;
        }

        .immersive-active #main-content {
            max-width: 100vw;
            padding: 1rem;
        }

        /* Landscape specific for Immersive Mode */
        @media (orientation: landscape) {
            .immersive-active #layout-wrapper {
                flex-direction: row !important;
                align-items: flex-start;
                justify-content: center;
                gap: 3rem;
            }
            .immersive-active #word-bank-container {
                width: 250px;
                max-height: 80vh;
                overflow-y: auto;
            }
        }

        .btn-basic {
            border: 2px solid #000;
            padding: 0.4rem 0.8rem;
            font-weight: 600;
            transition: all 0.2s;
            text-transform: uppercase;
            font-size: 0.7rem;
            letter-spacing: 0.05em;
        }
        .btn-basic:hover {
            background: #000;
            color: #fff;
        }
        .btn-basic:disabled {
            opacity: 0.3;
            cursor: not-allowed;
        }

        #loading-overlay {
            position: fixed;
            inset: 0;
            background: rgba(255,255,255,0.8);
            display: none;
            align-items: center;
            justify-content: center;
            z-index: 100;
            font-weight: bold;
            text-transform: uppercase;
        }
    </style>
</head>
<body class="p-4">

    <div id="loading-overlay">Generating Puzzle...</div>

    <div id="main-content" class="max-w-5xl mx-auto">
        <header class="mb-8 border-b-2 border-black pb-4 flex justify-between items-end">
            <div>
                <h1 class="text-3xl font-bold uppercase tracking-tighter">Word Search Machine</h1>
                <p id="current-theme-display" class="text-sm opacity-60 italic">Manual Mode</p>
            </div>
            <div class="flex gap-2">
                <button id="ai-generate-top" class="btn-basic">AI Random</button>
                <button id="toggle-immersive" class="btn-basic">Enter Immersive</button>
            </div>
        </header>

        <div id="layout-wrapper" class="flex flex-col gap-8">
            <!-- Left Panel: Configuration (Hidden in immersive) -->
            <div id="config-panel" class="w-full lg:w-80 space-y-4">
                <div class="border-2 border-black p-4">
                    <h2 class="font-bold uppercase mb-2 text-sm">Custom Input</h2>
                    <textarea id="word-input" class="w-full p-2 border border-black text-sm h-32 outline-none font-mono" placeholder="Words...">GEMINI, CANVAS, ALGORITHM, INTERFACE, DESIGN, PUZZLE, CODING, REACT, HTML, CSS, JAVASCRIPT, LOGIC</textarea>
                    <input type="text" id="hidden-message" class="w-full mt-2 p-2 border border-black text-sm outline-none font-mono" placeholder="Secret message...">
                    <button id="generate-btn" class="w-full mt-4 bg-black text-white py-2 font-bold uppercase text-sm">Generate Manual</button>
                </div>
            </div>

            <!-- Puzzle Area -->
            <div id="puzzle-area" class="flex-1 flex flex-col items-center">
                <div class="relative">
                    <div id="grid-root" class="grid-container"></div>
                    <div id="selection-layer"></div>
                </div>

                <!-- Word Bank -->
                <div id="word-bank-container" class="mt-8 w-full max-w-xl">
                    <h2 class="font-bold uppercase mb-4 border-b border-black text-sm flex justify-between">
                        <span id="bank-label">Word Bank</span>
                        <div class="flex gap-4">
                            <button id="ai-generate-immersive" class="hidden text-[10px] underline uppercase tracking-tighter">Random AI Puzzle</button>
                            <button id="exit-immersive" class="hidden text-[10px] underline uppercase tracking-tighter">Exit Immersive</button>
                        </div>
                    </h2>
                    <div id="word-list" class="flex flex-wrap gap-x-6 gap-y-2 font-mono text-sm uppercase"></div>
                </div>

                <div id="game-status" class="mt-8">
                    <div id="completion-msg" class="hidden text-center border-2 border-black p-4 bg-yellow-50">
                        <span class="font-bold block uppercase">Puzzle Solved</span>
                        <div id="revealed-secret" class="mt-1 font-mono tracking-widest text-lg border-t border-black pt-2"></div>
                    </div>
                </div>
            </div>
        </div>
    </div>

    <script>
        const apiKey = "AIzaSyBJrI9lOXJNkkDexHUhFly23RVaZJgzSR0"; // Environment provided key

        class WordSearch {
            constructor(words, hiddenMessage = "", size = 15) {
                this.words = words.map(w => w.toUpperCase().replace(/[^A-Z]/g, '')).filter(w => w.length > 1);
                this.hiddenMessage = hiddenMessage.toUpperCase().replace(/[^A-Z]/g, '');
                this.size = size;
                this.grid = Array(size).fill().map(() => Array(size).fill(null));
                this.placedWords = [];
                this.directions = [[0, 1], [0, -1], [1, 0], [-1, 0], [1, 1], [1, -1], [-1, 1], [-1, -1]];
            }

            generate() {
                const sortedWords = [...this.words].sort((a, b) => b.length - a.length);
                for (const word of sortedWords) { this.placeWord(word); }
                this.fillEmpty();
                return { grid: this.grid, placed: this.placedWords };
            }

            placeWord(word) {
                let bestPos = null;
                let maxOverlap = -1;
                for (let i = 0; i < 250; i++) {
                    const dir = this.directions[Math.floor(Math.random() * this.directions.length)];
                    const r = Math.floor(Math.random() * this.size);
                    const c = Math.floor(Math.random() * this.size);
                    const overlap = this.checkPlacement(word, r, c, dir);
                    if (overlap > maxOverlap) {
                        maxOverlap = overlap;
                        bestPos = { r, c, dir };
                    }
                    if (overlap > 1) break;
                }
                if (bestPos && maxOverlap !== -1) {
                    this.applyPlacement(word, bestPos.r, bestPos.c, bestPos.dir);
                    this.placedWords.push({
                        word,
                        start: [bestPos.r, bestPos.c],
                        end: [
                            bestPos.r + (word.length - 1) * bestPos.dir[0],
                            bestPos.c + (word.length - 1) * bestPos.dir[1]
                        ]
                    });
                }
            }

            checkPlacement(word, r, c, dir) {
                let overlap = 0;
                for (let i = 0; i < word.length; i++) {
                    const nr = r + i * dir[0], nc = c + i * dir[1];
                    if (nr < 0 || nr >= this.size || nc < 0 || nc >= this.size) return -1;
                    const current = this.grid[nr][nc];
                    if (current !== null && current !== word[i]) return -1;
                    if (current === word[i]) overlap++;
                }
                return overlap;
            }

            applyPlacement(word, r, c, dir) {
                for (let i = 0; i < word.length; i++) {
                    this.grid[r + i * dir[0]][c + i * dir[1]] = word[i];
                }
            }

            fillEmpty() {
                const pool = this.words.join('') || "ABCDEFGHIJKLMNOPQRSTUVWXYZ";
                let secretIdx = 0;
                for (let r = 0; r < this.size; r++) {
                    for (let c = 0; c < this.size; c++) {
                        if (this.grid[r][c] === null) {
                            if (secretIdx < this.hiddenMessage.length) {
                                this.grid[r][c] = this.hiddenMessage[secretIdx++];
                            } else {
                                this.grid[r][c] = pool[Math.floor(Math.random() * pool.length)];
                            }
                        }
                    }
                }
            }
        }

        // --- AI Logic ---
        async function fetchAIPuzzle() {
            const loading = document.getElementById('loading-overlay');
            loading.style.display = 'flex';
            
            const systemPrompt = "You are a Word Search architect. Create a high-quality word search theme with 10-15 themed words and a relevant secret message. Respond ONLY in valid JSON.";
            const userPrompt = "Generate a random interesting theme (e.g. Victorian Exploration, Microscopic Life, Retro Computing, French Pastries). Provide a 'theme' string, an 'words' array of strings (max 12 letters each), and a 'secretMessage' string (max 20 chars).";

            try {
                const response = await fetch(`https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-09-2025:generateContent?key=${apiKey}`, {
                    method: 'POST',
                    headers: { 'Content-Type': 'application/json' },
                    body: JSON.stringify({
                        contents: [{ parts: [{ text: userPrompt }] }],
                        systemInstruction: { parts: [{ text: systemPrompt }] },
                        generationConfig: { 
                            responseMimeType: "application/json",
                            responseSchema: {
                                type: "OBJECT",
                                properties: {
                                    theme: { type: "STRING" },
                                    words: { type: "ARRAY", items: { type: "STRING" } },
                                    secretMessage: { type: "STRING" }
                                }
                            }
                        }
                    })
                });

                const data = await response.json();
                const result = JSON.parse(data.candidates[0].content.parts[0].text);
                
                // Update inputs for visual confirmation and game state
                document.getElementById('word-input').value = result.words.join(', ');
                document.getElementById('hidden-message').value = result.secretMessage;
                document.getElementById('current-theme-display').textContent = `Theme: ${result.theme}`;
                
                initGame();
            } catch (error) {
                console.error("AI Error:", error);
            } finally {
                loading.style.display = 'none';
            }
        }

        // --- UI Logic ---
        let currentGame = null, isSelecting = false, selectionStart = null, foundWords = new Set();
        const gridRoot = document.getElementById('grid-root'), selectionLayer = document.getElementById('selection-layer');
        const wordListEl = document.getElementById('word-list'), wordInput = document.getElementById('word-input');
        const hiddenInput = document.getElementById('hidden-message'), generateBtn = document.getElementById('generate-btn');
        const toggleImmersive = document.getElementById('toggle-immersive'), exitImmersive = document.getElementById('exit-immersive');
        const aiTop = document.getElementById('ai-generate-top'), aiImm = document.getElementById('ai-generate-immersive');

        function initGame() {
            const words = wordInput.value.split(/,|\n/).map(s => s.trim()).filter(s => s);
            const engine = new WordSearch(words, hiddenInput.value, 15);
            currentGame = engine.generate();
            foundWords.clear();
            renderGrid(currentGame.grid);
            renderWordList(currentGame.placed);
            document.getElementById('completion-msg').classList.add('hidden');
        }

        function renderGrid(grid) {
            gridRoot.innerHTML = ''; selectionLayer.innerHTML = '';
            gridRoot.style.gridTemplateColumns = `repeat(${grid.length}, 1fr)`;
            grid.forEach((row, r) => {
                row.forEach((char, c) => {
                    const cell = document.createElement('div');
                    cell.className = 'cell'; cell.textContent = char;
                    cell.dataset.r = r; cell.dataset.c = c;
                    cell.addEventListener('mousedown', () => startSelection(r, c));
                    cell.addEventListener('mouseenter', () => updateSelection(r, c));
                    cell.addEventListener('touchstart', (e) => { e.preventDefault(); startSelection(r, c); });
                    gridRoot.appendChild(cell);
                });
            });
        }

        function startSelection(r, c) { isSelecting = true; selectionStart = { r, c }; }

        function updateSelection(r, c) {
            if (!isSelecting) return;
            const dr = r - selectionStart.r, dc = c - selectionStart.c;
            if (dr !== 0 && dc !== 0 && Math.abs(dr) !== Math.abs(dc)) return;
            drawSelectionLine(selectionStart.r, selectionStart.c, r, c);
        }

        function drawSelectionLine(r1, c1, r2, c2, isFound = false) {
            const id = isFound ? `f-${r1}-${c1}-${r2}-${c2}` : 'active-sel';
            let line = document.getElementById(id);
            if (!line) {
                line = document.createElement('div');
                line.id = id;
                line.className = isFound ? 'found-word-line' : 'current-selection-line';
                selectionLayer.appendChild(line);
            }
            const c1El = document.querySelector(`[data-r="${r1}"][data-c="${c1}"]`);
            const c2El = document.querySelector(`[data-r="${r2}"][data-c="${c2}"]`);
            if (!c1El || !c2El) return;
            const r1Rect = c1El.getBoundingClientRect(), r2Rect = c2El.getBoundingClientRect();
            const root = gridRoot.getBoundingClientRect();
            const x1 = r1Rect.left + r1Rect.width/2 - root.left, y1 = r1Rect.top + r1Rect.height/2 - root.top;
            const x2 = r2Rect.left + r2Rect.width/2 - root.left, y2 = r2Rect.top + r2Rect.height/2 - root.top;
            const d = Math.sqrt((x2-x1)**2 + (y2-y1)**2), a = Math.atan2(y2-y1, x2-x1)*180/Math.PI;
            line.style.width = `${d + r1Rect.width * 0.8}px`;
            line.style.height = `${r1Rect.height * 0.8}px`;
            line.style.left = `${x1}px`; line.style.top = `${y1}px`;
            line.style.transform = `translate(-50%, -50%) rotate(${a}deg)`;
        }

        function endSelection() {
            if (!isSelecting) return;
            isSelecting = false;
            const active = document.getElementById('active-sel');
            if (!active) return;
            const endEl = document.elementFromPoint(event.clientX || (event.changedTouches?.[0].clientX), event.clientY || (event.changedTouches?.[0].clientY));
            if (endEl?.dataset?.r) checkMatch(selectionStart.r, selectionStart.c, parseInt(endEl.dataset.r), parseInt(endEl.dataset.c));
            active.remove();
        }

        function checkMatch(r1, c1, r2, c2) {
            const match = currentGame.placed.find(p => (p.start[0]===r1 && p.start[1]===c1 && p.end[0]===r2 && p.end[1]===c2) || (p.start[0]===r2 && p.start[1]===c2 && p.end[0]===r1 && p.end[1]===c1));
            if (match && !foundWords.has(match.word)) {
                foundWords.add(match.word);
                drawSelectionLine(r1, c1, r2, c2, true);
                document.getElementById(`word-${match.word}`).classList.add('found');
                if (foundWords.size === currentGame.placed.length) showWin();
            }
        }

        function renderWordList(placed) {
            wordListEl.innerHTML = '';
            placed.forEach(p => {
                const s = document.createElement('span');
                s.className = 'word-item'; s.textContent = p.word; s.id = `word-${p.word}`;
                wordListEl.appendChild(s);
            });
        }

        function showWin() {
            document.getElementById('completion-msg').classList.remove('hidden');
            if (currentGame.hiddenMessage) {
                document.getElementById('revealed-secret').textContent = currentGame.hiddenMessage;
            }
        }

        // Event Handlers
        generateBtn.onclick = initGame;
        aiTop.onclick = fetchAIPuzzle;
        aiImm.onclick = fetchAIPuzzle;
        window.onmouseup = endSelection;
        window.ontouchend = endSelection;
        gridRoot.ontouchmove = (e) => {
            const t = e.touches[0];
            const el = document.elementFromPoint(t.clientX, t.clientY);
            if (el?.dataset?.r) updateSelection(parseInt(el.dataset.r), parseInt(el.dataset.c));
        };

        toggleImmersive.onclick = () => {
            document.body.classList.add('immersive-active');
            exitImmersive.classList.remove('hidden');
            aiImm.classList.remove('hidden');
        };
        exitImmersive.onclick = () => {
            document.body.classList.remove('immersive-active');
            exitImmersive.classList.add('hidden');
            aiImm.classList.add('hidden');
        };

        window.onload = initGame;
    </script>
</body>
</html>

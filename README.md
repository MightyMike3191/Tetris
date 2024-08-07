<!DOCTYPE html>
<html lang="de">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Tetris</title>
    <style>
        body {
            display: flex;
            flex-direction: column;
            align-items: center;
            height: 100vh;
            background: #222;
            margin: 0;
            font-family: Arial, sans-serif;
            color: #fff;
            overflow: hidden; /* Verhindert Scrollbalken */
        }
        canvas {
            border: 1px solid #fff;
            background: #000;
        }
        .info, .menu, .start-container {
            margin-bottom: 20px;
        }
        .info p, .menu button, .start-container button {
            font-size: 20px;
            margin: 5px 0;
        }
        .menu {
            display: flex;
            gap: 10px;
        }
        #game-menu {
            display: none;
        }
        #game-menu button {
            font-size: 16px;
            margin: 5px;
        }
        .start-container {
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            position: absolute;
            top: 50%;
            left: 50%;
            transform: translate(-50%, -50%);
            text-align: center;
            z-index: 10;
        }
        #tetris-logo {
            font-size: 100px;
            color: rgba(255, 255, 255, 0.5);
            animation: spin 3s linear infinite;
        }
        @keyframes spin {
            0% { transform: rotate(0deg); }
            100% { transform: rotate(360deg); }
        }
    </style>
</head>
<body>
    <div class="start-container" id="start-screen">
        <div id="tetris-logo">TETRIS</div>
        <button onclick="startGame()">Spiel Starten</button>
    </div>
    <div class="info">
        <p id="score">Score: 0</p>
        <p id="time">Zeit: 00:00</p>
    </div>
    <canvas id="tetris" width="300" height="600"></canvas>
    <div class="menu">
        <button onclick="showMenu()">Menü</button>
    </div>
    <div id="game-menu">
        <button onclick="showHighscores()">Highscores anzeigen</button>
        <button onclick="saveGame()">Spiel speichern</button>
        <button onclick="loadGame()">Spiel laden</button>
        <button onclick="resetGame()">Spiel zurücksetzen</button>
        <button onclick="hideMenu()">Menü schließen</button>
    </div>
    <script>
        const canvas = document.getElementById('tetris');
        const context = canvas.getContext('2d');
        const rows = 20;
        const cols = 10;
        const blockSize = 30;
        let score = 0;
        let startTime = Date.now();
        let gameInterval;
        let isGameRunning = false;
        let speed = 1000; // Initiale Geschwindigkeit (langsamer)
        let speedIncrement = 0.95; // Wie schnell die Geschwindigkeit erhöht wird

        const tetrominoes = [
            [[1, 1, 1, 1]], // I
            [[1, 1, 1], [0, 1, 0]], // T
            [[1, 1], [1, 1]], // O
            [[1, 1, 0], [0, 1, 1]], // S
            [[0, 1, 1], [1, 1, 0]], // Z
            [[1, 1, 1], [1, 0, 0]], // L
            [[1, 1, 1], [0, 0, 1]]  // J
        ];

        const colors = ['cyan', 'purple', 'yellow', 'green', 'red', 'orange', 'blue'];

        let board = Array.from({ length: rows }, () => Array(cols).fill(0));
        let currentTetromino, currentPos, currentColor;

        function drawBlock(x, y, color) {
            context.fillStyle = color;
            context.fillRect(x * blockSize, y * blockSize, blockSize, blockSize);
            context.strokeRect(x * blockSize, y * blockSize, blockSize, blockSize);
        }

        function drawBoard() {
            context.clearRect(0, 0, canvas.width, canvas.height);
            for (let y = 0; y < rows; y++) {
                for (let x = 0; x < cols; x++) {
                    if (board[y][x]) {
                        drawBlock(x, y, colors[board[y][x] - 1]);
                    }
                }
            }
        }

        function drawTetromino() {
            const [shape] = currentTetromino;
            for (let y = 0; y < shape.length; y++) {
                for (let x = 0; x < shape[y].length; x++) {
                    if (shape[y][x]) {
                        drawBlock(currentPos.x + x, currentPos.y + y, currentColor);
                    }
                }
            }
        }

        function collide(x, y, shape) {
            for (let row = 0; row < shape.length; row++) {
                for (let col = 0; col < shape[row].length; col++) {
                    if (shape[row][col] && (board[y + row] && board[y + row][x + col]) !== 0) {
                        return true;
                    }
                }
            }
            return false;
        }

        function mergeTetromino() {
            const [shape] = currentTetromino;
            for (let row = 0; row < shape.length; row++) {
                for (let col = 0; col < shape[row].length; col++) {
                    if (shape[row][col]) {
                        board[currentPos.y + row][currentPos.x + col] = colors.indexOf(currentColor) + 1;
                    }
                }
            }
        }

        function clearLines() {
            let linesCleared = 0;
            outer: for (let row = rows - 1; row >= 0; row--) {
                for (let col = 0; col < cols; col++) {
                    if (!board[row][col]) {
                        continue outer;
                    }
                }
                board.splice(row, 1);
                board.unshift(Array(cols).fill(0));
                linesCleared++;
            }
            if (linesCleared > 0) {
                score += linesCleared * 100;
                updateScore();
                speed *= speedIncrement; // Geschwindigkeit erhöhen
                clearInterval(gameInterval);
                gameInterval = setInterval(update, speed);
            }
        }

        function rotateTetromino() {
            const [shape] = currentTetromino;
            const newShape = shape[0].map((_, i) => shape.map(row => row[i]).reverse());
            if (!collide(currentPos.x, currentPos.y, newShape)) {
                currentTetromino[0] = newShape;
            }
        }

        function moveTetromino(dx, dy) {
            if (!collide(currentPos.x + dx, currentPos.y + dy, currentTetromino[0])) {
                currentPos.x += dx;
                currentPos.y += dy;
            } else if (dy > 0) {
                mergeTetromino();
                clearLines();
                newTetromino();
            }
        }

        function newTetromino() {
            const index = Math.floor(Math.random() * tetrominoes.length);
            currentTetromino = [tetrominoes[index], [0, 0]];
            currentPos = { x: Math.floor(cols / 2) - 1, y: 0 };
            currentColor = colors[index];
            if (collide(currentPos.x, currentPos.y, currentTetromino[0])) {
                clearInterval(gameInterval);
                isGameRunning = false;
                updateHighscores();
                alert('Game Over');
            }
        }

        function update() {
            moveTetromino(0, 1);
            drawBoard();
            drawTetromino();
            updateTime();
        }

        function updateScore() {
            document.getElementById('score').textContent = `Score: ${score}`;
        }

        function updateTime() {
            const elapsed = Math.floor((Date.now() - startTime) / 1000);
            const minutes = Math.floor(elapsed / 60);
            const seconds = elapsed % 60;
            document.getElementById('time').textContent = `Zeit: ${String(minutes).padStart(2, '0')}:${String(seconds).padStart(2, '0')}`;
        }

        function keyHandler(e) {
            if (!isGameRunning) return;
            switch (e.key) {
                case 'ArrowLeft':
                    moveTetromino(-1, 0);
                    break;
                case 'ArrowRight':
                    moveTetromino(1, 0);
                    break;
                case 'ArrowDown':
                    moveTetromino(0, 1);
                    break;
                case 'ArrowUp':
                    rotateTetromino();
                    break;
            }
            drawBoard();
            drawTetromino();
        }

        function showMenu() {
            document.getElementById('game-menu').style.display = 'block';
        }

        function hideMenu() {
            document.getElementById('game-menu').style.display = 'none';
        }

        function showHighscores() {
            const highscores = JSON.parse(localStorage.getItem('highscores')) || [];
            alert('Highscores:\n' + highscores.map((score, index) => `${index + 1}. ${score}`).join('\n'));
        }

        function updateHighscores() {
            const highscores = JSON.parse(localStorage.getItem('highscores')) || [];
            highscores.push(score);
            highscores.sort((a, b) => b - a);
            if (highscores.length > 10) highscores.length = 10; // Keep top 10 scores
            localStorage.setItem('highscores', JSON.stringify(highscores));
        }

        function saveGame() {
            if (!isGameRunning) {
                alert('Kein laufendes Spiel zum Speichern.');
                return;
            }
            const gameState = {
                board,
                currentTetromino,
                currentPos,
                currentColor,
                score,
                startTime
            };
            localStorage.setItem('savedGame', JSON.stringify(gameState));
            alert('Spiel gespeichert.');
        }

        function loadGame() {
            const savedGame = localStorage.getItem('savedGame');
            if (savedGame) {
                const gameState = JSON.parse(savedGame);
                board = gameState.board;
                currentTetromino = gameState.currentTetromino;
                currentPos = gameState.currentPos;
                currentColor = gameState.currentColor;
                score = gameState.score;
                startTime = gameState.startTime;
                document.getElementById('score').textContent = `Score: ${score}`;
                gameInterval = setInterval(update, speed);
                isGameRunning = true;
                alert('Spiel geladen.');
            } else {
                alert('Kein gespeichertes Spiel gefunden.');
            }
        }

        function resetGame() {
            clearInterval(gameInterval);
            board = Array.from({ length: rows }, () => Array(cols).fill(0));
            score = 0;
            startTime = Date.now();
            speed = 1000; // Zurücksetzen der Geschwindigkeit
            document.getElementById('score').textContent = `Score: ${score}`;
            document.getElementById('time').textContent = `Zeit: 00:00`;
            isGameRunning = false;
            hideMenu();
        }

        function startGame() {
            if (isGameRunning) return;
            document.getElementById('start-screen').style.display = 'none';
            newTetromino();
            gameInterval = setInterval(update, speed);
            isGameRunning = true;
        }

        function init() {
            document.addEventListener('keydown', keyHandler);
            document.getElementById('start-screen').style.display = 'flex';
        }

        init();
    </script>
</body>
</html>

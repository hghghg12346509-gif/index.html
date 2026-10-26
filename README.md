<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Догони кругляшок</title>
    <style>
        body {
            margin: 0;
            padding: 0;
            height: 100vh;
            background-color: #1a1a2e;
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            overflow: hidden;
            user-select: none;
            cursor: crosshair;
        }

        #game-area {
            position: relative;
            width: 100vw;
            height: 100vh;
        }

        #circle {
            position: absolute;
            width: 200px;
            height: 200px;
            background-color: rgba(0, 255, 204, 0.15);
            border: 3px solid #00ffcc;
            border-radius: 50%;
            box-shadow: 0 0 20px #00ffcc, inset 0 0 20px #00ffcc;
            transform: translate(-50%, -50%);
            pointer-events: none;
            transition: width 0.3s, height 0.3s;
        }

        /* Линия, показывающая начальное направление */
        .trajectory-line {
            position: absolute;
            height: 6px;
            background-color: #ffffff;
            box-shadow: 0 0 15px #ffffff;
            border-radius: 3px;
            transform-origin: 0 50%;
            pointer-events: none;
            z-index: 15;
            opacity: 0.9;
            animation: flash 0.4s infinite alternate;
        }

        @keyframes flash {
            from { opacity: 0.4; }
            to { opacity: 1; }
        }

        #score-display {
            position: absolute;
            top: 20px;
            left: 20px;
            color: #fff;
            font-size: 24px;
            font-weight: bold;
            pointer-events: none;
            text-shadow: 0 0 10px rgba(255, 255, 255, 0.5);
            z-index: 5;
        }

        .overlay {
            position: absolute;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            background: rgba(26, 26, 46, 0.95);
            display: flex;
            flex-direction: column;
            justify-content: center;
            align-items: center;
            color: white;
            text-align: center;
            z-index: 10;
        }

        .overlay h1 {
            font-size: 48px;
            margin-bottom: 10px;
            color: #00ffcc;
            text-shadow: 0 0 20px #00ffcc;
        }

        .overlay p {
            font-size: 20px;
            margin-bottom: 30px;
            max-width: 600px;
            line-height: 1.5;
        }

        .overlay button {
            padding: 15px 40px;
            font-size: 24px;
            font-weight: bold;
            color: #1a1a2e;
            background-color: #00ffcc;
            border: none;
            border-radius: 30px;
            cursor: pointer;
            transition: transform 0.2s, box-shadow 0.2s;
            box-shadow: 0 0 15px #00ffcc;
        }

        .overlay button:hover {
            transform: scale(1.1);
            box-shadow: 0 0 30px #00ffcc;
        }

        #countdown {
            position: absolute;
            top: 50%;
            left: 50%;
            transform: translate(-50%, -50%);
            font-size: 150px;
            font-weight: bold;
            color: #fff;
            text-shadow: 0 0 30px #00ffcc;
            z-index: 20;
            pointer-events: none;
            animation: pulse 1s infinite;
        }

        @keyframes pulse {
            0% { transform: translate(-50%, -50%) scale(1); opacity: 1; }
            50% { transform: translate(-50%, -50%) scale(1.2); opacity: 0.7; }
            100% { transform: translate(-50%, -50%) scale(1); opacity: 1; }
        }

        .hidden {
            display: none !important;
        }
    </style>
</head>
<body>

    <div id="game-area">
        <div id="score-display">Время: 0.0с</div>
        <div id="countdown" class="hidden">3</div>
        <div id="circle" class="hidden"></div>

        <div id="start-screen" class="overlay">
            <h1>Догони кругляшок</h1>
            <p>Держите курсор внутри круга.<br>После отсчета у вас будет 1.5 секунды, чтобы увидеть, <b>куда он полетит</b>.<br>С каждой секундой круг становится меньше и быстрее!</p>
            <button id="start-btn">Начать игру</button>
        </div>

        <div id="game-over-screen" class="overlay hidden">
            <h1 style="color: #ff3366; text-shadow: 0 0 20px #ff3366;">Вы проиграли!</h1>
            <p>Вы продержались: <span id="final-score">0.0</span> секунд</p>
            <button id="restart-btn">Попробовать снова</button>
        </div>
    </div>

    <script>
        const circle = document.getElementById('circle');
        const scoreDisplay = document.getElementById('score-display');
        const startScreen = document.getElementById('start-screen');
        const gameOverScreen = document.getElementById('game-over-screen');
        const countdownEl = document.getElementById('countdown');
        const startBtn = document.getElementById('start-btn');
        const restartBtn = document.getElementById('restart-btn');
        const finalScoreSpan = document.getElementById('final-score');
        const gameArea = document.getElementById('game-area');

        let isPlaying = false;
        let isPreview = false; // Флаг фазы показа направления
        let previewTimer = null;
        
        let mouseX = window.innerWidth / 2; 
        let mouseY = window.innerHeight / 2;
        
        let circleX = 0, circleY = 0;
        let radius = 100;
        let speedX = 5;
        let speedY = 5;
        
        let startTime = 0;
        let currentScore = 0;
        let animationFrameId;

        window.addEventListener('mousemove', (e) => {
            mouseX = e.clientX;
            mouseY = e.clientY;
        });

        function prepareGame() {
            isPlaying = false;
            isPreview = false;
            clearTimeout(previewTimer);
            cancelAnimationFrame(animationFrameId);
            
            // Удаляем старую линию, если она осталась
            const oldLine = document.querySelector('.trajectory-line');
            if (oldLine) oldLine.remove();

            startScreen.classList.add('hidden');
            gameOverScreen.classList.add('hidden');
            circle.classList.add('hidden');
            scoreDisplay.textContent = 'Время: 0.0с';
            
            countdownEl.classList.remove('hidden');
            let count = 3;
            countdownEl.textContent = count;
            
            const timer = setInterval(() => {
                count--;
                if (count > 0) {
                    countdownEl.textContent = count;
                } else {
                    clearInterval(timer);
                    countdownEl.classList.add('hidden');
                    startActualGame();
                }
            }, 1000);
        }

        function startActualGame() {
            isPlaying = true;
            isPreview = true; // Включаем фазу предпросмотра
            radius = 100;
            currentScore = 0;
            circle.lastSizeUpdate = 0;

            // Случайное направление (от 0 до 360 градусов)
            const randomAngle = Math.random() * Math.PI * 2;
            const startSpeed = 5;
            speedX = Math.cos(randomAngle) * startSpeed;
            speedY = Math.sin(randomAngle) * startSpeed;

            // Круг появляется под мышкой
            circleX = mouseX;
            circleY = mouseY;

            // Не даем кругу выйти за края
            if (circleX - radius < 0) circleX = radius;
            if (circleX + radius > window.innerWidth) circleX = window.innerWidth - radius;
            if (circleY - radius < 0) circleY = radius;
            if (circleY + radius > window.innerHeight) circleY = window.innerHeight - radius;

            circle.style.width = (radius * 2) + 'px';
            circle.style.height = (radius * 2) + 'px';
            circle.style.left = circleX + 'px';
            circle.style.top = circleY + 'px';
            circle.classList.remove('hidden');

            // Рисуем линию направления
            const line = document.createElement('div');
            line.classList.add('trajectory-line');
            line.style.left = circleX + 'px';
            line.style.top = (circleY - 3) + 'px'; // Выравниваем по центру (высота линии 6px)
            line.style.width = (radius - 10) + 'px'; // Чуть короче радиуса
            line.style.transform = `rotate(${randomAngle}rad)`;
            gameArea.appendChild(line);

            // Через 1.5 секунды убираем линию и запускаем движение
            previewTimer = setTimeout(() => {
                isPreview = false;
                line.remove();
                startTime = Date.now(); // Таймер игры стартует только сейчас
                updateGame();
            }, 1500); 
        }

        function gameOver() {
            isPlaying = false;
            cancelAnimationFrame(animationFrameId);
            circle.classList.add('hidden');
            gameOverScreen.classList.remove('hidden');
            finalScoreSpan.textContent = currentScore.toFixed(1);
        }

        function updateGame() {
            if (!isPlaying) return;

            // Если идет предпросмотр, круг стоит на месте
            if (!isPreview) {
                circleX += speedX;
                circleY += speedY;

                // Отскок от стен
                if (circleX - radius <= 0 || circleX + radius >= window.innerWidth) {
                    speedX *= -1;
                }
                if (circleY - radius <= 0 || circleY + radius >= window.innerHeight) {
                    speedY *= -1;
                }

                // Проверка на проигрыш
                const dist = Math.sqrt(Math.pow(mouseX - circleX, 2) + Math.pow(mouseY - circleY, 2));
                if (dist > radius) {
                    gameOver();
                    return;
                }

                // Таймер и счет
                currentScore = (Date.now() - startTime) / 1000;
                scoreDisplay.textContent = `Время: ${currentScore.toFixed(1)}с`;

                // Усложнение
                if (Math.floor(currentScore) > 0 && Math.floor(currentScore) % 5 === 0 && Math.floor(currentScore) > circle.lastSizeUpdate) {
                    circle.lastSizeUpdate = Math.floor(currentScore);
                    
                    if (radius > 20) {
                        radius -= 15;
                        circle.style.width = (radius * 2) + 'px';
                        circle.style.height = (radius * 2) + 'px';
                    }
                    
                    if (Math.abs(speedX) < 20) {
                        speedX *= 1.15;
                        speedY *= 1.15;
                    }
                }
            }

            // Обновляем позицию круга на экране (даже если он стоит)
            circle.style.left = circleX + 'px';
            circle.style.top = circleY + 'px';

            animationFrameId = requestAnimationFrame(updateGame);
        }

        startBtn.addEventListener('click', prepareGame);
        restartBtn.addEventListener('click', prepareGame);
    </script>
</body>
</html>

# Pet2<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
    <title>🐾 Pet & Co</title>
    <script src="https://telegram.org/js/telegram-web-app.js"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/lottie-web/5.12.2/lottie.min.js"></script>
    <style>
        body {
            margin: 0;
            padding: 16px;
            background: #1a2a1f;
            font-family: 'Segoe UI', Roboto, system-ui, sans-serif;
            color: #e2e8d5;
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            min-height: 100vh;
            box-sizing: border-box;
            text-align: center;
        }
        .game-container {
            max-width: 400px;
            width: 100%;
            background: #2d3e2a;
            border-radius: 32px;
            padding: 24px 20px;
            box-shadow: 0 12px 0 #1b2619;
            border: 1px solid #5b8c4e;
            position: relative;
        }
        #pet-container {
            width: 200px;
            height: 200px;
            margin: 10px auto;
        }
        h1 {
            margin: 0 0 8px;
            font-weight: 600;
            font-size: 28px;
            color: #f3f0d9;
        }
        .status-card {
            background: #1e2e1b;
            border-radius: 20px;
            padding: 16px;
            margin: 20px 0;
            border-left: 6px solid #f5b342;
        }
        .user-info {
            display: flex;
            align-items: center;
            justify-content: center;
            gap: 8px;
            margin-bottom: 16px;
            background: #3b5535;
            padding: 8px 16px;
            border-radius: 60px;
        }
        button {
            background: #f5b342;
            border: none;
            color: #1e2e1b;
            font-weight: bold;
            font-size: 22px;
            padding: 18px 24px;
            border-radius: 60px;
            width: 100%;
            box-shadow: 0 6px 0 #b87e1e;
            transition: all 0.05s ease-in;
            cursor: pointer;
            border: 1px solid #ffe49e;
            margin-top: 10px;
        }
        button:active {
            transform: translateY(4px);
            box-shadow: 0 2px 0 #b87e1e;
        }
        button:disabled {
            opacity: 0.5;
            transform: none;
            box-shadow: 0 6px 0 #b87e1e;
            cursor: not-allowed;
            background: #aab5a3;
        }
        .footer-note {
            margin-top: 16px;
            font-size: 14px;
            color: #98ab8f;
        }
        .streak {
            font-size: 20px;
            font-weight: bold;
            color: #ffd966;
        }

        /* Облачко эмоций */
        #bubble {
            position: absolute;
            top: 60px;
            left: 50%;
            transform: translateX(-50%);
            background: white;
            border-radius: 20px;
            padding: 8px 16px;
            font-size: 22px;
            box-shadow: 0 4px 8px rgba(0,0,0,0.2);
            opacity: 0;
            transition: opacity 0.3s ease;
            pointer-events: none;
            z-index: 10;
        }
        #bubble::after {
            content: '';
            position: absolute;
            bottom: -10px;
            left: 50%;
            transform: translateX(-50%);
            width: 0;
            height: 0;
            border-left: 10px solid transparent;
            border-right: 10px solid transparent;
            border-top: 12px solid white;
        }
    </style>
</head>
<body>
<div class="game-container">
    <h1>🐾 Pet</h1>
    <div class="user-info" id="userInfo">Загрузка профиля...</div>
    
    <div id="pet-container"></div>
    <div id="bubble"></div> <!-- облачко эмоций -->
    
    <div class="status-card">
        <div>❤️ Сытость: <span id="hungerValue">•••</span></div>
        <div>🔥 Серия: <span class="streak" id="streakValue">0 дней</span></div>
        <div style="font-size: 14px; margin-top: 8px;" id="lastVisitText"></div>
    </div>

    <button id="feedBtn">🥬 ПОКОРМИТЬ ПИТОМЦА</button>
    <div class="footer-note">
        Вместе с @другом
    </div>
</div>

<script>
    // ========== НАСТРОЙКИ ==========
    const BOT_TOKEN = 'ТВОЙ_ТОКЕН_БОТА';          // ← ВСТАВЬ СВОЙ ТОКЕН
    const ANIMATION_PATH = 'cat.json';             // ← имя твоего файла

    const tg = window.Telegram.WebApp;
    tg.expand();
    tg.ready();
    
    const userInfoEl = document.getElementById('userInfo');
    const hungerValueEl = document.getElementById('hungerValue');
    const streakValueEl = document.getElementById('streakValue');
    const lastVisitTextEl = document.getElementById('lastVisitText');
    const feedBtn = document.getElementById('feedBtn');

    // Переменные для облачка
    let baseEmoji = '💤';
    let isTempBubble = false;
    let bubbleTimeout = null;

    // Инициализация Lottie
    let petAnimation = lottie.loadAnimation({
        container: document.getElementById('pet-container'),
        renderer: 'svg',
        loop: true,
        autoplay: true,
        path: ANIMATION_PATH
    });
    
    let petState = {
        hunger: 100,
        lastFed: Date.now(),
        streakDays: 3,
    };

    function updateUI() {
        hungerValueEl.textContent = petState.hunger + '%';
        streakValueEl.textContent = petState.streakDays + ' дн.';
        
        if (petAnimation) {
            if (petState.hunger > 70) {
                petAnimation.setSpeed(1.2);
            } else if (petState.hunger > 30) {
                petAnimation.setSpeed(0.8);
            } else {
                petAnimation.setSpeed(0.4);
            }
        }

        const lastFedDate = new Date(petState.lastFed);
        lastVisitTextEl.textContent = `🕒 Кормили: ${lastFedDate.toLocaleTimeString().slice(0,5)}`;

        // Определяем, какое сейчас базовое состояние
        if (petState.hunger <= 30) {
            baseEmoji = '🍔';
        } else {
            baseEmoji = '💤';
        }

        // Если нет временной эмоции, показываем базовую
        if (!isTempBubble) {
            showBubble(baseEmoji, 0);
        }
    }

    function showBubble(emoji, duration = 0) {
        const bubble = document.getElementById('bubble');
        if (!bubble) return;
        
        clearTimeout(bubbleTimeout);
        bubble.textContent = emoji;
        bubble.style.opacity = '1';
        
        if (duration > 0) {
            isTempBubble = true;
            bubbleTimeout = setTimeout(() => {
                isTempBubble = false;
                // Возвращаемся к базовой эмоции
                showBubble(baseEmoji, 0);
            }, duration);
        } else {
            isTempBubble = false;
        }
    }

    function feedPet() {
        if (petState.hunger >= 100) {
            alert('Питомец наелся! Давай попозже.');
            return;
        }

        petState.hunger = Math.min(100, petState.hunger + 25);
        petState.lastFed = Date.now();
        
        showBubble('💕', 1500);   // эмоция во время кормёжки
        
        // Визуальный прыжок
        const container = document.getElementById('pet-container');
        if (container) {
            container.style.transition = 'transform 0.15s cubic-bezier(0.34, 1.56, 0.64, 1)';
            container.style.transform = 'scale(1.05) translateY(-4px)';
            setTimeout(() => {
                container.style.transform = 'scale(1) translateY(0)';
            }, 180);
        }
        
        updateUI();
    }

    feedBtn.addEventListener('click', feedPet);

    setInterval(() => {
        if (petState.hunger > 0) {
            petState.hunger = Math.max(0, petState.hunger - 1);
            updateUI();
        }
    }, 30000);

    const initData = tg.initDataUnsafe;
    let currentUser = null;
    if (initData && initData.user) {
        currentUser = initData.user;
        const firstName = currentUser.first_name || 'Хозяин';
        const username = currentUser.username ? '@' + currentUser.username : '';
        userInfoEl.innerHTML = `👤 ${firstName} ${username}`;
    } else {
        userInfoEl.innerHTML = '👤 Гость (открой через бота)';
    }

    updateUI();
    tg.setHeaderColor('#2d3e2a');
    tg.setBackgroundColor('#1a2a1f');
</script>
</body>
</html>

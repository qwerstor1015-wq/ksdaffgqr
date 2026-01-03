<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Запись на прием</title>
    <script src="https://telegram.org/js/telegram-web-app.js"></script>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { font-family: Arial, sans-serif; padding: 20px; background: #f5f5f5; }
        .container { max-width: 400px; margin: 0 auto; }
        h1 { text-align: center; margin-bottom: 20px; color: #333; }
        .day { background: white; padding: 15px; margin: 10px 0; border-radius: 10px; cursor: pointer; }
        .day:hover { background: #e3f2fd; }
        .selected { background: #2196f3 !important; color: white; }
        .time { display: inline-block; padding: 10px; margin: 5px; background: #eee; border-radius: 5px; cursor: pointer; }
        .time.selected { background: #4caf50; color: white; }
        input { width: 100%; padding: 12px; margin: 10px 0; border: 1px solid #ddd; border-radius: 5px; }
        button { width: 100%; padding: 15px; background: #2196f3; color: white; border: none; border-radius: 10px; font-size: 16px; cursor: pointer; }
        button:disabled { background: #ccc; }
        .step { display: none; }
        .active { display: block; }
    </style>
</head>
<body>
    <div class="container">
        <!-- Шаг 1: Выбор дня -->
        <div id="step1" class="step active">
            <h1>📅 Выберите день</h1>
            <div id="days"></div>
            <button onclick="nextStep()" id="nextBtn1" disabled>Далее →</button>
        </div>
        
        <!-- Шаг 2: Выбор времени -->
        <div id="step2" class="step">
            <h1>🕐 Выберите время</h1>
            <div id="times"></div>
            <button onclick="prevStep()">← Назад</button>
            <button onclick="nextStep()" id="nextBtn2" disabled>Далее →</button>
        </div>
        
        <!-- Шаг 3: Ввод данных -->
        <div id="step3" class="step">
            <h1>👤 Ваши данные</h1>
            <input id="name" placeholder="Ваше имя" type="text">
            <input id="phone" placeholder="Телефон" type="tel">
            <button onclick="prevStep()">← Назад</button>
            <button onclick="book()">✅ Записаться</button>
        </div>
        
        <!-- Шаг 4: Подтверждение -->
        <div id="step4" class="step">
            <h1>✅ Готово!</h1>
            <p id="result"></p>
            <button onclick="startOver()">📅 Новая запись</button>
        </div>
        
        <!-- Админ панель -->
        <div id="admin" class="step">
            <h1>👑 Админ</h1>
            <div id="bookings"></div>
            <button onclick="startOver()">← Назад</button>
        </div>
    </div>

    <script>
        // ============ ПРОСТЫЕ ДАННЫЕ ============
        let selectedDay = null;
        let selectedTime = null;
        let bookings = JSON.parse(localStorage.getItem('bookings') || '[]');
        const tg = window.Telegram.WebApp;
        
        // ============ ЗАГРУЗКА ============
        window.onload = function() {
            tg.expand(); // Раскрываем приложение на весь экран
            generateDays(); // Показываем дни
        };
        
        // ============ ПОКАЗЫВАЕМ ДНИ НЕДЕЛИ ============
        function generateDays() {
            const daysDiv = document.getElementById('days');
            const days = ['Понедельник', 'Вторник', 'Среда', 'Четверг', 'Пятница', 'Суббота', 'Воскресенье'];
            
            days.forEach((day, index) => {
                const date = new Date();
                date.setDate(date.getDate() + index);
                const dateStr = date.toLocaleDateString('ru-RU');
                
                const dayDiv = document.createElement('div');
                dayDiv.className = 'day';
                dayDiv.innerHTML = `<strong>${day}</strong><br>${dateStr}`;
                
                dayDiv.onclick = () => {
// Снимаем выделение со всех
                    document.querySelectorAll('.day').forEach(d => d.classList.remove('selected'));
                    // Выделяем выбранный
                    dayDiv.classList.add('selected');
                    selectedDay = dateStr;
                    document.getElementById('nextBtn1').disabled = false;
                };
                
                daysDiv.appendChild(dayDiv);
            });
        }
        
        // ============ ПОКАЗЫВАЕМ ВРЕМЯ ============
        function generateTimes() {
            const timesDiv = document.getElementById('times');
            timesDiv.innerHTML = '';
            const times = ['09:00', '10:00', '11:00', '12:00', '14:00', '15:00', '16:00', '17:00'];
            
            // Получаем занятое время на этот день
            const busyTimes = bookings
                .filter(b => b.day === selectedDay && b.status !== 'отменена')
                .map(b => b.time);
            
            times.forEach(time => {
                const timeDiv = document.createElement('span');
                timeDiv.className = 'time';
                timeDiv.textContent = time;
                
                // Если время занято - серый цвет
                if (busyTimes.includes(time)) {
                    timeDiv.style.background = '#ccc';
                    timeDiv.style.cursor = 'not-allowed';
                } else {
                    timeDiv.onclick = () => {
                        document.querySelectorAll('.time').forEach(t => t.classList.remove('selected'));
                        timeDiv.classList.add('selected');
                        selectedTime = time;
                        document.getElementById('nextBtn2').disabled = false;
                    };
                }
                
                timesDiv.appendChild(timeDiv);
            });
        }
        
        // ============ НАВИГАЦИЯ ============
        function showStep(stepNumber) {
            // Скрываем все шаги
            for (let i = 1; i <= 4; i++) {
                document.getElementById('step' + i).classList.remove('active');
            }
            document.getElementById('step' + stepNumber).classList.add('active');
        }
        
        function nextStep() {
            if (document.getElementById('step1').classList.contains('active')) {
                generateTimes();
                showStep(2);
            } else if (document.getElementById('step2').classList.contains('active')) {
                showStep(3);
            }
        }
        
        function prevStep() {
            if (document.getElementById('step2').classList.contains('active')) {
                showStep(1);
            } else if (document.getElementById('step3').classList.contains('active')) {
                showStep(2);
            }
        }
        
        // ============ СОХРАНЕНИЕ ЗАПИСИ ============
        function book() {
            const name = document.getElementById('name').value;
            const phone = document.getElementById('phone').value;
            
            if (!name || !phone) {
                alert('Введите имя и телефон!');
                return;
            }
            
            // Создаем запись
            const booking = {
                id: Date.now(),
                day: selectedDay,
                time: selectedTime,
                name: name,
                phone: phone,
                status: 'новая',
                date: new Date().toLocaleString()
            };
            
            // Сохраняем в localStorage
            bookings.push(booking);
            localStorage.setItem('bookings', JSON.stringify(bookings));
            
            // Отправляем уведомление в Telegram (если бот настроен)
            sendToTelegram(booking);
            
            // Показываем результат
            document.getElementById('result').innerHTML = `
                ✅ Запись создана!<br><br>
                <strong>${name}</strong><br>
                📞 ${phone}<br>
📅 ${selectedDay}<br>
                🕐 ${selectedTime}<br><br>
                <small>Мы с вами свяжемся!</small>
            `;
            
            showStep(4);
        }
        
        // ============ ОТПРАВКА В TELEGRAM ============
        function sendToTelegram(booking) {
            // Получаем данные пользователя из Telegram Web App
            const user = tg.initDataUnsafe.user;
            const userId = user ? user.id : 'неизвестен';
            const username = user ? '@' + user.username : 'нет';
            
            // Формируем сообщение для админа
            const message = `📝 НОВАЯ ЗАПИСЬ:
            
👤 Имя: ${booking.name}
📞 Телефон: ${booking.phone}
📅 Дата: ${booking.day}
🕐 Время: ${booking.time}
🆔 Пользователь: ${username}
ID: ${userId}

✅ Для подтверждения: /confirm_${booking.id}
❌ Для отмены: /cancel_${booking.id}`;
            
            // Отправляем сообщение через бота (нужно настроить бота)
            // fetch(`https://api.telegram.org/bot7970001072:AAGyodHk7tzHskZP_Ew4lMvWSO-BPN-McqM/sendMessage?chat_id= 8073628158
          text=${encodeURIComponent(message)}`);
            
            // Или просто показываем в консоли
            console.log('Сообщение для Telegram:', message);
        }
        
        // ============ АДМИН ПАНЕЛЬ ============
        function showAdmin() {
            // Простая проверка пароля
            const password = prompt('Введите пароль админа:');
            if (password !== 'PoligamnoeVeslo41') { // Простой пароль, поменяйте!
                alert('Неверный пароль!');
                return;
            }
            
            const bookingsDiv = document.getElementById('bookings');
            bookingsDiv.innerHTML = '<h3>Все записи:</h3>';
            
            if (bookings.length === 0) {
                bookingsDiv.innerHTML += '<p>Нет записей</p>';
            } else {
                bookings.forEach(booking => {
                    bookingsDiv.innerHTML += `
                        <div class="day" style="margin: 10px 0; padding: 10px;">
                            <strong>${booking.name}</strong> (${booking.phone})<br>
                            📅 ${booking.day} в ${booking.time}<br>
                            Статус: <span id="status_${booking.id}">${booking.status}</span><br>
                            <button onclick="changeStatus(${booking.id}, 'подтверждена')">✅</button>
                            <button onclick="changeStatus(${booking.id}, 'отменена')">❌</button>
                        </div>
                    `;
                });
            }
            
            document.getElementById('admin').classList.add('active');
            document.getElementById('step1').classList.remove('active');
        }
        
        function changeStatus(id, status) {
            // Находим запись
            const booking = bookings.find(b => b.id === id);
            if (booking) {
                booking.status = status;
                localStorage.setItem('bookings', JSON.stringify(bookings));
                document.getElementById('status_' + id).textContent = status;
            }
        }
        
        // ============ ВСПОМОГАТЕЛЬНЫЕ ============
        function startOver() {
            // Сбрасываем выбор
            selectedDay = null;
            selectedTime = null;
            document.getElementById('name').value = '';
            document.getElementById('phone').value = '';
            document.querySelectorAll('.day.selected, .time.selected').forEach(el => {
                el.classList.remove('selected');
            });
            document.getElementById('nextBtn1').disabled = true;
            document.getElementById('nextBtn2').disabled = true;
            
            // Показываем первый шаг
            showStep(1);
        }
        
        // ============ КНОПКА АДМИНА В ИНТЕРФЕЙСЕ ============
        // Добавляем кнопку админа если нажать на заголовок 3 раза
        let clickCount = 0;
        document.querySelector('h1').onclick = function() {
            clickCount++;
            if (clickCount >= 3) {
showAdmin();
                clickCount = 0;
            }
        };
    </script>
</body>
</html>

import sqlite3
from telebot import TeleBot
import random
import time

# Введи свій токен бота
TOKEN = ""
bot = TeleBot(TOKEN)

# Підключення до бази даних SQLite (створюється файл players.db)
conn = sqlite3.connect('players.db', check_same_thread=False)
conn.row_factory = sqlite3.Row  # Додаємо можливість працювати з результатами запиту як з рядками

# Створення таблиці, якщо вона не існує
cursor = conn.cursor()
cursor.execute('''CREATE TABLE IF NOT EXISTS players (
                    user_id INTEGER PRIMARY KEY,
                    username TEXT,
                    score INTEGER,
                    last_action_time REAL)''')
conn.commit()  # Зберігаємо зміни

# Функція для отримання або додавання гравця
def get_or_create_player(user_id, username):
    cursor = conn.cursor()
    cursor.execute("SELECT * FROM players WHERE user_id = ?", (user_id,))
    player = cursor.fetchone()

    if player is None:
        # Якщо користувача ще немає, додаємо його в базу
        cursor.execute("INSERT INTO players (user_id, username, score, last_action_time) VALUES (?, ?, ?, ?)",
                       (user_id, username, 0, None))  # Встановлюємо last_action_time = None, бо ще не було взаємодії
        conn.commit()  # Зберігаємо зміни
        print(f"User {username} added with 0 points.")
        return {"user_id": user_id, "username": username, "score": 0, "last_action_time": None}
    else:
        # Якщо користувач вже є в базі, повертаємо його дані
        return {"user_id": player["user_id"], "username": player["username"], "score": player["score"], "last_action_time": player["last_action_time"]}

# Обробник команди /start
@bot.message_handler(commands=['start'])
def start(message):
    user_id = message.from_user.id
    username = message.from_user.username or message.from_user.first_name

    # Отримуємо або створюємо користувача
    player = get_or_create_player(user_id, username)

    # Генерація випадкового числа для додавання (від 1 до 10) та віднімання (від 1 до 5)
    random_number_add = random.randint(1, 10)
    random_number_subtract = random.randint(1, 5)
    
    # Збільшення шансів на додавання
    random_action = random.choices(['add', 'subtract'], weights=[65, 35], k=1)[0]  # 65% шанс на додавання, 35% на віднімання

    # Перевірка чи це перша взаємодія
    if player['last_action_time'] is None:
        # Перша взаємодія, бали даються без перевірки таймера
        player['score'] += random_number_add if random_action == "add" else -random_number_subtract
        player['score'] = max(player['score'], 0)  # Запобігаємо від'ємним балам
        cursor = conn.cursor()
        cursor.execute("UPDATE players SET score = ? WHERE user_id = ?", (player['score'], user_id))
        conn.commit()
        bot.send_message(message.chat.id, f"@{username}, ти отримав {random_number_add} см! Загальний розмір кия: {player['score']} см.")
        
        # Оновлюємо час останньої взаємодії
        current_time = time.time()
        cursor.execute("UPDATE players SET last_action_time = ? WHERE user_id = ?", (current_time, user_id))
        conn.commit()
        
    else:
        # Перевірка часу останньої взаємодії (якщо не перша взаємодія)
        current_time = time.time()
        time_difference = current_time - player['last_action_time']

        if time_difference < 86400:  # Якщо пройшло менше 24 годин (86400 секунд)
            remaining_time = 86400 - time_difference
            messages = [
                f"@{username}, ти занадто швидко хочеш терти свій кий. Відпочинь ще {round(remaining_time, 1)} сек.",
                f"@{username}, твої шарики для боулінгу ще не відпочили. Зачекай ще {round(remaining_time, 1)} сек.",
                f"@{username}, твій кий ще червоний. Дай йому відпочити ще {round(remaining_time, 1)} сек."
            ]
            message_text = random.choice(messages)
            bot.send_message(message.chat.id, message_text)
            return

        # Оновлюємо час останньої взаємодії
        cursor = conn.cursor()
        cursor.execute("UPDATE players SET last_action_time = ? WHERE user_id = ?", (current_time, user_id))
        conn.commit()

        # Оновлюємо бали
        if random_action == "add":
            player['score'] += random_number_add
            cursor.execute("UPDATE players SET score = ? WHERE user_id = ?", (player['score'], user_id))
            conn.commit()
            bot.send_message(message.chat.id, f"@{username}, ти отримав {random_number_add} см! Кий збільшено! Загальний розмір кия: {player['score']} см.")
        else:
            player['score'] -= random_number_subtract
            player['score'] = max(player['score'], 0)  # Запобігаємо від'ємним балам
            cursor.execute("UPDATE players SET score = ? WHERE user_id = ?", (player['score'], user_id))
            conn.commit()
            bot.send_message(message.chat.id, f"@{username}, ти втратив {random_number_subtract} см! Кий зменшено! Загальний розмір кия: {player['score']} см.")

# Функція для виведення топу користувачів
@bot.message_handler(commands=['top'])
def show_top(message):
    cursor = conn.cursor()
    cursor.execute("SELECT username, score FROM players ORDER BY score DESC")
    players = cursor.fetchall()

    if not players:
        bot.send_message(message.chat.id, "В клубі ще немає людей.")
    else:
        top_message = "Топ більярдистів:\n"
        for idx, player in enumerate(players, start=1):
            top_message += f"{idx}. @{player[0]} - {player[1]} см\n"
        
        bot.send_message(message.chat.id, top_message)

# Запуск бота
bot.polling()

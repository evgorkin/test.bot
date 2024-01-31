#это мой первый опыт в создании бота в тг, выполняет следующие функции: имеет кнопку загрузки, при нажатии которой, пользователь прикрепляет файл excel в формате #таблицы с полями,бот получает файл, сохраняет,открывает файл библиотекой pandas,выводит содержимое в ответ пользователю,сохраняет содержимое в локальную БД #sqlite. Не до конца понял принцип xpath, так как не доводилось еще работать с SQL. По итогу вышло сыро, а может и вообще не вышло(  
import telebot
from telebot.types import ReplyKeyboardMarkup, KeyboardButton
import pandas as pd
import sqlite3
import os
database_file = "data.db"

if os.path.exists(database_file):
    print("База данных существует.")
else:
    print("База данных не существует.")

TOKEN = '6572313115:AAFc4oXFmd5WKArzZuMu9CQ-PFgkchvhS4I'
bot = telebot.TeleBot(TOKEN)

keyboard = ReplyKeyboardMarkup(row_width=1)
button = KeyboardButton('Загрузить файл')
keyboard.add(button)

@bot.message_handler(commands=['start'])
def start(message):
    bot.send_message(message.chat.id, 'Привет! Нажми кнопку "Загрузить файл", чтобы загрузить файл Excel с данными.', reply_markup=keyboard)

@bot.message_handler(content_types=['document'])
def handle_document(message):
    file_info = bot.get_file(message.document.file_id)
    downloaded_file = bot.download_file(file_info.file_path)
    with open('data.xlsx', 'wb') as file:
        file.write(downloaded_file)
    
    bot.send_message(message.chat.id, 'Файл успешно загружен. Обрабатываю данные...')
    try:
        data = pd.read_excel('data.xlsx')
        if set(['title', 'url', 'xpath']).issubset(data.columns):
            data.columns = ['Название', 'Ссылка', 'Цена']
            bot.send_message(message.chat.id, 'Содержимое файла:')
            with sqlite3.connect('data.db') as conn:
                cursor = conn.cursor()
                cursor.execute('''CREATE TABLE IF NOT EXISTS data (title TEXT, url TEXT, price TEXT)''')
                for row in data.itertuples():
                    bot.send_message(message.chat.id, f'Название: {row.Название}')
                    bot.send_message(message.chat.id, f'Ссылка: {row.Ссылка}')
                    bot.send_message(message.chat.id, f'Цена: {row.Цена}')
                    bot.send_message(message.chat.id, '---')
                    cursor.execute("INSERT INTO data VALUES (?, ?, ?)", (row.Название, row.Ссылка, row.Цена))
                conn.commit()
            bot.send_message(message.chat.id, 'Данные сохранены в базе данных.')
        else:
            bot.send_message(message.chat.id, 'Файл не содержит необходимых столбцов.')
    except Exception as e:
        bot.send_message(message.chat.id, f'Произошла ошибка при обработке файла: {e}')

bot.polling()

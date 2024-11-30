import aiohttp
from datetime import datetime, timedelta
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import Application, CommandHandler, CallbackQueryHandler, ContextTypes
import os

# Токен вашого бота
TOKEN = "7914907052:AAH2ZZFLmyIhA9CzTG8y_9jAefVe-IjfR6k"

# URL API
API_URL = "https://ubilling.net.ua/aerialalerts/"
MAP_URL = f"{API_URL}?map=true"

# Словник для відстеження обраного регіону
user_regions = {}

# Функція для отримання даних API
async def fetch_alerts(days=None):
    try:
        url = API_URL
        if days:
            url += f"?days={days}"  # Додаємо параметр днів, якщо він підтримується API

        async with aiohttp.ClientSession() as session:
            async with session.get(url) as response:
                if response.status != 200:
                    return None, f"Помилка: статус відповіді {response.status}"

                data = await response.json()
                if "states" not in data:
                    return None, "Некоректний формат даних від API."

                return data["states"], None
    except aiohttp.ClientError as e:
        return None, f"Помилка при отриманні даних: {e}"
    except ValueError as e:
        return None, f"Помилка при обробці даних: {e}"

# Головне меню
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    keyboard = [
        [InlineKeyboardButton("Перевірити тривоги", callback_data='check_alerts')],
        [InlineKeyboardButton("Обрати регіон", callback_data='choose_region')],
        [InlineKeyboardButton("Графік тривог для регіону", callback_data='view_chart')],
        [InlineKeyboardButton("Мапа тривог", callback_data='view_map')]
    ]
    reply_markup = InlineKeyboardMarkup(keyboard)
    
    if update.message:
        await update.message.reply_text("Оберіть дію:", reply_markup=reply_markup)
    elif update.callback_query:
        await update.callback_query.edit_message_text("Оберіть дію:", reply_markup=reply_markup)

# Перевірка тривог
async def check_alerts(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    query = update.callback_query
    await query.answer()

    states, error = await fetch_alerts()
    if error:
        keyboard = [[InlineKeyboardButton("🔙 Назад до меню", callback_data='back_to_menu')]]
        await query.edit_message_text(error, reply_markup=InlineKeyboardMarkup(keyboard))
        return

    alerts = []
    for region, info in states.items():
        alert_status = "🔴 ТРИВОГА" if info.get("alertnow") else "🟢 Спокійно"
        changed_time = info.get("changed", "Невідомий час")
        alerts.append(f"{region}: {alert_status} (змінено: {changed_time})")

    keyboard = [[InlineKeyboardButton("🔙 Назад до меню", callback_data='back_to_menu')]]
    await query.edit_message_text("\n".join(alerts), reply_markup=InlineKeyboardMarkup(keyboard))

# Вибір регіону
async def choose_region(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    query = update.callback_query
    await query.answer()

    states, error = await fetch_alerts()
    if error:
        keyboard = [[InlineKeyboardButton("🔙 Назад до меню", callback_data='back_to_menu')]]
        await query.edit_message_text(error, reply_markup=InlineKeyboardMarkup(keyboard))
        return

    keyboard = [
        [InlineKeyboardButton(region, callback_data=f"region_{region}")]
        for region in states.keys()
    ]
    keyboard.append([InlineKeyboardButton("🔙 Назад до меню", callback_data='back_to_menu')])

    reply_markup = InlineKeyboardMarkup(keyboard)
    await query.edit_message_text("Оберіть регіон для отримання сповіщень:", reply_markup=reply_markup)

# Підписка на регіон
async def subscribe_region(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    query = update.callback_query
    await query.answer()

    region = query.data.split("_", 1)[1]
    user_id = query.from_user.id

    user_regions[user_id] = region

    keyboard = [[InlineKeyboardButton("🔙 Назад до меню", callback_data='back_to_menu')]]
    await query.edit_message_text(
        f"Ви підписалися на сповіщення про тривоги для регіону: {region}.",
        reply_markup=InlineKeyboardMarkup(keyboard)
    )

# Графік тривог для регіону
async def view_chart(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    query = update.callback_query
    await query.answer()

    user_id = query.from_user.id
    region = user_regions.get(user_id)

    if not region:
        keyboard = [[InlineKeyboardButton("🔙 Назад до меню", callback_data='back_to_menu')]]
        await query.edit_message_text(
            "Ви ще не обрали регіон. Спочатку оберіть регіон через меню.",
            reply_markup=InlineKeyboardMarkup(keyboard)
        )
        return

    states, error = await fetch_alerts(days=7)
    if error or region not in states:
        keyboard = [[InlineKeyboardButton("🔙 Назад до меню", callback_data='back_to_menu')]]
        await query.edit_message_text(
            f"Немає достатніх даних для створення графіка для регіону {region}.",
            reply_markup=InlineKeyboardMarkup(keyboard)
        )
        return

    region_data = states[region]
    chart = generate_weekly_chart(region_data)

    keyboard = [[InlineKeyboardButton("🔙 Назад до меню", callback_data='back_to_menu')]]
    await query.edit_message_text(
        f"Графік тривог для регіону {region} за останні 7 днів:\n\n{chart}",
        reply_markup=InlineKeyboardMarkup(keyboard)
    )

# Генерація графіка для останніх 7 днів
def generate_weekly_chart(region_data):
    now = datetime.utcnow()
    days = ["Понеділок", "Вівторок", "Середа", "Четвер", "П'ятниця", "Субота", "Неділя"]
    weekly_chart = []

    history = region_data.get("history", {})
    
    for i in range(7):
        day = now - timedelta(days=i)
        day_name = days[day.weekday()]
        alert_status = history.get(day.strftime("%Y-%m-%d"), "🟢")  # Спокій за замовчуванням
        weekly_chart.append(f"{day_name}: {alert_status}")

    return "\n".join(reversed(weekly_chart))

# Завантаження та відправка карти тривог
async def send_alert_map(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    query = update.callback_query
    await query.answer()

    try:
        async with aiohttp.ClientSession() as session:
            async with session.get(MAP_URL) as response:
                if response.status != 200:
                    await query.edit_message_text(f"Помилка: статус відповіді {response.status}")
                    return

                file_path = "map_alerts.png"
                with open(file_path, "wb") as f:
                    f.write(await response.read())

        with open(file_path, "rb") as photo:
            await query.message.reply_photo(photo, caption="Мапа тривог в Україні")
        
        keyboard = [[InlineKeyboardButton("🔙 Назад до меню", callback_data='back_to_menu')]]
        await query.message.reply_text("Оберіть дію:", reply_markup=InlineKeyboardMarkup(keyboard))
        os.remove(file_path)

    except Exception as e:
        await query.edit_message_text(f"Помилка при завантаженні карти: {e}")

# Повернення в меню
async def back_to_menu(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    query = update.callback_query
    await query.answer()
    await start(update, context)

# Головна функція
def main():
    application = Application.builder().token(TOKEN).build()

    application.add_handler(CommandHandler("start", start))
    application.add_handler(CallbackQueryHandler(check_alerts, pattern='check_alerts'))
    application.add_handler(CallbackQueryHandler(choose_region, pattern='choose_region'))
    application.add_handler(CallbackQueryHandler(subscribe_region, pattern='region_'))
    application.add_handler(CallbackQueryHandler(view_chart, pattern='view_chart'))
    application.add_handler(CallbackQueryHandler(send_alert_map, pattern='view_map'))
    application.add_handler(CallbackQueryHandler(back_to_menu, pattern='back_to_menu'))

    application.run_polling()

if __name__ == '__main__':
    main()

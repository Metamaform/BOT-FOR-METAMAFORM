import logging
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import (
    Application, CommandHandler, MessageHandler, CallbackQueryHandler,
    filters, ContextTypes, ConversationHandler
)

# Настраиваем логирование, чтобы видеть, что происходит
logging.basicConfig(
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s', level=logging.INFO
)
logging.getLogger(name).addHandler(logging.StreamHandler())

# --- КОНФИГУРАЦИЯ БОТА ---
TOKEN = "7017264384:AAExI9yiSPA8EVs5JtW5TSobb-ABCzE_5Yg"  # Замените на свой токен бота!
ADMIN_IDS = [5698050836, 785532468]  # ID администраторов. 

# Словари для хранения данных
# user_messages: хранит связку user_chat_id -> admin_message_id (для ответов)
# banned_users: множество ID забаненных пользователей
user_messages_map = {}
banned_users = set()

# Состояния для ConversationHandler (для админ-панели)
BAN_USER_ID, UNBAN_USER_ID = range(2)

# --- Вспомогательные функции ---

def is_admin(user_id: int) -> bool:
    """Проверяет, является ли пользователь администратором."""
    return user_id in ADMIN_IDS

async def get_user_id_from_admin_message(message_text: str) -> int | None:
    """
    Пытаемся вытащить ID пользователя из текста пересланного админу сообщения.
    """
    user_id_prefix = "(ID: "
    if user_id_prefix in message_text:
        start_index = message_text.find(user_id_prefix) + len(user_id_prefix)
        end_index = message_text.find(",", start_index)
        if start_index != -1 and end_index != -1:
            try:
                user_id_str = message_text[start_index:end_index]
                return int(user_id_str)
            except ValueError:
                logging.warning(f"Не удалось извлечь user_id из текста: {message_text}")
    return None

# --- КНОПКИ ---

def get_user_start_keyboard() -> InlineKeyboardMarkup:
    """Клавиатура для обычных пользователей."""
    keyboard = [
        [InlineKeyboardButton("Написать в поддержку", callback_data="ask_support")],
        [InlineKeyboardButton("Помощь", callback_data="help_user")],
    ]
    return InlineKeyboardMarkup(keyboard)

def get_admin_panel_keyboard() -> InlineKeyboardMarkup:
    """Клавиатура для админ-панели."""
    keyboard = [
        [InlineKeyboardButton("Забанить пользователя", callback_data="admin_ban_user")],
        [InlineKeyboardButton("Разбанить пользователя", callback_data="admin_unban_user")],
        [InlineKeyboardButton("Показать забаненных", callback_data="admin_show_banned")],
        [InlineKeyboardButton("Закрыть панель", callback_data="admin_close_panel")],
    ]
    return InlineKeyboardMarkup(keyboard)

# --- КОМАНДЫ БОТА (только для админов) ---

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    """
    Обрабатывает команду /start.
    Если админ, показывает админ-панель. Если обычный пользователь, кнопки поддержки.
    """
    user = update.effective_user
    if is_admin(user.id):
        await update.message.reply_text(
            f"Привет, {user.full_name}! Добро пожаловать в админ-панель.",
            reply_markup=get_admin_panel_keyboard()
        )
    else:
        await update.message.reply_text(
            f"Привет, {user.mention_html()}! Я бот поддержки. Выберите действие:",
            reply_markup=get_user_start_keyboard()
        )

async def admin_panel(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    """
    Команда /admin_panel для быстрого доступа к панели админа.
    """
    if is_admin(update.effective_user.id):
        await update.message.reply_text(
            "Админ-панель:",
            reply_markup=get_admin_panel_keyboard()
        )
    else:
        await update.message.reply_text("У вас нет доступа к этой команде.")

# --- Обработчики CallbackQuery (нажатия на кнопки) ---

async def handle_button_press(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    """Обрабатывает нажатия на инлайн-кнопки."""
    query = update.callback_query
    await query.answer() # Обязательно "отвечаем" на callback, чтобы кнопка не висела "нажатой"

    if query.data == "ask_support":
        await query.edit_message_text(
            "Напишите свой вопрос или сообщение."
        )
    elif query.data == "help_user":
        await query.edit_message_text(
            "Просто напишите мне свой вопрос."
        )
    elif query.data == "admin_ban_user":
        await query.edit_message_text("Введите ID пользователя, которого хотите забанить:")
        return BAN_USER_ID # Переходим в состояние ожидания ID для бана
    elif query.data == "admin_unban_user":
        await query.edit_message_text("Введите ID пользователя, которого хотите разбанить:")
        return UNBAN_USER_ID # Переходим в состояние ожидания ID для разбана
    elif query.data == "admin_show_banned":
        if banned_users:
            banned_list = "\n".join(str(uid) for uid in banned_users)
            await query.edit_message_text(f"Забаненные пользователи:\n{banned_list}", reply_markup=get_admin_panel_keyboard())
        else:
            await query.edit_message_text("Список забаненных пользователей пуст.", reply_markup=get_admin_panel_keyboard())
    elif query.data == "admin_close_panel":
        await query.edit_message_text("Админ-панель закрыта.")
        # Можете удалить кнопки, если хотите: await query.edit_message_reply_markup(reply_markup=None)
    
    return ConversationHandler.END # Если не нужно продолжать диалог, выходим из состояния

# --- Функции бана/разбана (внутри ConversationHandler) ---

async def ban_user_by_id(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    """Обрабатывает ввод ID для бана."""
    try:
        user_id_to_ban = int(update.message.text)
        if user_id_to_ban == update.effective_user.id:
            await update.message.reply_text("Вы не можете забанить самого себя!")
        elif is_admin(user_id_to_ban):
            await update.message.reply_text("Нельзя забанить другого администратора.")
        elif user_id_to_ban in banned_users:
            await update.message.reply_text(f"Пользователь с ID {user_id_to_ban} уже забанен.")
        else:
            banned_users.add(user_id_to_ban)
            await update.message.reply_text(f"Пользователь с ID {user_id_to_ban} забанен.", reply_markup=get_admin_panel_keyboard())
            logging.info(f"Админ {update.effective_user.id} забанил пользователя {user_id_to_ban}.")
            # Опционально: уведомить забаненного пользователя
            try:
                await context.bot.send_message(chat_id=user_id_to_ban, text="Вы были забанены администратором и больше не можете писать в поддержку.")
            except Exception as e:
                logging.warning(f"Не удалось уведомить забаненного пользователя {user_id_to_ban}: {e}")
    except ValueError:
        await update.message.reply_text("Некорректный ID. Пожалуйста, введите числовой ID пользователя.", reply_markup=get_admin_panel_keyboard())
    except Exception as e:
        await update.message.reply_text(f"Произошла ошибка при бане: {e}", reply_markup=get_admin_panel_keyboard())
    
    return ConversationHandler.END # Завершаем ConversationHandler

async def unban_user_by_id(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    """Обрабатывает ввод ID для разбана."""
    try:
        user_id_to_unban = int(update.message.text)
        if user_id_to_unban not in banned_users:
            await update.message.reply_text(f"Пользователь с ID {user_id_to_unban} не забанен.")
        else:
            banned_users.remove(user_id_to_unban)
            await update.message.reply_text(f"Пользователь с ID {user_id_to_unban} разбанен.", reply_markup=get_admin_panel_keyboard())
            logging.info(f"Админ {update.effective_user.id} разбанил пользователя {user_id_to_unban}.")
            # Опционально: уведомить разбаненного пользователя
            try:
                await context.bot.send_message(chat_id=user_id_to_unban, text="Вы были разбанены администратором и теперь можете писать в поддержку.")
            except Exception as e:
                logging.warning(f"Не удалось уведомить разбаненного пользователя {user_id_to_unban}: {e}")
    except ValueError:
        await update.message.reply_text("Некорректный ID. Пожалуйста, введите числовой ID пользователя.", reply_markup=get_admin_panel_keyboard())
    except Exception as e:
        await update.message.reply_text(f"Произошла ошибка при разбане: {e}", reply_markup=get_admin_panel_keyboard())

    return ConversationHandler.END # Завершаем ConversationHandler

# --- ОБРАБОТКА СООБЩЕНИЙ ---

async def handle_user_message(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    """
    Ловит сообщения от обычных пользователей и пересылает их админам.
    Проверяет, не забанен ли пользователь.
    """
    user = update.effective_user
    message_text = update.message.text
    chat_id = update.effective_chat.id

    if chat_id in banned_users:
        await update.message.reply_text("Вы забанены и не можете писать в поддержку.")
        return

    # Отправляем сообщение всем админам
    for admin_id in ADMIN_IDS:
        try:
            # Важно: сохраняем user_chat_id в data. Это поможет при ответе.
            # Вместо парсинга текста, мы будем использовать это.
            sent_message_to_admin = await context.bot.send_message(
                chat_id=admin_id,
                text=f"Новое сообщение от пользователя {user.full_name} (ID: {user.id}, @{user.username or 'нет имени пользователя'}):\n\n{message_text}\n\n"
                     f"Чтобы ответить, 'Ответьте' на это сообщение в Telegram."
            )
            # Сохраняем ID сообщения, которое мы отправили админу, и связываем его с chat_id пользователя
            user_messages_map[sent_message_to_admin.message_id] = chat_id
            logging.info(f"Сообщение от пользователя {user.id} переслано админу {admin_id}.")
        except Exception as e:
            logging.error(f"Не удалось отправить сообщение админу {admin_id}: {e}")

    await update.message.reply_text("Ваше сообщение отправлено в службу поддержки. Ожидайте ответа.")


async def handle_admin_reply(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    """
    Обрабатывает ответы администраторов и пересылает их пользователям.
    """
    admin_id = update.effective_user.id
    reply_text = update.message.text
    
    if not is_admin(admin_id):
        return

    # Проверяем, является ли это ответом на сообщение
    if update.message.reply_to_message:
        replied_message_id = update.message.reply_to_message.message_id
        
        # Ищем, кому предназначался этот ответ, используя нашу карту
        target_user_chat_id = user_messages_map.get(replied_message_id)

        if target_user_chat_id:
            try:
                # Проверяем, не забанен ли пользователь, которому отвечаем
                if target_user_chat_id in banned_users:
                    await update.message.reply_text(f"Пользователь с ID {target_user_chat_id} забанен. Ответ не отправлен.")
                    return

                await context.bot.send_message(
                    chat_id=target_user_chat_id,
                    text=f"Ответ от поддержки:\n\n{reply_text}"
                )
                logging.info(f"Админ {admin_id} ответил пользователю {target_user_chat_id}.")
                await update.message.reply_text(f"Ваш ответ отправлен пользователю {target_user_chat_id}.")
                
                # Удаляем запись из карты, так как ответ отправлен
                # user_messages_map.pop(replied_message_id, None) # Можно очищать, чтобы не рос слишком сильно
            except Exception as e:
                logging.error(f"Не удалось отправить ответ пользователю {target_user_chat_id}: {e}")
                await update.message.reply_text(f"Не удалось отправить ответ пользователю {target_user_chat_id}. Возможно, он заблокировал бота. Ошибка: {e}")
        else:
            await update.message.reply_text("Не удалось найти пользователя, которому предназначался ответ. Возможно, это очень старое сообщение или ошибка.")
    else:
        await update.message.reply_text("Чтобы ответить пользователю, пожалуйста, используйте функцию 'Ответить' на его сообщение.")


async def post_init(application: Application) -> None:
    """
    Выполняется после инициализации приложения.
    Отображает информацию о боте при запуске.
    """
    bot = application.bot
    bot_info = await bot.get_me()
    logging.info(f"Бот запущен. Имя: {bot_info.first_name}, Username: @{bot_info.username}")
    logging.info(f"Администраторы: {ADMIN_IDS}")
    logging.info(f"Забаненные пользователи (на старте): {banned_users}")


def main() -> None:
    """Запускает бота."""
    application = Application.builder().token(TOKEN).post_init(post_init).build()

    # ConversationHandler для бана/разбана
    admin_conversation_handler = ConversationHandler(
        entry_points=[CallbackQueryHandler(handle_button_press, pattern="^admin_.*")],
        states={
            BAN_USER_ID: [MessageHandler(filters.TEXT & ~filters.COMMAND, ban_user_by_id)],
            UNBAN_USER_ID: [MessageHandler(filters.TEXT & ~filters.COMMAND, unban_user_by_id)],
        },
        fallbacks=[CommandHandler("start", start)], # Если админ передумал, можно просто написать /start
        map_to_parent={
            ConversationHandler.END: ConversationHandler.END # Важно, чтобы ConversationHandler завершался корректно
        }
    )

    # Обработчики команд
    application.add_handler(CommandHandler("start", start))
    application.add_handler(CommandHandler("admin_panel", admin_panel, filters=filters.User(ADMIN_IDS))) # Только админы могут вызвать /admin_panel

    # Обработчики кнопок
    # Важно: сначала обрабатываем кнопки админ-панели через ConversationHandler
    application.add_handler(admin_conversation_handler)
    # Затем общие кнопки для пользователей
    application.add_handler(CallbackQueryHandler(handle_button_press, pattern="^(ask_support|help_user)$"))

    # Обработчик сообщений от пользователей (не админов)
    # Пропускаем сообщения от админов, так как они либо отвечают, либо взаимодействуют с панелью
    application.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND & ~filters.User(ADMIN_IDS), handle_user_message))

    # Обработчик сообщений от админов (для ответов)
    # Админы отвечают на пересланные сообщения, поэтому фильтр на reply_to_message
    application.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND & filters.User(ADMIN_IDS) & filters.REPLY, handle_admin_reply))

    # Запускаем бота
    application.run_polling(allowed_updates=Update.ALL_TYPES)

if name == "main":
    main()

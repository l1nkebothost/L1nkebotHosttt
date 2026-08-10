import datetime
import re
import asyncio
import time
import logging
import random
import json
import os
import signal
from collections import defaultdict
from telegram import Update, ChatPermissions, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import (
    ApplicationBuilder, CommandHandler, MessageHandler, filters,
    ContextTypes, ConversationHandler, CallbackQueryHandler
)
from telegram.request import HTTPXRequest

# ==================== НАСТРОЙКИ ЛОГГИРОВАНИЯ ====================
logging.basicConfig(
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    level=logging.INFO
)
logger = logging.getLogger(__name__)

# ==================== КОНФИГУРАЦИЯ ====================
TOKEN = 'ВАШ_ТОКЕН_БОТА'
ADMIN_IDS = [123456789]  # Ваш Telegram ID

OWNER_CHANNEL = "https://t.me/L1nkerChaserYT"
OWNER_CHAT = "https://t.me/ChatL1nkeChaser"

DATA_FILE = 'bot_data.json'
SESSION_FILE = 'active_sessions.json'
SAVE_LOCK = asyncio.Lock()
AUTO_SAVE_INTERVAL = 30
SESSION_LIFETIME = 30
INTERCEPT_TIMEOUT = 30
REPORT_REASON = 1

# ==================== СЛОВАРИ ИГРЫ ====================
BAD_WORDS = ['твоя мамка шалава','соси','еблан','уебище','пошел нахуй','нахуй','хуй','пизда','бля','блять','сука','хуесос','гандон','мудак','пидор','шлюха','курва','залупа','очко','жопа','срака','говно','дерьмо','тварь']
URL_PATTERN = re.compile(r'(https?://\S+|www\.\S+|t\.me/\S+)')

EF_SCALE = {
    1: {'min':0,'max':100,'reward_min':1000,'reward_max':3000,'weight':10,'name':'EF-1'},
    2: {'min':100,'max':175,'reward_min':3000,'reward_max':5000,'weight':20,'name':'EF-2'},
    3: {'min':175,'max':250,'reward_min':5000,'reward_max':8000,'weight':40,'name':'EF-3'},
    4: {'min':250,'max':300,'reward_min':8000,'reward_max':12000,'weight':20,'name':'EF-4'},
    5: {'min':300,'max':400,'reward_min':12000,'reward_max':15000,'weight':5,'name':'EF-5'}
}

# Машины. Добавлено поле 'hp' (прочность). При 0 HP - машина ломается.
CARS = {
    'civil': {'name':'🚗 Гражданская машина','price':0,'max_speed':None,'desc':'Случайная выносливость (0-300 mph)', 'hp': 9999},
    'Dom-1': {'name':'🏎️ Dom-1','price':100000,'max_speed':200,'desc':'Выдерживает до 200 mph', 'hp': 100},
    'Dom-2': {'name':'🏎️ Dom-2','price':150000,'max_speed':250,'desc':'Выдерживает до 250 mph', 'hp': 100},
    'Dom-3': {'name':'🏎️ Dom-3','price':250000,'max_speed':325,'desc':'Выдерживает до 325 mph', 'hp': 100},
    'Tiv-1': {'name':'🚙 Tiv-1','price':100000,'max_speed':225,'desc':'Выдерживает до 225 mph', 'hp': 100},
    'Tiv-2': {'name':'🚙 Tiv-2','price':300000,'max_speed':400,'desc':'Выдерживает до 400 mph', 'hp': 100},
    'Tornado Attack': {'name':'💪 Tornado Attack','price':125000,'max_speed':250,'desc':'Выдерживает до 250 mph', 'hp': 100},
    'Dorothy': {'name':'🌪️ Dorothy','price':200000,'max_speed':200,'desc':'Выдерживает до 200 mph', 'hp': 100},
    'Tornado Puncher': {'name':'👊 Tornado Puncher','price':75000,'max_speed':200,'desc':'Выдерживает до 200 mph', 'hp': 100},
    'Titus': {'name':'🛡️ Titus','price':175000,'max_speed':245,'desc':'Выдерживает до 245 mph', 'hp': 100}
}

PROBES = {
    'twisted': {'name':'🌀 Twisted power зонт','price':1250,'max_hp':5,'desc':'Самый мощный зонт, выдерживает 5 перехватов', 'bonus_multiplier':2.0},
    'doto': {'name':'📡 Doto зонт','price':1000,'max_hp':4,'desc':'Средний зонт, выдерживает 4 перехвата', 'bonus_multiplier':1.5},
    'tiv': {'name':'🔭 Tiv Hammer зонт','price':750,'max_hp':3,'desc':'Базовый зонт, выдерживает 3 перехвата', 'bonus_multiplier':1.2}
}

RADARS = {
    'raxpol': {'name':'📡 Raxpol','price':300000,'desc':'Сканирует торнадо, улетает от EF-2 до EF-5', 'accuracy':1.0},
    'dow7': {'name':'🛰️ Dow-7','price':500000,'desc':'Точность 100%', 'accuracy':1.0},
    'dow6': {'name':'🛰️ Dow-6','price':600000,'desc':'Точность 90%', 'accuracy':0.9},
    'dow5': {'name':'🛰️ Dow-5','price':500000,'desc':'Точность 80%', 'accuracy':0.8},
    'dow4': {'name':'🛰️ Dow-4','price':400000,'desc':'Точность 70%', 'accuracy':0.7},
    'dow3': {'name':'🛰️ Dow-3','price':300000,'desc':'Точность 60%', 'accuracy':0.6},
    'umass': {'name':'🛰️ UMass','price':200000,'desc':'Точность 50%', 'accuracy':0.5}
}

STAR_PACKAGES = [
    {'stars': 5, 'money': 100000},
    {'stars': 10, 'money': 200000},
    {'stars': 15, 'money': 300000},
    {'stars': 20, 'money': 500000},
    {'stars': 35, 'money': 700000}
]

# ==================== ГЛОБАЛЬНЫЕ ПЕРЕМЕННЫЕ ====================
user_data = {}
chat_settings = {}
reports = []
bot_chats = set()
private_messages = []
chat_messages = {}
chat_user_stats = {}
cooldowns = {}
user_warn_count = {}
promocodes = {}
user_msg_times = {}
pending_star_purchases = {}
custom_interceptors = {}
BOT_START_TIME = time.time()

SPAM_WINDOW, SPAM_LIMIT, SPAM_MUTE_MINUTES, SPAM_BAN_MINUTES = 5, 3, 10, 60

# ==================== РАБОТА С JSON ====================
def load_sessions():
    if not os.path.exists(SESSION_FILE): return {}
    try:
        with open(SESSION_FILE, 'r', encoding='utf-8') as f: return json.load(f)
    except Exception as e:
        logger.error(f'Ошибка загрузки сессий: {e}')
        return {}

def save_sessions(sessions):
    try:
        with open(SESSION_FILE, 'w', encoding='utf-8') as f:
            json.dump(sessions, f, ensure_ascii=False, indent=2)
    except Exception as e:
        logger.error(f'Ошибка сохранения сессий: {e}')

def clear_old_sessions():
    sessions = load_sessions()
    now = time.time()
    expired = [sid for sid, s in sessions.items() if now - s.get('last_action', 0) > SESSION_LIFETIME]
    for sid in expired: del sessions[sid]
    if expired: save_sessions(sessions)

async def save_data():
    async with SAVE_LOCK:
        try:
            data = {
                'user_data': user_data, 'chat_settings': chat_settings, 'reports': reports,
                'bot_chats': list(bot_chats), 'private_messages': private_messages,
                'chat_messages': chat_messages, 'chat_user_stats': chat_user_stats,
                'cooldowns': cooldowns, 'user_warn_count': user_warn_count, 'promocodes': promocodes,
                'pending_star_purchases': pending_star_purchases, 'custom_interceptors': custom_interceptors
            }
            with open(DATA_FILE, 'w', encoding='utf-8') as f:
                json.dump(data, f, ensure_ascii=False, indent=2)
        except Exception as e:
            logger.error(f'Ошибка сохранения данных: {e}')

def load_data():
    global user_data, chat_settings, reports, bot_chats, private_messages, chat_messages, chat_user_stats, cooldowns, user_warn_count, promocodes, pending_star_purchases, custom_interceptors
    if not os.path.exists(DATA_FILE): return
    try:
        with open(DATA_FILE, 'r', encoding='utf-8') as f:
            data = json.load(f)
        user_data = data.get('user_data', {})
        chat_settings = data.get('chat_settings', {})
        reports = data.get('reports', [])
        bot_chats = set(data.get('bot_chats', []))
        private_messages = data.get('private_messages', [])
        chat_messages = data.get('chat_messages', {})
        chat_user_stats = data.get('chat_user_stats', {})
        cooldowns = data.get('cooldowns', {})
        user_warn_count = data.get('user_warn_count', {})
        promocodes = data.get('promocodes', {})
        pending_star_purchases = data.get('pending_star_purchases', {})
        custom_interceptors = data.get('custom_interceptors', {})
        
        for uid, udata in user_data.items():
            for field in ['current_car','cars_owned','blocked','rank','awards','messages_total','messages_today',
                          'probes','radars','current_radar','frozen','balance','warned','banned','muted','mute_until']:
                if field not in udata:
                    if field == 'cars_owned': udata[field] = ['civil']
                    elif field == 'current_car': udata[field] = 'civil'
                    elif field == 'probes': udata[field] = {}
                    elif field == 'radars': udata[field] = []
                    elif field == 'current_radar': udata[field] = None
                    elif field in ('frozen','banned','muted','blocked'): udata[field] = False
                    elif field in ('rank','awards','messages_total','messages_today','warned','balance'): udata[field] = 0
                    elif field == 'mute_until': udata[field] = None
    except Exception as e:
        logger.error(f'Ошибка загрузки данных: {e}')

async def auto_save_loop():
    while True:
        await asyncio.sleep(AUTO_SAVE_INTERVAL)
        await save_data()
        clear_old_sessions()

# ==================== УТИЛИТЫ ====================
def get_user(user_id): return user_data.get(str(user_id))

def ensure_user_exists(uid, full_name=None, username=None):
    uid = str(uid)
    if uid not in user_data:
        first = full_name.split()[0] if full_name else f'User{uid}'
        last = ' '.join(full_name.split()[1:]) if full_name else ''
        user_data[uid] = {
            'username': username, 'first_name': first, 'last_name': last,
            'messages_total': 0, 'messages_today': 0, 'last_message_date': None,
            'warned': 0, 'banned': False, 'muted': False, 'mute_until': None,
            'rank': 0, 'awards': 0, 'blocked': False, 'balance': 0,
            'current_car': 'civil', 'cars_owned': ['civil'],
            'probes': {}, 'radars': [], 'current_radar': None, 'frozen': False
        }
        asyncio.create_task(save_data())

def get_setting(chat_id, setting):
    chat_id = str(chat_id)
    if chat_id not in chat_settings:
        chat_settings[chat_id] = {'rules':'Не установлены','mat_filter':0,'link_filter':0,'del_msg':0,'greetings':'Добро пожаловать'}
        asyncio.create_task(save_data())
    return chat_settings[chat_id].get(setting, 0 if setting not in ('rules','greetings') else 'Не установлено')

def update_setting(chat_id, setting, value):
    chat_id = str(chat_id)
    if chat_id not in chat_settings:
        chat_settings[chat_id] = {'rules':'Не установлены','mat_filter':0,'link_filter':0,'del_msg':0,'greetings':'Добро пожаловать'}
    chat_settings[chat_id][setting] = value
    asyncio.create_task(save_data())

def get_user_rank(user_id): return get_user(user_id).get('rank', 0) if get_user(user_id) else 0

def is_owner_or_creator(user_id):
    uid = str(user_id)
    return uid in [str(a) for a in ADMIN_IDS] or get_user_rank(uid) >= 4

def can_mute(user_id): return str(user_id) in [str(a) for a in ADMIN_IDS] or get_user_rank(user_id) >= 1
def can_warn(user_id): return str(user_id) in [str(a) for a in ADMIN_IDS] or get_user_rank(user_id) >= 2
def can_ban(user_id): return str(user_id) in [str(a) for a in ADMIN_IDS] or get_user_rank(user_id) >= 3
def can_kick(user_id): return str(user_id) in [str(a) for a in ADMIN_IDS] or get_user_rank(user_id) in (2,3)

def parse_time(time_str: str) -> int:
    if not time_str: return 0
    try:
        unit = time_str[-1].lower()
        value = int(time_str[:-1])
        if unit == 's': return value
        elif unit == 'm': return value * 60
        elif unit == 'h': return value * 3600
        elif unit == 'd': return value * 86400
        elif unit == 'M': return value * 2592000
        else: return 0
    except: return 0

async def get_target_user(update, context):
    if update.message.reply_to_message:
        user = update.message.reply_to_message.from_user
        return str(user.id), user.full_name, user.username
    if context.args:
        arg = context.args[0]
        if arg.startswith('@'):
            username = arg[1:]
            for uid, data in user_data.items():
                if data.get('username') == username:
                    return uid, f"{data.get('first_name','')} {data.get('last_name','')}".strip() or username, username
            return None, None, None
        else:
            try:
                uid = str(int(arg))
                user = get_user(uid)
                if user:
                    return uid, f"{user.get('first_name','')} {user.get('last_name','')}".strip() or user.get('username') or uid, user.get('username')
                else:
                    ensure_user_exists(uid, f'User {uid}', None)
                    return uid, f'User {uid}', None
            except:
                return None, None, None
    return None, None, None

# ==================== ДЕКОРАТОРЫ ====================
def block_check(func):
    async def wrapper(update, context, *args, **kwargs):
        user_id = str(update.effective_user.id)
        user = get_user(user_id)
        if user and user.get('blocked', False):
            try:
                await update.message.delete()
                await update.message.reply_text(f'🚫 ВЫ ЗАБЛОКИРОВАНЫ В БОТЕ!\n\n📌 Если вы считаете, что это ошибка, обратитесь к владельцу:\n👤 Канал: {OWNER_CHANNEL}\n💬 Чат: {OWNER_CHAT}')
            except: pass
            return
        return await func(update, context, *args, **kwargs)
    return wrapper

def check_session_exists(func):
    async def wrapper(update: Update, context: ContextTypes.DEFAULT_TYPE, *args, **kwargs):
        query = update.callback_query
        if not query: return
        data = query.data.split('||')
        if len(data) != 2:
            await query.edit_message_text('❌ Ошибка данных сессии'); await query.answer(); return
        session_id = data[1]
        sessions = load_sessions()
        session = sessions.get(session_id)
        if not session or session.get('user_id') != str(update.effective_user.id):
            await query.edit_message_text('⏰ Сессия истекла или не найдена. Начните заново /intercept'); await query.answer(); return
        if time.time() - session.get('last_action', 0) > INTERCEPT_TIMEOUT:
            await query.edit_message_text('⏰ Сессия истекла (30 секунд). Начните заново /intercept')
            sessions = load_sessions()
            if session_id in sessions: del sessions[session_id]; save_sessions(sessions)
            await query.answer(); return
        session['last_action'] = time.time()
        sessions[session_id] = session; save_sessions(sessions)
        return await func(update, context, session, session_id, *args, **kwargs)
    return wrapper

# ==================== ПРОВЕРКА ИЗНОСА (4 ДНЯ) ====================
async def check_and_breakdown_items():
    today = datetime.date.today().isoformat()
    for user_id, user in user_data.items():
        # Проверка машин на HP
        cars = user.get('cars_owned', ['civil'])
        changed = False
        for car_key in cars[:]: # Итерируемся по копии списка
            if car_key == 'civil': continue
            
            all_cars = {**CARS, **custom_interceptors}
            car_data = all_cars.get(car_key)
            if not car_data: continue
            
            current_hp = user.get('car_hp', {}).get(car_key, car_data.get('hp', 100))
            
            # Если HP упал до 0 или меньше - ломаем
            if current_hp <= 0:
                cars.remove(car_key)
                if car_key in user.get('car_hp', {}): del user['car_hp'][car_key]
                if user.get('current_car') == car_key:
                    user['current_car'] = 'civil'
                changed = True
                
                # Отправляем уведомление в ЛС и в чат!
                try:
                    await bot_instance.bot.send_message(
                        int(user_id),
                        f"💥 ВАША МАШИНА {car_data['name']} УНИЧТОЖЕНА! 💥\n\nПричина: Прочность упала до 0."
                    )
                except: pass
                
                # Оповещение в чат по юзеру
                for chat_id in bot_chats:
                    try:
                        await bot_instance.bot.send_message(
                            int(chat_id),
                            f"💥 Машина пользователя [ID: {user_id}] уничтожена!"
                        )
                    except: pass

        if changed:
            user['cars_owned'] = cars
            await save_data()

async def breakdown_check_loop():
    while True:
        await asyncio.sleep(21600) # Раз в 6 часов
        try: await check_and_breakdown_items()
        except Exception as e: logger.error(f'Ошибка в цикле разрушения: {e}')

# ==================== ОСНОВНЫЕ КОМАНДЫ ====================
@block_check
async def start(update, context):
    ensure_user_exists(update.effective_user.id, update.effective_user.full_name, update.effective_user.username)
    await update.message.reply_text('🤖 Бот-модератор и охотник за торнадо! Используйте /help для списка команд 🌪️')

@block_check
async def help_command(update, context):
    text = """
🤖 КОМАНДЫ БОТА 🤖

🌪️ ПЕРЕХВАТ ТОРНАДО:
/intercept - начать перехват (кулдаун 30 сек)

💰 ЭКОНОМИКА:
/balance - баланс
/buy_money - купить деньги за звёзды ⭐
/shop_intercept - магазин машин
/buy_intercept <название> - купить машину
/my_interceptors - мои машины
/wear <название> - надеть машину
/my_car - текущая машина и её прочность
/give_money <ID/юзернейм> <сумма> - выдать (админы)
/take_money <ID/юзернейм> <сумма> - снять (админы)

☂️ ЗОНТЫ:
/shop_probs - магазин зонтов
/buy_prob <название> - купить зонт
/probs - мои зонты

🛰️ РАДАРЫ:
/shop_radars - магазин радаров
/buy_radar <название> - купить радар
/radars - мои радары
/wear_radar <название> - надеть радар

👤 ПРОФИЛЬ:
/profile - профиль
/rules - правила чата
/top - топ за сегодня
/top_all - топ за всё время
/chat_top - топ-10 активных
/users - все участники чата
/users_count - всего пользователей
/awards_top - топ по наградам

🎫 ПРОМОКОДЫ:
/promocode <код> - активировать
/createpromo <код> <сумма> [лимит] [часы] - создать (владелец)

🔨 МОДЕРАЦИЯ:
/mute, /unmute, /warn, /ban, /kick, /report

👑 ВЛАДЕЛЕЦ (ЛС):
/block, /promote, /demote, /award, /sync, /say, /inbox, /clean, /reports, /save
/add_interceptor - добавить кастомную машину
/list_interceptors - список всех машин
/remove_interceptor - удалить кастомную машину
/fix_json - починить битые данные в магазине
/join - добавить бота в чат
/remove_from_chat - удалить бота из чата
    """
    await update.message.reply_text(text)

# ==================== МАГАЗИН И МАШИНЫ (HP) ====================
@block_check
async def shop_intercept(update, context):
    try:
        all_cars = {**CARS, **custom_interceptors}
        text = '🚗 МАГАЗИН МАШИН 🚗\n\n'
        
        civil = all_cars.get('civil', CARS.get('civil', {}))
        text += f'{civil.get("name", "Гражданская")}\n'
        text += f'   💰 Цена: бесплатно\n'
        text += f'   💪 Выносливость: случайная (0-300 mph)\n'
        text += f'   📝 {civil.get("desc", "")}\n\n'
        
        for key, car in all_cars.items():
            if key == 'civil': continue
            car_name = car.get('name')
            if car_name is None: continue
            speed = car.get('max_speed', 'случайная') or 'случайная'
            text += f'{car_name}\n'
            text += f'   💰 Цена: ${car.get("price", 0):,}\n'
            text += f'   💪 Выносливость: {speed} mph\n'
            text += f'   🛡️ Прочность (HP): {car.get("hp", 100)}\n'
            text += f'   📝 {car.get("desc", "")}\n\n'
            
        text += '⚠️ Прочность тратится при перехватах. При HP=0 машина уничтожается!\n'
        text += '📝 Используйте /buy_intercept <название>\n\n'
        available = [key for key in all_cars if key != 'civil']
        text += '📋 Доступные модели:\n• ' + '\n• '.join(available)
        
        if len(text) > 4096:
            parts = []
            current_part = ''
            for line in text.split('\n'):
                if len(current_part) + len(line) + 1 > 4000:
                    parts.append(current_part)
                    current_part = line + '\n'
                else:
                    current_part += line + '\n'
            if current_part: parts.append(current_part)
            for part in parts: await update.message.reply_text(part)
        else:
            await update.message.reply_text(text)
    except Exception as e:
        logger.error(f'Ошибка в shop_intercept: {e}')
        await update.message.reply_text(f'❌ Ошибка при загрузке магазина: {str(e)}')

@block_check
async def buy_intercept(update, context):
    if not context.args:
        await update.message.reply_text('📝 Укажите название: /buy_intercept Dom-1')
        return
    car_name = ' '.join(context.args).strip()
    all_cars = {**CARS, **custom_interceptors}
    found = None
    for k in all_cars:
        if k.lower() == car_name.lower():
            found = k; break
    
    if not found or found == 'civil':
        await update.message.reply_text(f'❌ Машина "{car_name}" не найдена. Используйте /shop_intercept')
        return
    
    user = get_user(str(update.effective_user.id))
    if not user: await update.message.reply_text('💰 Сначала /start'); return
    
    car_info = all_cars.get(found)
    price = car_info.get('price', 0)
    if user.get('balance', 0) < price:
        await update.message.reply_text(f'❌ Недостаточно денег. Нужно ${price:,}')
        return
    
    owned = user.get('cars_owned', [])
    if found in owned:
        await update.message.reply_text('❌ У вас уже есть эта машина')
        return
    
    user['balance'] -= price
    owned.append(found)
    user['cars_owned'] = owned
    user['current_car'] = found
    # Инициализация HP для машины при покупке
    if 'car_hp' not in user: user['car_hp'] = {}
    user['car_hp'][found] = car_info.get('hp', 100)
    
    await save_data()
    await update.message.reply_text(f'✅ Куплена {car_info.get("name", found)} за ${price:,}\n💰 Баланс: ${user["balance"]:,}')

@block_check
async def my_interceptors(update, context):
    user = get_user(str(update.effective_user.id))
    if not user: await update.message.reply_text('💰 Сначала /start'); return
    owned = user.get('cars_owned', [])
    if not owned or owned == ['civil']:
        await update.message.reply_text('🚫 Нет купленных машин'); return
    
    current = user.get('current_car','civil')
    all_cars = {**CARS, **custom_interceptors}
    text = '🏎️ ВАШИ МАШИНЫ:\n\n'
    for key in owned:
        car = all_cars.get(key)
        if not car: continue
        marker = ' ✅ ТЕКУЩАЯ' if key == current else ''
        speed = car.get('max_speed', 'случайная') or 'случайная'
        
        # Получаем текущее HP машины
        hp = user.get('car_hp', {}).get(key, car.get('hp', 100))
        
        text += f'• {car["name"]}{marker}\n'
        text += f'  💪 Выносливость: {speed} mph\n'
        text += f'  🛡️ Прочность (HP): {hp}/{car.get("hp", 100)}\n\n'
        
    text += '📝 Используйте /wear <название> чтобы надеть машину'
    await update.message.reply_text(text)

@block_check
async def my_car(update, context):
    user = get_user(str(update.effective_user.id))
    if not user: await update.message.reply_text('💰 Сначала /start'); return
    
    current_key = user.get('current_car','civil')
    all_cars = {**CARS, **custom_interceptors}
    car = all_cars.get(current_key, all_cars['civil'])
    
    hp = user.get('car_hp', {}).get(current_key, car.get('hp', 100))
    max_hp = car.get('hp', 100)
    
    max_speed_display = 'случайная (0-300 mph)' if current_key == 'civil' else f'{car.get("max_speed", "неизвестно")} mph'
    text = f'🚗 Текущая машина: {car["name"]}\n📝 Описание: {car["desc"]}\n💪 Макс. выносливость: {max_speed_display}\n🛡️ Прочность (HP): {hp}/{max_hp}'
    await update.message.reply_text(text)

@block_check
async def wear_car(update, context):
    if not context.args:
        await update.message.reply_text('📝 Укажите название: /wear Dom-1'); return
    car_name = ' '.join(context.args).strip()
    all_cars = {**CARS, **custom_interceptors}
    found = None
    for k in all_cars:
        if k.lower() == car_name.lower():
            found = k; break
    if not found:
        await update.message.reply_text('❌ Машина не найдена'); return
    
    user = get_user(str(update.effective_user.id))
    if not user: await update.message.reply_text('💰 Сначала /start'); return
    if found not in user.get('cars_owned', []):
        await update.message.reply_text('❌ У вас нет этой машины'); return
    
    user['current_car'] = found
    await save_data()
    await update.message.reply_text(f'✅ Текущая машина: {all_cars[found]["name"]}')

# ==================== ДЕНЬГИ (Выдача/Снятие) ====================
@block_check
async def give_money(update, context):
    if not is_owner_or_creator(update.effective_user.id):
        await update.message.reply_text('🚫 Нет прав'); return
    if len(context.args) < 2 and not update.message.reply_to_message:
        await update.message.reply_text('📝 Используйте: /give_money <ID/юзернейм> <сумма> ИЛИ ответьте на сообщение пользователя: /give_money <сумма>')
        return
    
    target_id, full_name, username = await get_target_user(update, context)
    if not target_id:
        await update.message.reply_text('❌ Не удалось определить пользователя'); return
    
    try:
        amount = int(context.args[-1])
    except:
        await update.message.reply_text('❌ Сумма должна быть числом'); return
    if amount <= 0:
        await update.message.reply_text('❌ Сумма должна быть положительной'); return
    
    user = get_user(target_id)
    if not user:
        ensure_user_exists(target_id, full_name, username)
        user = get_user(target_id)
    
    user['balance'] = user.get('balance',0) + amount
    await save_data()
    await update.message.reply_text(f'✅ {full_name} получил ${amount:,}. Баланс: ${user["balance"]:,}')

@block_check
async def take_money(update, context):
    if not is_owner_or_creator(update.effective_user.id):
        await update.message.reply_text('🚫 Нет прав'); return
    if len(context.args) < 2 and not update.message.reply_to_message:
        await update.message.reply_text('📝 Используйте: /take_money <ID/юзернейм> <сумма> ИЛИ ответьте на сообщение пользователя: /take_money <сумма>')
        return
    
    target_id, full_name, username = await get_target_user(update, context)
    if not target_id:
        await update.message.reply_text('❌ Не удалось определить пользователя'); return
    
    try:
        amount = int(context.args[-1])
    except:
        await update.message.reply_text('❌ Сумма должна быть числом'); return
    if amount <= 0:
        await update.message.reply_text('❌ Сумма должна быть положительной'); return
    
    user = get_user(target_id)
    if not user:
        ensure_user_exists(target_id, full_name, username)
        user = get_user(target_id)
    
    if user.get('balance',0) < amount:
        await update.message.reply_text(f'❌ Недостаточно средств. Баланс: ${user["balance"]:,}'); return
    
    user['balance'] -= amount
    await save_data()
    await update.message.reply_text(f'✅ Снято ${amount:,} у {full_name}. Баланс: ${user["balance"]:,}')

@block_check
async def balance(update, context):
    user = get_user(str(update.effective_user.id))
    if not user: await update.message.reply_text('💰 Сначала /start'); return
    await update.message.reply_text(f'💰 Баланс: ${user.get("balance",0):,}')

# ==================== ОСТАЛЬНЫЕ КОМАНДЫ (зонты, радары, профиль) ====================
@block_check
async def shop_probs(update, context):
    text = '☂️ МАГАЗИН ЗОНТОВ ☂️\n\n'
    for key, prob in PROBES.items():
        text += f'{prob["name"]}\n   💰 Цена: ${prob["price"]:,}\n   💪 HP: {prob["max_hp"]}\n   📝 {prob["desc"]}\n   🎯 Бонус: x{prob["bonus_multiplier"]}\n\n'
    text += '📝 Используйте /buy_prob <название>\nДоступно: twisted, tiv, doto'
    await update.message.reply_text(text)

@block_check
async def buy_prob(update, context):
    if not context.args: await update.message.reply_text('📝 Укажите тип: /buy_prob twisted'); return
    prob_type = context.args[0].lower()
    if prob_type not in PROBES: await update.message.reply_text('❌ Доступно: twisted, tiv, doto'); return
    user = get_user(str(update.effective_user.id))
    if not user: await update.message.reply_text('💰 Сначала /start'); return
    prob = PROBES[prob_type]
    if user.get('balance',0) < prob['price']: await update.message.reply_text(f'❌ Недостаточно денег. Нужно ${prob["price"]:,}'); return
    probes = user.get('probes', {})
    if prob_type in probes: await update.message.reply_text('❌ У вас уже есть такой зонт'); return
    probes[prob_type] = prob['max_hp']
    user['probes'] = probes
    user['balance'] -= prob['price']
    await save_data()
    await update.message.reply_text(f'✅ Куплен {prob["name"]} за ${prob["price"]:,}')

@block_check
async def probs(update, context):
    user = get_user(str(update.effective_user.id))
    if not user: await update.message.reply_text('💰 Сначала /start'); return
    probes = user.get('probes', {})
    if not probes: await update.message.reply_text('🚫 Нет зонтов. Купите в /shop_probs'); return
    text = '☂️ ВАШИ ЗОНТЫ:\n\n'
    for pkey, hp in probes.items():
        prob = PROBES.get(pkey)
        if prob: text += f'• {prob["name"]}\n  💪 HP: {hp}/{prob["max_hp"]}\n  🎯 Бонус: x{prob["bonus_multiplier"]}\n\n'
    await update.message.reply_text(text)

@block_check
async def shop_radars(update, context):
    text = '🛰️ МАГАЗИН РАДАРОВ 🛰️\n\n'
    for key, rad in RADARS.items():
        text += f'{rad["name"]}\n   💰 Цена: ${rad["price"]:,}\n   🎯 Точность: {int(rad["accuracy"]*100)}%\n   📝 {rad["desc"]}\n\n'
    text += '📝 Используйте /buy_radar <название>\nДоступно: raxpol, dow7, dow6, dow5, dow4, dow3, umass'
    await update.message.reply_text(text)

@block_check
async def buy_radar(update, context):
    if not context.args: await update.message.reply_text('📝 Укажите тип: /buy_radar raxpol'); return
    radar_type = context.args[0].lower()
    if radar_type not in RADARS: await update.message.reply_text('❌ Доступно: raxpol, dow7, dow6, dow5, dow4, dow3, umass'); return
    user = get_user(str(update.effective_user.id))
    if not user: await update.message.reply_text('💰 Сначала /start'); return
    radar = RADARS[radar_type]
    if user.get('balance',0) < radar['price']: await update.message.reply_text(f'❌ Недостаточно денег. Нужно ${radar["price"]:,}'); return
    radars = user.get('radars', [])
    if radar_type in radars: await update.message.reply_text('❌ У вас уже есть этот радар'); return
    radars.append(radar_type)
    user['radars'] = radars
    user['balance'] -= radar['price']
    if not user.get('current_radar'): user['current_radar'] = radar_type
    await save_data()
    await update.message.reply_text(f'✅ Куплен {radar["name"]} за ${radar["price"]:,}')

@block_check
async def radars(update, context):
    user = get_user(str(update.effective_user.id))
    if not user: await update.message.reply_text('💰 Сначала /start'); return
    radars = user.get('radars', [])
    if not radars: await update.message.reply_text('🚫 Нет радаров. Купите в /shop_radars'); return
    current = user.get('current_radar')
    text = '🛰️ ВАШИ РАДАРЫ:\n\n'
    for r in radars:
        radar = RADARS.get(r)
        if radar:
            marker = ' ✅ АКТИВНЫЙ' if r == current else ''
            text += f'• {radar["name"]}{marker}\n  🎯 Точность: {int(radar["accuracy"]*100)}%\n\n'
    await update.message.reply_text(text)

@block_check
async def wear_radar(update, context):
    if not context.args: await update.message.reply_text('📝 Укажите тип: /wear_radar raxpol'); return
    radar_type = context.args[0].lower()
    if radar_type not in RADARS: await update.message.reply_text('❌ Доступно: raxpol, dow7, dow6, dow5, dow4, dow3, umass'); return
    user = get_user(str(update.effective_user.id))
    if not user: await update.message.reply_text('💰 Сначала /start'); return
    if radar_type not in user.get('radars', []): await update.message.reply_text('❌ У вас нет этого радара'); return
    user['current_radar'] = radar_type
    await save_data()
    await update.message.reply_text(f'✅ Активный радар: {RADARS[radar_type]["name"]}')

@block_check
async def profile(update, context):
    u = get_user(str(update.effective_user.id))
    if not u: await update.message.reply_text('❌ Профиль не найден'); return
    rank_names = {0:'Пользователь',1:'Младший модер',2:'Средний модер',3:'Старший модер',4:'Создатель'}
    current_car_key = u.get('current_car','civil')
    all_cars = {**CARS, **custom_interceptors}
    car_info = all_cars.get(current_car_key, all_cars['civil'])
    car_name = car_info.get('name', 'Гражданская')
    car_hp = u.get('car_hp', {}).get(current_car_key, car_info.get('hp', 100))
    
    radar_name = RADARS.get(u.get('current_radar'), {}).get('name', 'Нет') if u.get('current_radar') else 'Нет'
    
    probes = u.get('probes', {})
    probes_text = "Нет"
    if probes:
        probes_list = [f"{PROBES[p]['name']} (HP: {hp})" for p, hp in probes.items() if p in PROBES]
        probes_text = ", ".join(probes_list)
    
    text = f"""👤 ПРОФИЛЬ 👤

Имя: {u.get('first_name','')} {u.get('last_name','')}
Username: @{u.get('username','')}
Ранг: {rank_names.get(u.get('rank',0),'Неизвестно')}
Сообщений: всего {u.get('messages_total',0)}
Наград: {u.get('awards',0)}
Предупреждений: {u.get('warned',0)}
💰 Баланс: ${u.get('balance',0):,}
🚗 Машина: {car_name} (HP: {car_hp})
🛰️ Радар: {radar_name}
☂️ Зонты: {probes_text}"""
    await update.message.reply_text(text)

# ==================== ОСНОВНОЙ ПЕРЕХВАТ (ИЗМЕНЕНА МЕХАНИКА) ====================
@block_check
async def intercept(update, context):
    user_id = str(update.effective_user.id)
    user = get_user(user_id)
    if not user: await update.message.reply_text('💰 Сначала /start'); return
    
    sessions = load_sessions()
    for sid in list(sessions.keys()):
        if sessions[sid]['user_id'] == user_id: del sessions[sid]
    save_sessions(sessions)
    
    cooldown_key = f'{user_id}_intercept'
    now = time.time()
    if cooldown_key in cooldowns and now - cooldowns[cooldown_key] < 30:
        remaining = int(30 - (now - cooldowns[cooldown_key]))
        await update.message.reply_text(f'⏰ Подождите {remaining} секунд перед следующим перехватом')
        return
    
    cooldowns[cooldown_key] = now
    await save_data()
    
    current_car_key = user.get('current_car','civil')
    all_cars = {**CARS, **custom_interceptors}
    car_data = all_cars.get(current_car_key, all_cars['civil'])
    has_radar = user.get('current_radar') is not None
    radar_name = RADARS[user['current_radar']]['name'] if has_radar else None
    
    session_id = f"{user_id}_{int(time.time())}"
    sessions = load_sessions()
    sessions[session_id] = {
        'user_id': user_id, 'chat_id': str(update.effective_chat.id),
        'car_key': current_car_key, 'has_radar': has_radar,
        'radar_name': radar_name, 'radar_type': user.get('current_radar') if has_radar else None,
        'use_probes': False, 'mode': None, 'start_time': time.time(),
        'ef_level': None, 'wind_speed': None, 'final_speed': None,
        'last_action': time.time(), 'scan_reward_given': False
    }
    save_sessions(sessions)
    context.user_data['intercept_session'] = session_id
    
    keyboard = [[InlineKeyboardButton("📡 Сканировать", callback_data=f"intercept_mode_scan||{session_id}")],
                [InlineKeyboardButton("🌪️ Перехватывать", callback_data=f"intercept_mode_intercept||{session_id}")]]
    
    await update.message.reply_text(
        f"🌪️ ТОРНАДО ОБНАРУЖЕНО! 🌪️\n\n🚗 Машина: {car_data['name']}\n{'🛰️ Радар: ' + radar_name if has_radar else '❌ Радар не установлен'}\n\n⏰ У вас 30 секунд на выбор режима.",
        reply_markup=InlineKeyboardMarkup(keyboard)
    )

@block_check
@check_session_exists
async def intercept_callback(update: Update, context: ContextTypes.DEFAULT_TYPE, session, session_id):
    query = update.callback_query
    await query.answer()
    parts = query.data.split('||')
    action_data = parts[0]
    action_parts = action_data.split('_')
    action = action_parts[1]
    user = get_user(session['user_id'])
    
    if action in ('leave', 'stay'):
        if action == 'leave': await intercept_leave(update, context, session, session_id)
        else: await intercept_stay(update, context, session, session_id)
        return
    
    if action == 'mode':
        mode = action_parts[2]
        session['mode'] = mode
        session['last_action'] = time.time()
        sessions = load_sessions(); sessions[session_id] = session; save_sessions(sessions)
        
        has_probes = len(user.get('probes', {})) > 0
        keyboard = []
        if has_probes: keyboard.append([InlineKeyboardButton("☂️ Поставить зонты", callback_data=f"intercept_probes_on||{session_id}")])
        keyboard.append([InlineKeyboardButton("❌ Без зонтов", callback_data=f"intercept_probes_off||{session_id}")])
        
        await query.edit_message_text(
            f"{'📡 СКАНИРОВАНИЕ' if mode == 'scan' else '🌪️ ПЕРЕХВАТ'}\n\n🚗 Машина: {CARS[session['car_key']]['name']}\n{'🛰️ Радар: ' + session['radar_name'] if session['has_radar'] else '❌ Без радара'}\n{'✅ Есть зонты' if has_probes else '❌ Нет зонтов'}\n\nВыберите опции:",
            reply_markup=InlineKeyboardMarkup(keyboard)
        )
    
    elif action == 'probes':
        session['use_probes'] = action_parts[2] == 'on'
        session['last_action'] = time.time()
        sessions = load_sessions(); sessions[session_id] = session; save_sessions(sessions)
        if session['mode'] == 'scan': await start_scanning(update, context, session, session_id)
        else: await start_intercept_mode(update, context, session, session_id)

async def start_intercept_mode(update: Update, context: ContextTypes.DEFAULT_TYPE, session, session_id):
    chat_id = session['chat_id']
    user_id = session['user_id']
    user = get_user(user_id)
    car_key = session['car_key']
    all_cars = {**CARS, **custom_interceptors}
    car_data = all_cars.get(car_key, all_cars['civil'])
    
    await context.bot.send_message(chat_id, f"🌪️ ПЕРЕХВАТ НАЧАЛСЯ! 🌪️\n🚗 Машина: {car_data['name']}\n{'☂️ Зонты установлены' if session.get('use_probes') else '❌ Без зонтов'}\n⚠️ Торнадо приближается!")
    
    ef_levels = list(EF_SCALE.keys())
    weights = [EF_SCALE[l]['weight'] for l in ef_levels]
    ef_choice = random.choices(ef_levels, weights=weights, k=1)[0]
    ef_info = EF_SCALE[ef_choice]
    wind_speed = random.randint(ef_info['min'], ef_info['max'])
    
    session['ef_level'] = ef_choice; session['wind_speed'] = wind_speed
    session['last_action'] = time.time()
    sessions = load_sessions(); sessions[session_id] = session; save_sessions(sessions)
    
    for i in range(3):
        await asyncio.sleep(1)
        current_speed = max(0, wind_speed + random.randint(-20, 20))
        await context.bot.send_message(chat_id, f"🌪️ Тряска усиливается... {i+1}/3\n💨 Скорость ветра: ~{current_speed} mph")
        session['last_action'] = time.time(); sessions = load_sessions(); sessions[session_id] = session; save_sessions(sessions)
    
    car_endurance = random.randint(0, 300) if car_key == 'civil' else car_data.get('max_speed', 0)
    endurance_text = f"случайная ({car_endurance} mph)" if car_key == 'civil' else f"{car_endurance} mph"
    
    success = wind_speed <= car_endurance
    
    probe_bonus = 1.0; probes_text = ""
    if session.get('use_probes') and user.get('probes'):
        best_probe = max([p for p in user.get('probes', {}) if p in PROBES], key=lambda x: PROBES[x]['bonus_multiplier'], default=None)
        if best_probe:
            probe_bonus = PROBES[best_probe]['bonus_multiplier']
            probes_text = f"\n☂️ Бонус зонта {PROBES[best_probe]['name']}: x{probe_bonus}"
    
    # --- ИЗМЕНЕНИЕ МЕХАНИКИ ПРОЧНОСТИ МАШИН ---
    # Вычитаем ХП в зависимости от успеха/провала
    hp_loss = 10 if success else 25
    car_hps = user.get('car_hp', {})
    current_hp = car_hps.get(car_key, car_data.get('hp', 100))
    new_hp = max(0, current_hp - hp_loss)
    car_hps[car_key] = new_hp
    user['car_hp'] = car_hps
    
    reward_text = ""
    if success:
        reward = random.randint(ef_info['reward_min'], ef_info['reward_max'])
        bonus_reward = int(reward * (probe_bonus - 1))
        total_reward = reward + bonus_reward
        user['balance'] = user.get('balance', 0) + total_reward
        reward_text = f"💰 Базовая награда: +${reward:,}\n💰 Бонус зонта: +${bonus_reward:,}\n💰 Итого: +${total_reward:,}"
    else:
        base_reward = random.randint(ef_info['reward_min'] // 2, ef_info['reward_max'] // 2)
        bonus_reward = int(base_reward * (probe_bonus - 1))
        total_reward = base_reward + bonus_reward
        user['balance'] = user.get('balance', 0) + total_reward
        reward_text = f"💰 Компенсация: +${base_reward:,}\n💰 Бонус зонта: +${bonus_reward:,}\n💰 Итого: +${total_reward:,}"
    
    # Расход зонтов
    if session.get('use_probes') and user.get('probes'):
        probes = user.get('probes', {})
        new_probes = {}
        broken = []
        for pkey, hp in probes.items():
            new_hp = hp - 1
            if new_hp <= 0: broken.append(PROBES[pkey]['name'])
            else: new_probes[pkey] = new_hp
        user['probes'] = new_probes
        if broken: probes_text += f"\n☂️ Зонты сломаны: {', '.join(broken)}"
    
    await save_data()
    
    # ОТВЕТ БОТА (УБРАНО УНИЧТОЖЕНИЕ МАШИНЫ!)
    if success:
        await context.bot.send_message(chat_id,
            f"🎉 ПЕРЕХВАТ УСПЕШЕН! 🎉\n📊 Категория: {EF_SCALE[ef_choice]['name']}\n💨 Скорость ветра: {wind_speed} mph\n🚗 Машина: {car_data['name']}\n{reward_text}{probes_text}\n🛡️ Прочность машины: {new_hp}/{car_data.get('hp', 100)}\n💰 Баланс: ${user['balance']:,}"
        )
    else:
        await context.bot.send_message(chat_id,
            f"💨 ВАС УНЕСЛО ТОРНАДО! Машина улетела! 💨\n📊 Категория: {EF_SCALE[ef_choice]['name']}\n💨 Скорость ветра: {wind_speed} mph\n🚗 Машина: {car_data['name']} (выносливость {endurance_text})\n{reward_text}{probes_text}\n🛡️ Прочность машины: {new_hp}/{car_data.get('hp', 100)}\n💰 Баланс: ${user['balance']:,}"
        )
    
    sessions = load_sessions()
    if session_id in sessions: del sessions[session_id]; save_sessions(sessions)

# ==================== ЗАПУСК БОТА ====================
bot_instance = None

async def run_bot():
    global bot_instance
    load_data()
    logger.info(f'Загружено {len(user_data)} пользователей')
    
    request = HTTPXRequest(connect_timeout=30.0, read_timeout=30.0, pool_timeout=30.0)
    app = ApplicationBuilder().token(TOKEN).request(request).build()
    bot_instance = app

    asyncio.create_task(auto_save_loop())
    asyncio.create_task(breakdown_check_loop())

    # Регистрация команд
    app.add_handler(CommandHandler('start', start))
    app.add_handler(CommandHandler('help', help_command))
    app.add_handler(CommandHandler('profile', profile))
    app.add_handler(CommandHandler('balance', balance))
    app.add_handler(CommandHandler('shop_intercept', shop_intercept))
    app.add_handler(CommandHandler('buy_intercept', buy_intercept))
    app.add_handler(CommandHandler('my_interceptors', my_interceptors))
    app.add_handler(CommandHandler('my_car', my_car))
    app.add_handler(CommandHandler('wear', wear_car))
    app.add_handler(CommandHandler('give_money', give_money))
    app.add_handler(CommandHandler('take_money', take_money))
    
    app.add_handler(CommandHandler('intercept', intercept))
    app.add_handler(CallbackQueryHandler(intercept_callback, pattern='^intercept_'))
    
    app.add_handler(CommandHandler('shop_probs', shop_probs))
    app.add_handler(CommandHandler('buy_prob', buy_prob))
    app.add_handler(CommandHandler('probs', probs))
    
    app.add_handler(CommandHandler('shop_radars', shop_radars))
    app.add_handler(CommandHandler('buy_radar', buy_radar))
    app.add_handler(CommandHandler('radars', radars))
    app.add_handler(CommandHandler('wear_radar', wear_radar))
    
    app.add_handler(CommandHandler('save', lambda u,c: u.message.reply_text('✅ Данные сохранены') if is_owner_or_creator(u.effective_user.id) else u.message.reply_text('🚫 Нет прав')))
    app.add_handler(CommandHandler('sync', lambda u,c: bot_chats.add(str(u.effective_chat.id)) or u.message.reply_text('✅ Чат синхронизирован') if is_owner_or_creator(u.effective_user.id) else u.message.reply_text('🚫 Нет прав')))
    
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, handle_all_messages))
    
    logger.info("🚀 Бот запущен!")
    await app.initialize()
    await app.start()
    await app.updater.start_polling()
    
    # Обработка остановки
    stop_signal = asyncio.Future()
    def handle_stop():
        if not stop_signal.done(): stop_signal.set_result(True)
    try:
        loop = asyncio.get_running_loop()
        loop.add_signal_handler(signal.SIGINT, handle_stop)
        loop.add_signal_handler(signal.SIGTERM, handle_stop)
    except NotImplementedError: pass
        
    try: await stop_signal
    except (KeyboardInterrupt, asyncio.CancelledError): pass
    finally:
        await save_data()
        await app.updater.stop()
        await app.stop()
        await app.shutdown()
        logger.info("👋 Бот остановлен. Данные сохранены.")

# ==================== ОБРАБОТЧИК ВСЕХ СООБЩЕНИЙ ====================
async def handle_all_messages(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not update.message: return
    if update.message.date.timestamp() < BOT_START_TIME: return
    if update.message.date < datetime.datetime.now() - datetime.timedelta(seconds=5): return
    
    user_id = str(update.effective_user.id)
    ensure_user_exists(user_id, update.effective_user.full_name, update.effective_user.username)

# ==================== БЕСКОНЕЧНЫЙ ЦИКЛ ЗАПУСКА (ДЛЯ ХОСТИНГА) ====================
if __name__ == '__main__':
    while True:
        try:
            print("🚀 Запуск бота...")
            asyncio.run(run_bot())
        except KeyboardInterrupt:
            logger.info("Бот остановлен вручную")
            break
        except Exception as e:
            logger.error(f"⚠️ Бот упал с ошибкой: {e}. Перезапуск через 5 секунд...")
            time.sleep(5)

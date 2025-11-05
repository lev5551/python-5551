import logging
import random
from datetime import datetime, timedelta
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import Application, CommandHandler, ContextTypes, CallbackQueryHandler
from sqlalchemy import create_engine, Column, Integer, String, DateTime, Float, ForeignKey, func
from sqlalchemy.orm import declarative_base, sessionmaker, relationship

BOT_TOKEN = "8455342253:AAGWPJvGe9LHTXCD2yGTtp9FJ6fBBOhSjwY"

logging.basicConfig(format='%(asctime)s - %(name)s - %(levelname)s - %(message)s', level=logging.INFO)
logger = logging.getLogger(__name__)

Base = declarative_base()

class User(Base):
    __tablename__ = 'users'
    user_id = Column(Integer, primary_key=True)
    username = Column(String)
    language = Column(String, default='en')
    created_at = Column(DateTime, default=datetime.utcnow)
    user_words = relationship("UserWord", back_populates="user")

class Word(Base):
    __tablename__ = 'words'
    word_id = Column(Integer, primary_key=True, autoincrement=True)
    language = Column(String)
    word = Column(String, nullable=False)
    translation = Column(String, nullable=False)
    part_of_speech = Column(String)
    user_words = relationship("UserWord", back_populates="word")

class UserWord(Base):
    __tablename__ = 'user_words'
    user_id = Column(Integer, ForeignKey('users.user_id'), primary_key=True)
    word_id = Column(Integer, ForeignKey('words.word_id'), primary_key=True)
    interval = Column(Integer, default=0)
    repetitions = Column(Integer, default=0)
    ease_factor = Column(Float, default=2.5)
    next_review = Column(DateTime, default=datetime.utcnow)
    user = relationship("User", back_populates="user_words")
    word = relationship("Word", back_populates="user_words")

engine = create_engine('sqlite:///language_bot.db')
Base.metadata.create_all(engine)
Session = sessionmaker(bind=engine)

def init_words():
    sample_words = [
        {'word': 'hello', 'translation': 'привет', 'language': 'en'},
        {'word': 'goodbye', 'translation': 'пока', 'language': 'en'},
        {'word': 'thank you', 'translation': 'спасибо', 'language': 'en'},
        {'word': 'please', 'translation': 'пожалуйста', 'language': 'en'},
        {'word': 'yes', 'translation': 'да', 'language': 'en'},
        {'word': 'no', 'translation': 'нет', 'language': 'en'},
        {'word': 'water', 'translation': 'вода', 'language': 'en'},
        {'word': 'food', 'translation': 'еда', 'language': 'en'},
        {'word': 'house', 'translation': 'дом', 'language': 'en'},
        {'word': 'car', 'translation': 'машина', 'language': 'en'}
    ]
    
    with Session() as session:
        for word_data in sample_words:
            if not session.query(Word).filter_by(word=word_data['word']).first():
                word = Word(**word_data)
                session.add(word)
        session.commit()

async def get_or_create_user(user_id: int, username: str):
    with Session() as session:
        user = session.query(User).filter(User.user_id == user_id).first()
        if not user:
            user = User(user_id=user_id, username=username)
            session.add(user)
            session.commit()
        return user

async def get_next_word_for_user(user_id: int):
    with Session() as session:
        next_word = (session.query(Word)
                    .join(UserWord, (UserWord.word_id == Word.word_id) & (UserWord.user_id == user_id))
                    .filter(UserWord.next_review <= datetime.utcnow())
                    .first())
        
        if not next_word:
            learned_word_ids = session.query(UserWord.word_id).filter(UserWord.user_id == user_id).all()
            learned_word_ids = [id[0] for id in learned_word_ids]
            next_word = (session.query(Word)
                        .filter(~Word.word_id.in_(learned_word_ids) if learned_word_ids else True)
                        .order_by(func.random())
                        .first())
        return next_word

async def update_srs_stats(user_id: int, word_id: int, quality: int):
    with Session() as session:user_word = session.query(UserWord).filter_by(user_id=user_id, word_id=word_id).first()
    if not user_word:
            user_word = UserWord(user_id=user_id, word_id=word_id)
            session.add(user_word)

    if quality >= 3:
            if user_word.repetitions == 0:
                user_word.interval = 1
            elif user_word.repetitions == 1:
                user_word.interval = 6
            else:
                user_word.interval = round(user_word.interval * user_word.ease_factor)
            
            user_word.repetitions += 1
            user_word.ease_factor = max(1.3, user_word.ease_factor + (0.1 - (5 - quality) * (0.08 + (5 - quality) * 0.02)))
    else:
            user_word.repetitions = 0
            user_word.interval = 1

    user_word.next_review = datetime.utcnow() + timedelta(days=user_word.interval)
    session.commit()

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user = update.effective_user
    await get_or_create_user(user.id, user.username)
    
    welcome_text = f"""
Привет, {user.first_name}! 👋
Я бот для тренировки словарного запаса.

Доступные команды:
/train - Начать тренировку
/stats - Показать статистику
"""
    await update.message.reply_text(welcome_text)

async def train_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    word = await get_next_word_for_user(user_id)
    
    if not word:
        await update.message.reply_text("Поздравляю! Вы выучили все слова в базе. 🎉")
        return

    context.user_data['current_word'] = word
    
    with Session() as session:
        wrong_translations = (session.query(Word.translation)
                             .filter(Word.word_id != word.word_id, Word.language == word.language)
                             .order_by(func.random())
                             .limit(3)
                             .all())
        wrong_translations = [t[0] for t in wrong_translations]
    
    choices = [word.translation] + wrong_translations
    random.shuffle(choices)
    
    keyboard = []
    for choice in choices:
        keyboard.append([InlineKeyboardButton(choice, callback_data=f"choice_{word.word_id}_{choice}")])
    
    reply_markup = InlineKeyboardMarkup(keyboard)
    await update.message.reply_text(f"Как переводится слово:\n\n*{word.word}*", reply_markup=reply_markup, parse_mode='Markdown')

async def handle_choice(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()
    
    user_id = query.from_user.id
    data = query.data.split('_')
    word_id = int(data[1])
    selected_translation = '_'.join(data[2:])

    print(f"DEBUG: user_id={user_id}, word_id={word_id}, selected='{selected_translation}'")  # Для отладки

    with Session() as session:
        word = session.query(Word).filter(Word.word_id == word_id).first()
    
    if not word:
        await query.edit_message_text("Произошла ошибка. Слово не найдено.")
        return

    print(f"DEBUG: correct translation='{word.translation}'")  
    
    is_correct = selected_translation.strip() == word.translation.strip()
    quality = 5 if is_correct else 0
    
    print(f"DEBUG: is_correct={is_correct}")  
    await update_srs_stats(user_id, word_id, quality)
    
    if is_correct:
        message = "✅ Верно! Отличная работа!"
    else:
        message = f"❌ Неверно. Правильный ответ: *{word.translation}*"
    
    next_button = [[InlineKeyboardButton("➡️ Следующее слово", callback_data=f"next_{user_id}")]]
    reply_markup = InlineKeyboardMarkup(next_button)
    
    await query.edit_message_text(
        f"{message}\n\nСлово: *{word.word}*\nПеревод: *{word.translation}*", 
        reply_markup=reply_markup, 
        parse_mode='Markdown'
    )       

async def handle_next(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()
    
    user_id = int(query.data.split('_')[1])
    
    word = await get_next_word_for_user(user_id)
    if not word:
        await query.edit_message_text("Поздравляю! Вы выучили все слова на сегодня. 🎉")
        return
    
    with Session() as session:
        wrong_translations = (session.query(Word.translation)
                             .filter(Word.word_id != word.word_id, Word.language == word.language)
                             .order_by(func.random())
                             .limit(3)
                             .all())
        wrong_translations = [t[0] for t in wrong_translations]
    
    choices = [word.translation] + wrong_translations
    random.shuffle(choices)
    
    keyboard = []
    for choice in choices:
        keyboard.append([InlineKeyboardButton(choice, callback_data=f"choice_{word.word_id}_{choice}")])
    
    reply_markup = InlineKeyboardMarkup(keyboard)
    await query.edit_message_text(
        f"Как переводится слово:\n\n*{word.word}*", 
        reply_markup=reply_markup, 
        parse_mode='Markdown'
    )


async def stats_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    
    with Session() as session:
        total_words = session.query(Word).count()
        learned_words = session.query(UserWord).filter(UserWord.user_id == user_id).count()
        due_for_review = (session.query(UserWord)
                         .filter(UserWord.user_id == user_id, UserWord.next_review <= datetime.utcnow())
                         .count())
    
    stats_text = f"""
📊 Ваша статистика:

Всего слов в базе: {total_words}
Вы начали учить: {learned_words}
Слов к повторению сегодня: {due_for_review}
"""
    await update.message.reply_text(stats_text)

def main():
    init_words()
    application = Application.builder().token(BOT_TOKEN).build()
    
    application.add_handler(CommandHandler("start", start))
    application.add_handler(CommandHandler("train", train_command))
    application.add_handler(CommandHandler("stats", stats_command))
    application.add_handler(CallbackQueryHandler(handle_choice, pattern="^choice_"))
    application.add_handler(CallbackQueryHandler(handle_next, pattern="^next_"))
    
    application.run_polling()

if __name__== "__main__":
    main()

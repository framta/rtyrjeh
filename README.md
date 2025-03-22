import asyncio
import random
import logging
import feedparser
import pandas as pd
import json
import os
from datetime import datetime
from pathlib import Path

from telegram import Update
from telegram.ext import (
    ApplicationBuilder,
    CommandHandler,
    MessageHandler,
    ContextTypes,
    filters,
)
from telegram.constants import ParseMode
from ta.momentum import RSIIndicator, StochasticOscillator
from ta.trend import MACD, EMAIndicator, SMAIndicator

from pybit.unified_trading import HTTP

TOKEN = "7934246565:AAFIM_VD2qsLvTnC0uRAqKqHL3qJk0iA3g8"
RSS_FEED = "https://ru.investing.com/rss/news_301.rss"
chat_ids = set()
session = HTTP(testnet=False)

logging.basicConfig(level=logging.INFO)

# ========== Новости ==========
async def send_latest_news(context: ContextTypes.DEFAULT_TYPE):
    feed = feedparser.parse(RSS_FEED)
    if not feed.entries:
        return

    news = feed.entries[0]
    title = news.get("title", "Без заголовка")
    link = news.get("link", "")
    image_url = None
    media = news.get("media_content", [])
    if media and isinstance(media, list):
        image_url = media[0].get("url")

    message_text = (
        f"<b>📰 {title}</b>\n\n"
        f"<a href=\"{link}\">🔗 Подробнее</a>"
    )

    for chat_id in chat_ids:
        if image_url:
            await context.bot.send_photo(
                chat_id=chat_id,
                photo=image_url,
                caption=message_text,
                parse_mode=ParseMode.HTML,
                disable_web_page_preview=True
            )
        else:
            await context.bot.send_message(
                chat_id=chat_id,
                text=message_text,
                parse_mode=ParseMode.HTML,
                disable_web_page_preview=True
            )

async def news_loop(app):
    while True:
        delay = random.randint(5 * 60, 55 * 60)
        await asyncio.sleep(delay)
        await send_latest_news(app)

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    chat_id = update.effective_chat.id
    chat_ids.add(chat_id)
    await update.message.reply_text(
        "✅ Я буду присылать тебе крипто-новости 📈\n"
        "\n<b>Доступные команды:</b>\n"
        "/analyze — анализ криптовалюты\n"
        "/help — помощь по командам",
        parse_mode=ParseMode.HTML
    )
    await send_latest_news(context)

# ========== Команда /analyze ==========
async def analyze_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text("🔍 Введите тикер криптовалюты")
    context.user_data["awaiting_symbol"] = True

async def handle_symbol_input(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not context.user_data.get("awaiting_symbol"):
        return

    symbol = update.message.text.strip().upper()
    full_pair = f"{symbol}USDT"

    try:
        instruments = session.get_instruments_info(category="linear")
        valid_symbols = {item["symbol"] for item in instruments["result"]["list"]}

        if full_pair not in valid_symbols:
            await update.message.reply_text(f"⚠️ Пара {full_pair} не найдена на бирже.")
            return

        data = session.get_kline(
            category="linear",
            symbol=full_pair,
            interval=15,
            limit=100
        )["result"]["list"]

        if not data:
            raise ValueError("Нет данных по выбранной паре.")

        df = pd.DataFrame(data, columns=[
            "timestamp", "open", "high", "low", "close", "volume", "_"
        ])
        df = df.iloc[::-1]
        for col in ["open", "high", "low", "close", "volume"]:
            df[col] = pd.to_numeric(df[col])

        df["timestamp"] = pd.to_datetime(df["timestamp"], unit="ms")
        df.set_index("timestamp", inplace=True)

        close = df["close"]
        high = df["high"]
        low = df["low"]

        rsi_val = RSIIndicator(close).rsi().iloc[-1]
        macd_val = MACD(close).macd_diff().iloc[-1]
        ema20 = EMAIndicator(close, window=20).ema_indicator().iloc[-1]
        sma50 = SMAIndicator(close, window=50).sma_indicator().iloc[-1]
        stoch = StochasticOscillator(high=high, low=low, close=close).stoch().iloc[-1]

        latest_price = close.iloc[-1]
        lines = [
            f"📊 <b>Анализ для {full_pair}</b>",
            f"💰 Цена: <b>{latest_price:.2f}</b> USDT",
            f"📈 RSI: <b>{rsi_val:.1f}</b> — {'Покупка' if rsi_val > 60 else 'Продажа' if rsi_val < 40 else 'Нейтрально'}",
            f"📉 MACD: <b>{macd_val:.4f}</b> — {'Бычий' if macd_val > 0 else 'Медвежий'} сигнал",
            f"🔹 EMA(20): <b>{ema20:.2f}</b>",
            f"🔹 SMA(50): <b>{sma50:.2f}</b>",
            f"🔄 Stochastic: <b>{stoch:.1f}</b> — {'Перекуплен' if stoch > 80 else 'Перепродан' if stoch < 20 else 'Нейтрально'}",
        ]

        result = "\n".join(lines)
        await update.message.reply_text(result, parse_mode=ParseMode.HTML)

    except Exception as e:
        await update.message.reply_text(f"⚠️ Ошибка при получении данных по {full_pair}.\n{e}")

    context.user_data["awaiting_symbol"] = False

# ========== /help ==========
async def help_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    help_text = (
        "<b>🤖 Команды и описание:</b>\n\n"
        "/start — запустить бота, получить последнюю новость и список команд\n"
        "/analyze — ввести тикер криптовалюты и получить технический анализ\n"
        "/help — показать это сообщение помощи\n\n"
        "💬 Связь с админом: @O2llO7"
    )
    await update.message.reply_text(help_text, parse_mode=ParseMode.HTML)

# ========== Main ==========
async def main():
    app = ApplicationBuilder().token(TOKEN).build()
    app.add_handler(CommandHandler("start", start))
    app.add_handler(CommandHandler("analyze", analyze_command))
    app.add_handler(CommandHandler("help", help_command))
    app.add_handler(MessageHandler(filters.TEXT & (~filters.COMMAND), handle_symbol_input))

    print("Бот запущен...")
    await app.initialize()
    await app.start()
    asyncio.create_task(news_loop(app))
    await app.updater.start_polling()
    await asyncio.Event().wait()

if __name__ == '__main__':
    import sys
    if sys.platform.startswith("win") and sys.version_info >= (3, 8):
        asyncio.set_event_loop_policy(asyncio.WindowsSelectorEventLoopPolicy())

    asyncio.run(main())

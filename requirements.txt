import os
import logging
import aiohttp
from telegram import Update
from telegram.ext import Application, CommandHandler, MessageHandler, filters, ContextTypes

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

BOT_TOKEN = os.environ.get("BOT_TOKEN")
API_BASE = f"https://api.telegram.org/bot{BOT_TOKEN}"


async def get_user_info(user_id: int):
    async with aiohttp.ClientSession() as session:
        async with session.get(f"{API_BASE}/getChat?chat_id={user_id}") as resp:
            if resp.status != 200:
                return None
            data = await resp.json()
            if not data.get("ok"):
                return None
            return data["result"]


async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text(
        "سلام 👋\n"
        "یه آیدی عددی تلگرام بفرست تا اطلاعاتش رو بهت نشون بدم.\n\n"
        "مثال: `123456789`"
    )


async def handle_id(update: Update, context: ContextTypes.DEFAULT_TYPE):
    text = update.message.text.strip()

    if not text.lstrip("-").isdigit():
        await update.message.reply_text("لطفاً فقط عدد بفرست.")
        return

    user_id = int(text)
    info = await get_user_info(user_id)

    if not info:
        await update.message.reply_text("❌ این آیدی معتبر نیست یا اکانت حذف شده.")
        return

    name = info.get("first_name", "")
    last = info.get("last_name", "")
    username = info.get("username")
    bio = info.get("bio", "—")

    full_name = f"{name} {last}".strip()
    profile_link = f"tg://user?id={user_id}"

    msg = (
        f"✅ اطلاعات پیدا شد:\n\n"
        f"👤 نام: {full_name or '—'}\n"
        f"🆔 آیدی: `{user_id}`\n"
    )
    if username:
        msg += f"🔗 یوزرنیم: @{username}\n"
    msg += f"📝 بیو: {bio}\n"
    msg += f"\n🔍 [باز کردن پروفایل]({profile_link})"

    photos = info.get("photo")
    if photos and photos.get("big_file_id"):
        await update.message.reply_photo(photos["big_file_id"], caption=msg, parse_mode="Markdown")
    else:
        await update.message.reply_text(msg, parse_mode="Markdown", disable_web_page_preview=True)


def main():
    app = Application.builder().token(BOT_TOKEN).build()
    app.add_handler(CommandHandler("start", start))
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, handle_id))
    app.run_polling()


if __name__ == "__main__":
    main()

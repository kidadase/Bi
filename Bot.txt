import os
import logging
from flask import Flask, request, Response
from telegram import Update
from telegram.ext import ApplicationBuilder, CommandHandler, MessageHandler, ContextTypes, filters

# --- Configuration ---
# Get token from environment variables (set securely on Render)
BOT_TOKEN = os.environ.get("BOT_TOKEN")
# The webhook URL is the base public URL of your Render service
WEBHOOK_URL = os.environ.get("WEBHOOK_URL") 
# The path Telegram will post updates to (must match setWebhook URL)
WEBHOOK_PATH = "/webhooks/telegram/action" 

# --- Setup Logging ---
logging.basicConfig(format='%(asctime)s - %(name)s - %(levelname)s - %(message)s', level=logging.INFO)

# --- Handlers ---
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Replies to the /start command."""
    await update.message.reply_text("Selam Ahmed 👋 Your bot is active and ready (via webhook)!")

async def echo(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Echoes the user's text message."""
    user_message = update.message.text
    if user_message:
        await update.message.reply_text(f"You said: {user_message}")

# --- Initialize Application (Required once) ---
application = ApplicationBuilder().token(BOT_TOKEN).build()
application.add_handler(CommandHandler("start", start))
application.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, echo))

# --- Flask Server Setup ---
app = Flask(__name__)

@app.route("/")
def index():
    """A basic endpoint to confirm the server is running."""
    return "Bot Webhook Listener is Live!"

@app.route(WEBHOOK_PATH, methods=["POST"])
async def telegram_webhook():
    """Endpoint for Telegram to send updates to."""
    if request.method == "POST":
        # 1. Get the JSON update from Telegram
        update = Update.de_json(request.get_json(force=True), application.bot)
        
        # 2. Process the update asynchronously
        await application.process_update(update)
        
        # 3. Return a successful HTTP status code immediately
        return Response(status=200)

    # Return 405 Method Not Allowed for other requests
    return Response(status=405)

# --- Startup Function (Sets the webhook URL on Telegram) ---
# This is typically run once after deployment, NOT continuously.
async def set_telegram_webhook():
    if WEBHOOK_URL and BOT_TOKEN:
        webhook_full_url = f"{WEBHOOK_URL.rstrip('/')}{WEBHOOK_PATH}"
        
        # Ensure we are stopping polling and setting the webhook
        await application.bot.delete_webhook()
        await application.bot.set_webhook(url=webhook_full_url)
        logging.info(f"Webhook successfully set to: {webhook_full_url}")
    else:
        logging.error("BOT_TOKEN or WEBHOOK_URL environment variables are missing.")

# --- Server Start ---
if __name__ == "__main__":
    import asyncio
    # Start the webhook configuration task
    asyncio.run(set_telegram_webhook())
    
    # Run the Flask app using Gunicorn (or Flask's simple server for local testing)
    # NOTE: Render will use the Procfile/Start Command, not this line.
    app.run(host="0.0.0.0", port=int(os.environ.get('PORT', 5000))) 

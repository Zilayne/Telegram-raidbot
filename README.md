import asyncio
import random
from aiogram import Bot, Dispatcher, types
from aiogram.filters import Command
from aiogram.types import Message
from aiogram.enums import ParseMode
from aiogram.client.default import DefaultBotProperties
from aiogram.exceptions import TelegramRetryAfter
from loguru import logger

# Убедитесь, что у вас есть файл config.py с этими переменными / Make sure you have a config.py file with these variables
import config

class RaidBot:
    def __init__(self, token):
        self.token = token
        self.bot = Bot(token=self.token, default=DefaultBotProperties(parse_mode=ParseMode.HTML))
        self.dp = Dispatcher()
        self.is_active = False  # Флаг для контроля рассылки / Flag to control the broadcast

    async def on_startup(self):
        me = await self.bot.get_me()
        username = me.username
        logger.info(f"@{username} is ready to attack!")

    def start(self):
        @self.dp.message(Command("raid"))
        async def start_raid(message: Message):
            if message.from_user.id == config.ADMIN_ID:
                self.is_active = True  # Включаем рассылку / Enable the broadcast
                logger.info("Raid start!")
                for _ in range(config.COUNT):
                    if not self.is_active:  # Если рассылка остановлена, выходим из цикла / If the broadcast is stopped, exit the loop
                        break
                    try:
                        await message.answer(random.choice(config.MESSAGES))
                        await asyncio.sleep(config.DELAY)  # Задержка между сообщениями / Delay between messages
                    except TelegramRetryAfter as e:
                        # Если превышен лимит, ждем указанное время / If the rate limit is exceeded, wait for the specified time
                        logger.warning(f"Flood control exceeded. Retry in {e.retry_after} seconds.")
                        await asyncio.sleep(e.retry_after)
                self.is_active = False  # Выключаем рассылку после завершения цикла / Disable the broadcast after the loop ends
                logger.info("Raid is over!")

        @self.dp.message(Command("stop"))
        async def stop_raid(message: Message):
            if message.from_user.id == config.ADMIN_ID:
                if self.is_active:
                    self.is_active = False  # Останавливаем рассылку / Stop the broadcast
                    logger.info("Raid stopped by /stop.")
                    await message.answer("Raid is over!")
                else:
                    await message.answer("ERROR Raid already stopped!")

        async def main():
            await self.on_startup()
            await self.dp.start_polling(self.bot)

        asyncio.run(main())

if __name__ == '__main__':
    if config.ADMIN_ID == 1:
        logger.error("Check config. ADMIN_ID should be your Telegram ID.")
        exit(1)

    for token in config.TOKENS:
        bot = RaidBot(token)
        bot.start()

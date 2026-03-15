import os
import random
import discord
from discord.ext import commands, tasks
from datetime import datetime, timedelta
import asyncio
import json
from collections import defaultdict
import sys
from pathlib import Path
import aiohttp
import math
from typing import Optional, List, Dict, Any
import re
import time
import threading
import logging
from functools import wraps
import traceback

# Настройка логирования
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    handlers=[
        logging.FileHandler('bot.log', encoding='utf-8'),
        logging.StreamHandler()
    ]
)
logger = logging.getLogger(__name__)

# ==================== КОНФИГУРАЦИЯ ====================
# ⚠️ ВАЖНО: Замените на ваш реальный токен!
TOKEN = 'ваш токен бота'
CREATOR_ID = айди создателя бота  # Ваш ID
PREFIX = 'префикс бота'
BOT_COLOR = 0x5865F2
WEBSITE_PORT = 5000
INVITE_URL = "ссылка на приглашение бота"

# ==================== СИСТЕМА ДАННЫХ ====================
class MultiServerDataManager:
    """Управление данными для нескольких серверов"""
    
    def __init__(self):
        self.data_dir = Path("bot_data")
        self.data_dir.mkdir(exist_ok=True)
        
        # Серверные данные
        self.servers_dir = self.data_dir / "servers"
        self.servers_dir.mkdir(exist_ok=True)
        
        # Кэширование данных
        self.cache = {}
        
    def _get_cache_key(self, guild_id, filename):
        return f"{guild_id}_{filename}"
    
    def get_server_file(self, guild_id: int, filename: str) -> Path:
        """Получить путь к файлу данных сервера"""
        server_dir = self.servers_dir / str(guild_id)
        server_dir.mkdir(exist_ok=True)
        return server_dir / filename
    
    def save_server_data(self, guild_id: int, filename: str, data: Any) -> bool:
        """Сохранение данных сервера"""
        try:
            filepath = self.get_server_file(guild_id, filename)
            # Обновляем кэш
            cache_key = self._get_cache_key(guild_id, filename)
            self.cache[cache_key] = data.copy() if hasattr(data, 'copy') else data
            
            with open(filepath, 'w', encoding='utf-8') as f:
                json.dump(data, f, ensure_ascii=False, indent=2, default=str)
            logger.info(f"Сохранены данные сервера {guild_id}/{filename}")
            return True
        except Exception as e:
            logger.error(f"Ошибка сохранения данных сервера {guild_id}/{filename}: {e}")
            return False
    
    def load_server_data(self, guild_id: int, filename: str, default: Any = None) -> Any:
        """Загрузка данных сервера с кэшированием"""
        cache_key = self._get_cache_key(guild_id, filename)
        
        # Проверяем кэш
        if cache_key in self.cache:
            return self.cache[cache_key]
        
        try:
            filepath = self.get_server_file(guild_id, filename)
            if filepath.exists():
                with open(filepath, 'r', encoding='utf-8') as f:
                    data = json.load(f)
                    self.cache[cache_key] = data
                    return data
        except Exception as e:
            logger.error(f"Ошибка загрузки данных сервера {guild_id}/{filename}: {e}")
        
        return default or {}
    
    # Серверные конфигурации
    def save_server_config(self, guild_id: int, config: Dict) -> bool:
        return self.save_server_data(guild_id, "config.json", config)
    
    def load_server_config(self, guild_id: int) -> Dict:
        config = self.load_server_data(guild_id, "config.json", {})
        # Устанавливаем значения по умолчанию
        defaults = {
            'prefix': PREFIX,
            'welcome_channel': None,
            'log_channel': None,
            'mute_role': None,
            'level_up_channel': None,
            'admin_roles': [],
            'commands_used': 0,
            'welcome_message': "Добро пожаловать, {user}! Рады видеть тебя на сервере {server}!",
            'auto_roles': [],
            'xp_cooldown': 60,  # Секунд между получением опыта
            'last_xp_gain': {}
        }
        
        for key, value in defaults.items():
            if key not in config:
                config[key] = value
                
        return config
    
    # Серверные варны
    def save_server_warns(self, guild_id: int, warns: Dict) -> bool:
        return self.save_server_data(guild_id, "warns.json", warns)
    
    def load_server_warns(self, guild_id: int) -> Dict:
        return self.load_server_data(guild_id, "warns.json", {})
    
    # Серверные уровни
    def save_server_levels(self, guild_id: int, levels: Dict) -> bool:
        return self.save_server_data(guild_id, "levels.json", levels)
    
    def load_server_levels(self, guild_id: int) -> Dict:
        return self.load_server_data(guild_id, "levels.json", {})
    
    # Глобальная статистика
    def save_global_stats(self, stats: Dict) -> bool:
        try:
            filepath = self.data_dir / "global_stats.json"
            with open(filepath, 'w', encoding='utf-8') as f:
                json.dump(stats, f, ensure_ascii=False, indent=2, default=str)
            return True
        except Exception as e:
            logger.error(f"Ошибка сохранения глобальной статистики: {e}")
            return False
    
    def load_global_stats(self) -> Dict:
        try:
            filepath = self.data_dir / "global_stats.json"
            if filepath.exists():
                with open(filepath, 'r', encoding='utf-8') as f:
                    return json.load(f)
        except Exception as e:
            logger.error(f"Ошибка загрузки глобальной статистики: {e}")
        
        return {
            'total_messages': 0,
            'total_commands': 0,
            'total_servers': 0,
            'total_users': 0,
            'start_time': datetime.now().isoformat(),
            'commands_by_guild': {},
            'messages_by_guild': {}
        }

# ==================== ИНИЦИАЛИЗАЦИЯ БОТА ====================
intents = discord.Intents.default()
intents.messages = True
intents.guilds = True
intents.members = True
intents.message_content = True
intents.reactions = True
intents.voice_states = True

class MultiServerBot(commands.Bot):
    def __init__(self):
        super().__init__(
            command_prefix=self.get_prefix,
            intents=intents,
            help_command=None,
            case_insensitive=True,
            owner_id=CREATOR_ID
        )
        self.start_time = datetime.now()
        self.data_manager = MultiServerDataManager()
        self.server_warns = defaultdict(lambda: defaultdict(list))
        self.server_levels = defaultdict(dict)
        self.server_configs = {}
        self.global_stats = {}
        
    async def get_prefix(self, message):
        """Динамическое получение префикса для сервера"""
        if not message.guild:
            return PREFIX
        
        guild_id = message.guild.id
        if guild_id not in self.server_configs:
            config = self.data_manager.load_server_config(guild_id)
            self.server_configs[guild_id] = config
        
        return self.server_configs[guild_id].get('prefix', PREFIX)

bot = MultiServerBot()

# ==================== ДЕКОРАТОРЫ И УТИЛИТЫ ====================
def admin_only():
    """Декоратор для команд, доступных только администраторам"""
    async def predicate(ctx):
        if ctx.author.guild_permissions.administrator:
            return True
        raise commands.MissingPermissions(['administrator'])
    return commands.check(predicate)

def create_progress_bar(value: int, max_value: int, length: int = 20) -> str:
    """Создание прогресс-бара"""
    percentage = value / max_value
    filled = int(length * percentage)
    bar = "█" * filled + "░" * (length - filled)
    return f"{bar} {percentage*100:.1f}%"

def format_time(seconds: int) -> str:
    """Форматирование времени"""
    days = seconds // 86400
    hours = (seconds % 86400) // 3600
    minutes = (seconds % 3600) // 60
    secs = seconds % 60
    
    parts = []
    if days > 0:
        parts.append(f"{days}д")
    if hours > 0:
        parts.append(f"{hours}ч")
    if minutes > 0:
        parts.append(f"{minutes}м")
    if secs > 0 or not parts:
        parts.append(f"{secs}с")
    
    return " ".join(parts)

# ==================== КОМАНДЫ АДМИНИСТРАЦИИ ====================
@bot.command(name='prefix', aliases=['префикс'])
@admin_only()
async def set_prefix(ctx, new_prefix: str):
    """Изменить префикс бота на сервере"""
    if len(new_prefix) > 3:
        embed = discord.Embed(
            title="❌ Ошибка",
            description="Префикс не может быть длиннее 3 символов!",
            color=discord.Color.red()
        )
        await ctx.send(embed=embed)
        return
    
    guild_id = ctx.guild.id
    config = bot.server_configs[guild_id]
    old_prefix = config.get('prefix', PREFIX)
    config['prefix'] = new_prefix
    
    bot.data_manager.save_server_config(guild_id, config)
    
    embed = discord.Embed(
        title="✅ Префикс обновлен",
        description=f"Старый префикс: `{old_prefix}`\nНовый префикс: `{new_prefix}`",
        color=discord.Color.green()
    )
    await ctx.send(embed=embed)

@bot.command(name='clear', aliases=['очистить', 'удалить'])
@commands.has_permissions(manage_messages=True)
async def clear_messages(ctx, amount: int = 10):
    """Очистить указанное количество сообщений"""
    amount = min(amount, 100)  # Ограничение
    
    if amount <= 0:
        embed = discord.Embed(
            title="❌ Ошибка",
            description="Укажите число больше 0!",
            color=discord.Color.red()
        )
        await ctx.send(embed=embed)
        return
    
    deleted = await ctx.channel.purge(limit=amount + 1)  # +1 для команды
    
    embed = discord.Embed(
        title="🧹 Очистка завершена",
        description=f"Удалено {len(deleted)-1} сообщений!",
        color=discord.Color.green()
    )
    msg = await ctx.send(embed=embed, delete_after=5)

@bot.command(name='mute', aliases=['мут'])
@commands.has_permissions(manage_roles=True)
async def mute_user(ctx, member: discord.Member, duration: str = "30m", *, reason: str = "Не указана"):
    """Замутить пользователя на указанное время"""
    # Парсим время
    time_units = {'s': 1, 'm': 60, 'h': 3600, 'd': 86400}
    duration_value = int(duration[:-1])
    duration_unit = duration[-1].lower()
    
    if duration_unit not in time_units:
        await ctx.send("❌ Неверный формат времени! Используйте: 30s, 10m, 1h, 1d")
        return
    
    seconds = duration_value * time_units[duration_unit]
    
    # Получаем роль мута
    config = bot.server_configs[ctx.guild.id]
    mute_role_id = config.get('mute_role')
    
    if not mute_role_id:
        # Создаем роль мута
        mute_role = await ctx.guild.create_role(
            name="Muted",
            color=discord.Color.dark_gray(),
            reason="Создание роли для мута"
        )
        
        # Настраиваем права
        for channel in ctx.guild.channels:
            await channel.set_permissions(
                mute_role,
                speak=False,
                send_messages=False,
                add_reactions=False
            )
        
        mute_role_id = mute_role.id
        config['mute_role'] = mute_role_id
        bot.data_manager.save_server_config(ctx.guild.id, config)
    
    mute_role = ctx.guild.get_role(mute_role_id)
    await member.add_roles(mute_role, reason=reason)
    
    # Создаем embed
    embed = discord.Embed(
        title="🔇 Пользователь замьючен",
        color=discord.Color.orange()
    )
    embed.add_field(name="👤 Пользователь", value=member.mention, inline=True)
    embed.add_field(name="⏱️ Длительность", value=format_time(seconds), inline=True)
    embed.add_field(name="👮 Модератор", value=ctx.author.mention, inline=True)
    embed.add_field(name="📝 Причина", value=reason, inline=False)
    embed.set_thumbnail(url=member.display_avatar.url)
    
    await ctx.send(embed=embed)
    
    # Автоматическое снятие мута
    await asyncio.sleep(seconds)
    if mute_role in member.roles:
        await member.remove_roles(mute_role)
        
        unmute_embed = discord.Embed(
            title="🔊 Мут снят",
            description=f"Мут с пользователя {member.mention} был автоматически снят.",
            color=discord.Color.green()
        )
        await ctx.send(embed=unmute_embed)

@bot.command(name='unmute', aliases=['размут'])
@commands.has_permissions(manage_roles=True)
async def unmute_user(ctx, member: discord.Member):
    """Снять мут с пользователя"""
    config = bot.server_configs[ctx.guild.id]
    mute_role_id = config.get('mute_role')
    
    if not mute_role_id:
        await ctx.send("❌ Роль мута не настроена!")
        return
    
    mute_role = ctx.guild.get_role(mute_role_id)
    if mute_role not in member.roles:
        await ctx.send("❌ Этот пользователь не замьючен!")
        return
    
    await member.remove_roles(mute_role)
    
    embed = discord.Embed(
        title="🔊 Мут снят",
        description=f"Мут с пользователя {member.mention} был снят.",
        color=discord.Color.green()
    )
    embed.add_field(name="👮 Модератор", value=ctx.author.mention)
    await ctx.send(embed=embed)

# ==================== КОМАНДЫ РАЗВЛЕЧЕНИЙ ====================
@bot.command(name='roll', aliases=['ролл', 'рандом'])
async def roll_dice(ctx, dice: str = "1d6"):
    """Бросить кости (формат: 2d20)"""
    try:
        if "d" not in dice:
            await ctx.send("❌ Используйте формат: `!roll 2d20`")
            return
        
        num, sides = map(int, dice.split('d'))
        num = min(num, 10)  # Ограничение
        sides = min(sides, 100)
        
        rolls = [random.randint(1, sides) for _ in range(num)]
        total = sum(rolls)
        
        embed = discord.Embed(
            title="🎲 Результат броска",
            color=discord.Color.purple()
        )
        embed.add_field(name="Бросок", value=dice, inline=True)
        embed.add_field(name="Результаты", value=", ".join(map(str, rolls)), inline=True)
        embed.add_field(name="Сумма", value=str(total), inline=True)
        
        if num == 1:
            if rolls[0] == sides:
                embed.set_footer(text="🎯 Критический успех!")
            elif rolls[0] == 1:
                embed.set_footer(text="💥 Критический провал!")
        
        await ctx.send(embed=embed)
    except:
        await ctx.send("❌ Используйте формат: `!roll 2d20`")

@bot.command(name='coin', aliases=['монетка', 'орёлрешка'])
async def coin_flip(ctx):
    """Подбросить монетку"""
    result = random.choice(["Орёл", "Решка"])
    emoji = "🦅" if result == "Орёл" else "🪙"
    
    embed = discord.Embed(
        title=f"{emoji} Результат",
        description=f"**{result}**!",
        color=discord.Color.gold()
    )
    
    message = await ctx.send(embed=embed)
    
    # Добавляем реакции для повторного броска
    await message.add_reaction("🔄")
    
    def check(reaction, user):
        return user == ctx.author and str(reaction.emoji) == "🔄" and reaction.message.id == message.id
    
    try:
        await bot.wait_for('reaction_add', timeout=30.0, check=check)
        
        # Повторный бросок
        result = random.choice(["Орёл", "Решка"])
        emoji = "🦅" if result == "Орёл" else "🪙"
        
        embed.description = f"**{result}**! (переброшено)"
        await message.edit(embed=embed)
        await message.clear_reactions()
        
    except asyncio.TimeoutError:
        await message.clear_reactions()

@bot.command(name='avatar', aliases=['аватар', 'ava'])
async def get_avatar(ctx, member: discord.Member = None):
    """Получить аватар пользователя"""
    member = member or ctx.author
    
    embed = discord.Embed(
        title=f"🖼️ Аватар {member.display_name}",
        color=member.color
    )
    embed.set_image(url=member.display_avatar.url)
    embed.set_footer(text=f"ID: {member.id}")
    
    await ctx.send(embed=embed)

@bot.command(name='userinfo', aliases=['юзер', 'инфо'])
async def user_info(ctx, member: discord.Member = None):
    """Информация о пользователе"""
    member = member or ctx.author
    
    # Уровень пользователя
    guild_id = ctx.guild.id
    levels = bot.server_levels.get(guild_id, {})
    user_data = levels.get(str(member.id), {'level': 1, 'xp': 0})
    
    # Варны пользователя
    warns = bot.server_warns.get(guild_id, {})
    warn_count = len(warns.get(str(member.id), []))
    
    embed = discord.Embed(
        title=f"📋 Информация о {member.display_name}",
        color=member.color
    )
    
    embed.set_thumbnail(url=member.display_avatar.url)
    
    # Основная информация
    embed.add_field(
        name="👤 Основное",
        value=f"**Имя:** {member.name}\n"
              f"**Ник:** {member.display_name}\n"
              f"**ID:** {member.id}\n"
              f"**Создан:** {member.created_at.strftime('%d.%m.%Y')}",
        inline=True
    )
    
    # Информация о сервере
    join_date = member.joined_at.strftime('%d.%m.%Y %H:%M') if member.joined_at else "Неизвестно"
    roles = [role.mention for role in member.roles[1:]]  # Без @everyone
    
    embed.add_field(
        name="🏠 Сервер",
        value=f"**Присоединился:** {join_date}\n"
              f"**Варны:** {warn_count}\n"
              f"**Уровень:** {user_data.get('level', 1)}",
        inline=True
    )
    
    # Роли
    if roles:
        roles_text = " ".join(roles[:5])
        if len(roles) > 5:
            roles_text += f" (+{len(roles)-5} еще)"
        embed.add_field(name="🎭 Роли", value=roles_text, inline=False)
    
    # Статус
    status_emojis = {
        'online': '🟢',
        'idle': '🟡',
        'dnd': '🔴',
        'offline': '⚫'
    }
    
    activity = ""
    if member.activity:
        activity_type = str(member.activity.type).split('.')[-1].title()
        activity = f"\n**Активность:** {activity_type} {member.activity.name}"
    
    embed.add_field(
        name="📊 Статус",
        value=f"**Статус:** {status_emojis.get(str(member.status), '⚫')} {str(member.status).title()}{activity}",
        inline=False
    )
    
    await ctx.send(embed=embed)

# ==================== СИСТЕМА УРОВНЕЙ ====================
@bot.command(name='leaderboard', aliases=['топ', 'лидеры'])
async def show_leaderboard(ctx, page: int = 1):
    """Показать таблицу лидеров по уровням"""
    guild_id = ctx.guild.id
    levels = bot.server_levels.get(guild_id, {})
    
    if not levels:
        embed = discord.Embed(
            title="🏆 Таблица лидеров",
            description="На этом сервере еще нет данных об уровнях!",
            color=discord.Color.blue()
        )
        await ctx.send(embed=embed)
        return
    
    # Сортировка пользователей
    leaderboard = []
    for user_id, data in levels.items():
        member = ctx.guild.get_member(int(user_id))
        if member:
            leaderboard.append({
                'member': member,
                'level': data.get('level', 1),
                'xp': data.get('xp', 0),
                'messages': data.get('messages', 0)
            })
    
    leaderboard.sort(key=lambda x: (x['level'], x['xp']), reverse=True)
    
    # Пагинация
    items_per_page = 10
    total_pages = (len(leaderboard) + items_per_page - 1) // items_per_page
    page = max(1, min(page, total_pages))
    
    start_idx = (page - 1) * items_per_page
    end_idx = min(start_idx + items_per_page, len(leaderboard))
    
    # Создание embed
    embed = discord.Embed(
        title=f"🏆 Таблица лидеров {ctx.guild.name}",
        color=discord.Color.gold()
    )
    
    # Медали для топ-3
    medals = ["🥇", "🥈", "🥉"]
    
    for i, data in enumerate(leaderboard[start_idx:end_idx], start=start_idx + 1):
        medal = medals[i-1] if i <= 3 else f"`{i}.`"
     embed.add_field(
            name=f"{medal} {data['member'].display_name}",
            value=f"**Уровень:** {data['level']} | "
                  f"**XP:** {data['xp']:,} | "
                  f"**Сообщения:** {data['messages']:,}",
            inline=False
        )
    
    embed.set_footer(text=f"Страница {page}/{total_pages} | Всего участников: {len(leaderboard)}")
    
    # Добавляем информацию о топ-1
    if leaderboard:
        top_user = leaderboard[0]
        embed.set_thumbnail(url=top_user['member'].display_avatar.url)
    
    message = await ctx.send(embed=embed)
    
    # Добавляем кнопки навигации
    if total_pages > 1:
        await message.add_reaction("⬅️")
        await message.add_reaction("➡️")
        
        def check(reaction, user):
            return (user == ctx.author and 
                   str(reaction.emoji) in ["⬅️", "➡️"] and 
                   reaction.message.id == message.id)
        
        try:
            while True:
                reaction, user = await bot.wait_for('reaction_add', timeout=60.0, check=check)
                
                if str(reaction.emoji) == "⬅️" and page > 1:
                    page -= 1
                elif str(reaction.emoji) == "➡️" and page < total_pages:
                    page += 1
                
                # Обновляем embed
                start_idx = (page - 1) * items_per_page
                end_idx = min(start_idx + items_per_page, len(leaderboard))
                
                embed.clear_fields()
                for i, data in enumerate(leaderboard[start_idx:end_idx], start=start_idx + 1):
                    medal = medals[i-1] if i <= 3 else f"`{i}.`"
                    
                    embed.add_field(
                        name=f"{medal} {data['member'].display_name}",
                        value=f"**Уровень:** {data['level']} | "
                              f"**XP:** {data['xp']:,} | "
                              f"**Сообщения:** {data['messages']:,}",
                        inline=False
                    )
                
                embed.set_footer(text=f"Страница {page}/{total_pages} | Всего участников: {len(leaderboard)}")
                await message.edit(embed=embed)
                await message.remove_reaction(reaction, user)
                
        except asyncio.TimeoutError:
            await message.clear_reactions()

@bot.command(name='rank', aliases=['ранг'])
async def rank_command(ctx, member: discord.Member = None):
    """Посмотреть свой ранг и прогресс"""
    member = member or ctx.author
    guild_id = ctx.guild.id
    
    levels = bot.server_levels.get(guild_id, {})
    user_data = levels.get(str(member.id), {'level': 1, 'xp': 0, 'messages': 0})
    
    level = user_data.get('level', 1)
    xp = user_data.get('xp', 0)
    messages = user_data.get('messages', 0)
    
    # Расчет прогресса
    next_level_xp = 100 * ((level + 1) ** 2)
    current_level_xp = 100 * (level ** 2)
    xp_progress = xp - current_level_xp
    xp_needed = next_level_xp - current_level_xp
    progress_percentage = (xp_progress / xp_needed) * 100 if xp_needed > 0 else 0
    
    # Создание embed
    embed = discord.Embed(
        title=f"📊 Ранг {member.display_name}",
        color=member.color
    )
    
    embed.set_thumbnail(url=member.display_avatar.url)
    
    # Прогресс-бар
    progress_bar = create_progress_bar(xp_progress, xp_needed)
    
    embed.add_field(
        name="🎯 Уровень",
        value=f"**{level}**",
        inline=True
    )
    
    embed.add_field(
        name="⭐ Опыт",
        value=f"**{xp:,}** / **{next_level_xp:,}**",
        inline=True
    )
    
    embed.add_field(
        name="💬 Сообщения",
        value=f"**{messages:,}**",
        inline=True
    )
    
    embed.add_field(
        name="📈 Прогресс",
        value=f"```{progress_bar}```\n"
              f"**{xp_progress:,}** / **{xp_needed:,}** XP "
              f"({progress_percentage:.1f}%)",
        inline=False
    )
    
    # Позиция в топе
    leaderboard = []
    for user_id, data in levels.items():
        guild_member = ctx.guild.get_member(int(user_id))
        if guild_member:
            leaderboard.append((guild_member, data.get('level', 1), data.get('xp', 0)))
    
    leaderboard.sort(key=lambda x: (x[1], x[2]), reverse=True)
    
    for i, (guild_member, lvl, xp_val) in enumerate(leaderboard, 1):
        if guild_member.id == member.id:
            embed.set_footer(text=f"Позиция в топе: #{i} из {len(leaderboard)}")
            
            # Добавляем информацию о следующей позиции
            if i > 1:
                prev_member, prev_level, prev_xp = leaderboard[i-2]
                xp_to_next = max(0, prev_xp - xp + 1)
                embed.add_field(
                    name="🏆 До следующей позиции",
                    value=f"Нужно еще **{xp_to_next:,}** XP\n"
                          f"чтобы обогнать {prev_member.display_name}",
                    inline=False
                )
            break
    
    await ctx.send(embed=embed)

# ==================== КОМАНДЫ МОДЕРАЦИИ ====================
@bot.command(name='warns', aliases=['варны'])
@commands.has_permissions(manage_messages=True)
async def check_warns(ctx, member: discord.Member = None):
    """Посмотреть предупреждения пользователя"""
    member = member or ctx.author
    guild_id = ctx.guild.id
    
    warns = bot.server_warns.get(guild_id, {})
    user_warns = warns.get(str(member.id), [])
    
    if not user_warns:
        embed = discord.Embed(
            title=f"✅ Предупреждения {member.display_name}",
            description="У этого пользователя нет предупреждений!",
            color=discord.Color.green()
        )
        await ctx.send(embed=embed)
        return
    
    embed = discord.Embed(
        title=f"⚠️ Предупреждения {member.display_name}",
        description=f"Всего предупреждений: **{len(user_warns)}**",
        color=discord.Color.orange()
    )
    
    for i, warn in enumerate(user_warns[:10], 1):
        moderator_id = warn.get('moderator')
        moderator = ctx.guild.get_member(moderator_id) if moderator_id else "Неизвестно"
        reason = warn.get('reason', 'Не указана')
        timestamp = warn.get('timestamp', 'Неизвестно')
        
        try:
            time_obj = datetime.fromisoformat(timestamp)
            time_str = time_obj.strftime('%d.%m.%Y %H:%M')
        except:
            time_str = timestamp
        
        embed.add_field(
            name=f"#{i} - {time_str}",
            value=f"**Модератор:** {moderator.mention if isinstance(moderator, discord.Member) else moderator}\n"
                  f"**Причина:** {reason}",
            inline=False
        )
    
    if len(user_warns) > 10:
        embed.set_footer(text=f"Показано 10 из {len(user_warns)} предупреждений")
    
    await ctx.send(embed=embed)

@bot.command(name='clearwarns', aliases=['очиститьварны'])
@commands.has_permissions(manage_messages=True)
async def clear_warns(ctx, member: discord.Member):
    """Очистить все предупреждения пользователя"""
    guild_id = ctx.guild.id
    
    if str(member.id) not in bot.server_warns[guild_id]:
        await ctx.send("❌ У этого пользователя нет предупреждений!")
        return
    
    warn_count = len(bot.server_warns[guild_id][str(member.id)])
    bot.server_warns[guild_id][str(member.id)] = []
    bot.data_manager.save_server_warns(guild_id, dict(bot.server_warns[guild_id]))
    
    embed = discord.Embed(
        title="✅ Предупреждения очищены",
        description=f"Удалено {warn_count} предупреждений у пользователя {member.mention}",
        color=discord.Color.green()
    )
    await ctx.send(embed=embed)

# ==================== ИНФОРМАЦИОННЫЕ КОМАНДЫ ====================
@bot.command(name='serverinfo', aliases=['сервер', 'инфосервер'])
async def server_info(ctx):
    """Информация о сервере"""
    guild = ctx.guild
    
    # Статистика
    total_members = guild.member_count
    online_members = len([m for m in guild.members if m.status != discord.Status.offline])
    bots = len([m for m in guild.members if m.bot])
    
    # Каналы
    text_channels = len(guild.text_channels)
    voice_channels = len(guild.voice_channels)
    
    # Уровни и варны
    levels = bot.server_levels.get(guild.id, {})
    warns = bot.server_warns.get(guild.id, {})
    
    # Создание embed
    embed = discord.Embed(
        title=f"🏰 Информация о {guild.name}",
        color=guild.owner.color if guild.owner else BOT_COLOR
    )
    
    if guild.icon:
        embed.set_thumbnail(url=guild.icon.url)
    
    # Основная информация
    embed.add_field(
        name="👑 Владелец",
        value=guild.owner.mention if guild.owner else "Неизвестно",
        inline=True
    )
    
    embed.add_field(
        name="📅 Создан",
        value=guild.created_at.strftime('%d.%m.%Y'),
        inline=True
    )
    
    embed.add_field(
        name="🆔 ID",
        value=guild.id,
        inline=True
    )
    
    # Статистика участников
    embed.add_field(
        name="👥 Участники",
        value=f"**Всего:** {total_members:,}\n"
              f"**Онлайн:** {online_members:,}\n"
              f"**Боты:** {bots:,}",
        inline=True
    )
    
    # Каналы
    embed.add_field(
        name="📊 Каналы",
        value=f"**Текстовые:** {text_channels:,}\n"
              f"**Голосовые:** {voice_channels:,}\n"
              f"**Всего:** {text_channels + voice_channels:,}",
        inline=True
    )
    
    # Система бота
    embed.add_field(
        name="🤖 Статистика бота",
        value=f"**Участников с уровнями:** {len(levels):,}\n"
              f"**Всего предупреждений:** {sum(len(w) for w in warns.values()):,}\n"
              f"**Использовано команд:** {bot.server_configs.get(guild.id, {}).get('commands_used', 0):,}",
        inline=True
    )
    
    # Уровни сервера
    if guild.verification_level != discord.VerificationLevel.none:
        verification_levels = {
            discord.VerificationLevel.none: "Нет",
            discord.VerificationLevel.low: "Низкий",
            discord.VerificationLevel.medium: "Средний",
            discord.VerificationLevel.high: "Высокий",
            discord.VerificationLevel.highest: "Максимальный"
        }
        
        embed.add_field(
            name="🛡️ Уровень проверки",
            value=verification_levels.get(guild.verification_level, "Неизвестно"),
            inline=True
        )
    
    # Бусты
    if guild.premium_subscription_count:
        embed.add_field(
            name="🚀 Бусты",
            value=f"**Количество:** {guild.premium_subscription_count:,}\n"
                  f"**Уровень:** {guild.premium_tier}",
            inline=True
        )
    
    embed.set_footer(text=f"Сервер #{ctx.guild.shard_id if ctx.guild.shard_id else 1}")
    await ctx.send(embed=embed)

@bot.command(name='botinfo', aliases=['би', 'инфобот'])
async def bot_info(ctx):
    """Информация о боте"""
    # Статистика
    total_members = sum(g.member_count for g in bot.guilds)
    total_channels = sum(len(g.channels) for g in bot.guilds)
    
    # Время работы
    uptime = datetime.now() - bot.start_time
    uptime_str = format_time(int(uptime.total_seconds()))
    
    # Производительность
    latency = round(bot.latency * 1000)
    
    # Создание embed
    embed = discord.Embed(
        title="🤖 Информация о боте",
        color=BOT_COLOR
    )
    
    embed.set_thumbnail(url=bot.user.display_avatar.url)
    
    # Основная информация
    embed.add_field(
        name="📊 Статистика",
        value=f"**Серверов:** {len(bot.guilds):,}\n"
              f"**Участников:** {total_members:,}\n"
              f"**Каналов:** {total_channels:,}",
        inline=True
    )
    
    embed.add_field(
        name="⚡ Производительность",
        value=f"**Задержка:** {latency}мс\n"
              f"**Время работы:** {uptime_str}\n"
              f"**Версия Python:** {sys.version.split()[0]}",
        inline=True
    )
    
    # Техническая информация
    embed.add_field(
        name="🔧 Техническое",
        value=f"**Библиотека:** discord.py {discord.__version__}\n"
              f"**Префикс:** {PREFIX} (настраиваемый)\n"
              f"**Создатель:** <@{CREATOR_ID}>",
        inline=False
    )
    
    # Команды
    total_commands = len(bot.commands)
    categories = {}
    for cmd in bot.commands:
        category = cmd.module.split('.')[-1] if hasattr(cmd, 'module') else 'Other'
        categories[category] = categories.get(category, 0) + 1
    
    commands_info = "\n".join([f"**{cat}:** {count}" for cat, count in list(categories.items())[:3]])
    embed.add_field(
        name="📁 Команды",
        value=f"**Всего команд:** {total_commands}\n{commands_info}",
        inline=True
    )
    
    # Ссылки
    embed.add_field(
        name="🔗 Ссылки",
        value=f"[Добавить бота]({INVITE_URL})\n"
              f"[Поддержка](https://discord.gg/ваш-сервер)\n"
              f"[Исходный код](https://github.com/ваш-репозиторий)",
        inline=True
    )
    
    embed.set_footer(text=f"ID: {bot.user.id} | Запущен: {bot.start_time.strftime('%d.%m.%Y %H:%M')}")
    await ctx.send(embed=embed)

@bot.command(name='help', aliases=['помощь', 'хелп'])
async def custom_help(ctx, command_name: str = None):
    """Показать список команд или помощь по конкретной команде"""
    if command_name:
        # Помощь по конкретной команде
        cmd = bot.get_command(command_name)
        if not cmd:
            embed = discord.Embed(
                title="❌ Команда не найдена",
                description=f"Команда `{command_name}` не существует!",
                color=discord.Color.red()
            )
            await ctx.send(embed=embed)
            return
        
        embed = discord.Embed(
            title=f"📖 Помощь по команде: {cmd.name}",
            color=BOT_COLOR
        )
        
        embed.add_field(
            name="Описание",
            value=cmd.help or "Описание отсутствует",
            inline=False
        )
        
        if cmd.aliases:
            embed.add_field(
                name="Алиасы",
                value=", ".join(f"`{alias}`" for alias in cmd.aliases),
                inline=True
            )
        
        usage = f"{ctx.prefix}{cmd.name} {cmd.signature}"
        embed.add_field(
            name="Использование",
            value=f"`{usage}`",
            inline=False
        )
        
        if hasattr(cmd, 'checks'):
            permissions = []
            for check in cmd.checks:
                if hasattr(check, '__name__') and check.__name__ == 'has_permissions':
                    permissions.append("Требуются права")
                elif hasattr(check, '__name__') and check.__name__ == 'admin_only':
                    permissions.append("Только для администраторов")
            
            if permissions:
                embed.add_field(
                    name="Требуемые права",
                    value=", ".join(permissions),
                    inline=True
                )
        
        await ctx.send(embed=embed)
    else:
        # Общая помощь
        embed = discord.Embed(
            title="📚 Список команд",
            description=f"Префикс бота: `{ctx.prefix}`\n"
                       f"Всего команд: **{len(bot.commands)}**\n"
                       f"Используйте `{ctx.prefix}help [команда]` для подробностей",
            color=BOT_COLOR
        )
        
        # Группируем команды по категориям
        categories = {
            "⚙️ Администрация": ['prefix', 'clear', 'mute', 'unmute'],
            "📊 Модерация": ['warns', 'clearwarns'],
            "🎮 Развлечения": ['roll', 'coin', 'avatar', 'userinfo'],
            "🏆 Уровни": ['rank', 'leaderboard'],
            "ℹ️ Информация": ['botinfo', 'serverinfo', 'help', 'ping'],
            "🔧 Утилиты": ['invite']
        }
        
        for category, commands_list in categories.items():
            commands_text = []
            for cmd_name in commands_list:
                cmd = bot.get_command(cmd_name)
                if cmd:
                    commands_text.append(f"`{cmd.name}`")
            
            if commands_text:
                embed.add_field(
                    name=category,
                    value=" ".join(commands_text),
                    inline=False
                )
        
        embed.set_footer(text=f"Бот находится на {len(bot.guilds)} серверах")
        await ctx.send(embed=embed)

# ==================== УТИЛИТЫ ====================
@bot.command(name='ping', aliases=['пинг'])
async def ping_command(ctx):
    """Проверить задержку бота"""
    latency = round(bot.latency * 1000)
    
    # Цвет в зависимости от задержки
    if latency < 100:
        color = discord.Color.green()
        status = "✅ Отличная"
    elif latency < 200:
        color = discord.Color.yellow()
        status = "⚠️ Нормальная"
    else:
        color = discord.Color.red()
        status = "❌ Высокая"
    
    embed = discord.Embed(
        title="🏓 Понг!",
        description=f"**Задержка:** {latency}мс\n"
                   f"**Статус:** {status}",
        color=color
    )
    
    # Добавляем смешное сообщение
    if latency < 50:
        embed.set_footer(text="⚡ Молниеносно!")
    elif latency < 100:
        embed.set_footer(text="🚀 Быстро как ракета!")
    elif latency < 200:
        embed.set_footer(text="🐢 Медленно, но верно...")
    else:
        embed.set_footer(text="🚶 Иду пешком...")
    
    await ctx.send(embed=embed)

@bot.command(name='invite', aliases=['пригласить', 'инвайт'])
async def invite_bot(ctx):
    """Получить ссылку для приглашения бота"""
    embed = discord.Embed(
        title="🤖 Пригласить бота",
        description="Нажмите на ссылку ниже, чтобы добавить бота на ваш сервер!",
        color=BOT_COLOR
    )
    
    embed.add_field(
        name="🔗 Ссылка для приглашения",
        value=f"[Нажмите здесь]({INVITE_URL})",
        inline=False
    )
    
    embed.add_field(
        name="📋 Требуемые права",
        value="```"
              "Администратор (рекомендуется)\n"
              "Или выберите:\n"
              "- Управление сообщениями\n"
              "- Управление ролями\n"
              "- Чтение/отправка сообщений\n"
              "```",
        inline=False
    )
    
    await ctx.send(embed=embed)

# ==================== СИСТЕМНЫЕ КОМАНДЫ ====================
@bot.command(name='reload', aliases=['перезагрузить'])
@commands.is_owner()
async def reload_cog(ctx, cog: str = None):
    """Перезагрузить ког (только для создателя)"""
    if not cog:
        embed = discord.Embed(
            title="❌ Укажите ког для перезагрузки",
            description="Использование: !reload [название_кога]",
            color=discord.Color.red()
        )
        await ctx.send(embed=embed)
        return
    
    try:
        await bot.reload_extension(f"cogs.{cog}")
        embed = discord.Embed(
            title="✅ Ког перезагружен",
            description=f"Ког `{cog}` успешно перезагружен!",
            color=discord.Color.green()
        )
    except Exception as e:
        embed = discord.Embed(
            title="❌ Ошибка перезагрузки",
            description=f"Ошибка при перезагрузке кога `{cog}`:\n```{e}```",
            color=discord.Color.red()
        )
    
    await ctx.send(embed=embed)

@bot.command(name='shutdown', aliases=['выключить'])
@commands.is_owner()
async def shutdown_bot(ctx):
    """Выключить бота (только для создателя)"""
    embed = discord.Embed(
        title="⏏️ Выключение бота",
        description="Бот выключается...",
        color=discord.Color.red()
    )
    
    await ctx.send(embed=embed)
    
    # Сохраняем данные
    await save_all_data()
    
    # Закрываем соединение
    await bot.close()

# ==================== ФОНОВЫЕ ЗАДАЧИ ====================
@tasks.loop(minutes=5)
async def save_all_data():
    """Автосохранение всех данных"""
    try:
        for guild_id, config in bot.server_configs.items():
            bot.data_manager.save_server_config(guild_id, config)
        
        for guild_id, warns in bot.server_warns.items():
            bot.data_manager.save_server_warns(guild_id, dict(warns))
        
        for guild_id, levels in bot.server_levels.items():
            bot.data_manager.save_server_levels(guild_id, levels)
        
        bot.data_manager.save_global_stats(bot.global_stats)
        
        logger.info("Все данные сохранены")
    except Exception as e:
        logger.error(f"Ошибка автосохранения: {e}")

@tasks.loop(minutes=2)
async def status_task():
    """Обновление статуса бота"""
    total_members = sum(g.member_count for g in bot.guilds)
    
    statuses = [
        discord.Activity(type=discord.ActivityType.watching, name=f"{len(bot.guilds)} серверов"),
        discord.Activity(type=discord.ActivityType.listening, name=f"{total_members} участников"),
        discord.Activity(type=discord.ActivityType.playing, name="мультисерверный режим"),
        discord.Activity(type=discord.ActivityType.competing, name="веб-панель"),
        discord.Activity(type=discord.ActivityType.streaming, name="команды", url="https://twitch.tv/discord")
    ]
    
    await bot.change_presence(activity=random.choice(statuses))

# ==================== ОБРАБОТЧИКИ ОШИБОК ====================
@bot.event
async def on_command_error(ctx, error):
    """Обработка ошибок команд"""
    if isinstance(error, commands.CommandNotFound):
        embed = discord.Embed(
            title="❌ Команда не найдена",
            description=f"Команда `{ctx.invoked_with}` не существует!\n"
                       f"Используйте `{ctx.prefix}help` для списка команд.",
            color=discord.Color.red()
        )
        await ctx.send(embed=embed)
    
    elif isinstance(error, commands.MissingPermissions):
        embed = discord.Embed(
            title="❌ Недостаточно прав",
            description="У вас нет прав для выполнения этой команды!",
            color=discord.Color.red()
        )
        await ctx.send(embed=embed)
    
    elif isinstance(error, commands.MissingRequiredArgument):
        embed = discord.Embed(
            title="❌ Отсутствует аргумент",
            description=f"Использование: `{ctx.prefix}{ctx.command.name} {ctx.command.signature}`\n"
                       f"Подробнее: `{ctx.prefix}help {ctx.command.name}`",
            color=discord.Color.red()
        )
        await ctx.send(embed=embed)
    
    elif isinstance(error, commands.BadArgument):
        embed = discord.Embed(
            title="❌ Неверный аргумент",
            description="Проверьте правильность введенных данных!",
            color=discord.Color.red()
        )
        await ctx.send(embed=embed)
    
    elif isinstance(error, commands.CommandOnCooldown):
        embed = discord.Embed(
            title="⏰ Охлаждение команды",
            description=f"Пожалуйста, подождите {error.retry_after:.1f} секунд!",
            color=discord.Color.orange()
        )
        await ctx.send(embed=embed)
    
    else:
        # Логируем неожиданные ошибки
        logger.error(f"Ошибка в команде {ctx.command}: {error}", exc_info=True)
        
        embed = discord.Embed(
            title="💥 Произошла ошибка",
            description="Произошла непредвиденная ошибка. Разработчик уже уведомлен!",
            color=discord.Color.red()
        )
        await ctx.send(embed=embed)

# ==================== СОБЫТИЯ БОТА ====================
@bot.event
async def on_ready():
    """Запуск бота"""
    print('╔' + '═' * 48 + '╗')
    print('║' + ' ' * 48 + '║')
    print('║{:^48}║'.format('🤖 Бот успешно запущен!'))
    print('║' + ' ' * 48 + '║')
    print('╠' + '═' * 48 + '╣')
    print(f'║ 📝 Имя: {bot.user.name:>36} ║')
    print(f'║ 🆔 ID: {bot.user.id:>39} ║')
    print(f'║ 🏠 Серверов: {len(bot.guilds):>33} ║')
    print(f'║ 👥 Пользователей: {sum(g.member_count for g in bot.guilds):>29} ║')
    print(f'║ ⚡ Задержка: {round(bot.latency * 1000):>35}мс ║')
    print('╠' + '═' * 48 + '╣')
    print(f'║ 🔗 Пригласить: {INVITE_URL:>34} ║')
    print('╚' + '═' * 48 + '╝')
    
    # Загрузка данных для каждого сервера
    for guild in bot.guilds:
        guild_id = guild.id
        
        # Загружаем конфигурацию
        config = bot.data_manager.load_server_config(guild_id)
        bot.server_configs[guild_id] = config
        
        # Загружаем варны
        warns = bot.data_manager.load_server_warns(guild_id)
        bot.server_warns[guild_id] = defaultdict(list, warns)
        
        # Загружаем уровни
        levels = bot.data_manager.load_server_levels(guild_id)
        bot.server_levels[guild_id] = levels
        
        logger.info(f"Загружены данные для сервера: {guild.name}")
    
    # Запуск фоновых задач
    status_task.start()
    save_all_data.start()

@bot.event
async def on_message(message):
    """Обработка сообщений"""
    if message.author.bot:
        return
    
    # Обновляем глобальную статистику
    bot.global_stats['total_messages'] = bot.global_stats.get('total_messages', 0) + 1
    
    # Обновляем статистику сервера
    if message.guild:
        guild_id = message.guild.id
        bot.global_stats.setdefault('messages_by_guild', {})
        bot.global_stats['messages_by_guild'][str(guild_id)] = bot.global_stats['messages_by_guild'].get(str(guild_id), 0) + 1
        
        # Система уровней с кулдауном
        config = bot.server_configs.get(guild_id, {})
        user_id = str(message.author.id)
        
        # Проверяем кулдаун
        last_xp = config.get('last_xp_gain', {}).get(user_id, 0)
        current_time = time.time()
        xp_cooldown = config.get('xp_cooldown', 60)
        
        if current_time - last_xp > xp_cooldown:
            # Обновляем время последнего получения XP
            config.setdefault('last_xp_gain', {})[user_id] = current_time
            bot.data_manager.save_server_config(guild_id, config)
            
            # Добавляем опыт
            levels = bot.server_levels[guild_id]
            if user_id not in levels:
                levels[user_id] = {'xp': 0, 'level': 1, 'messages': 0}
            
            user_data = levels[user_id]
            xp_gain = random.randint(10, 25)
            user_data['xp'] += xp_gain
            user_data['messages'] = user_data.get('messages', 0) + 1
            
            # Проверяем повышение уровня
            old_level = user_data['level']
            new_level = int(math.sqrt(user_data['xp'] / 100))
            
            if new_level > old_level:
                user_data['level'] = new_level
                
                # Отправляем уведомление
                channel_id = config.get('level_up_channel')
                channel = message.guild.get_channel(channel_id) if channel_id else message.channel
                
                if channel and channel.permissions_for(message.guild.me).send_messages:
                    embed = discord.Embed(
                        title="🎉 Новый уровень!",
                        description=f"{message.author.mention} достиг **{new_level} уровня**! 🚀",
                        color=discord.Color.gold()
                    )
                    embed.set_thumbnail(url=message.author.display_avatar.url)
                    
                    # Награды за уровни
                    rewards = {
                        5: "🏆 **Награда:** Специальная роль!",
                        10: "🎖️ **Награда:** Доступ к скрытным каналам!",
                        20: "💎 **Награда:** VIP статус!",
                        50: "👑 **Награда:** Легенда сервера!"
                    }
                    
                    if new_level in rewards:
                        embed.add_field(name="🎁 Поздравляем!", value=rewards[new_level], inline=False)
                    
                    await channel.send(embed=embed)
            
            bot.data_manager.save_server_levels(guild_id, levels)
    
    # Обрабатываем команды
    await bot.process_commands(message)

# ==================== ОСНОВНОЙ ЗАПУСК ====================
def main():
    """Основная функция запуска бота"""
    if TOKEN == 'ВАШ_ТОКЕН_БОТА_ЗДЕСЬ':
        print("\n" + "═" * 50)
        print("❌ ОШИБКА: Токен бота не установлен!")
        print("⚠️  Пожалуйста, установите ваш токен в переменной TOKEN")
        print("═" * 50)
        return
    
    try:
        print("\n" + "═" * 50)
        print("🚀 Запуск бота...")
        print("═" * 50)
        
        # Запускаем бота
        bot.run(TOKEN)
        
    except discord.LoginFailure:
        print("\n" + "═" * 50)
        print("❌ ОШИБКА: Неверный токен бота!")
        print("⚠️  Пожалуйста, проверьте правильность токена")
        print("═" * 50)
    
    except KeyboardInterrupt:
        print("\n" + "═" * 50)
        print("👋 Завершение работы...")
        
        # Сохраняем данные перед выходом
        loop = asyncio.get_event_loop()
        loop.run_until_complete(save_all_data())
        
        print("💾 Все данные сохранены")
        print("✅ Бот успешно выключен")
        print("═" * 50)
    
    except Exception as e:
        print(f"\n❌ Критическая ошибка: {e}")
        traceback.print_exc()

if __name__ == "__main__":
    main()

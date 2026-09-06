import asyncio
import discord
from discord import app_commands
from discord.ext import commands

# -----------------------------------------------------
# ІНІЦІАЛІЗАЦІЯ БОТА ТА НАЛАШТУВАННЯ
# -----------------------------------------------------

TOKEN = "MTU0NTg5MDkxNTI3NTUxMzg4Nw.Gndqx3.hHumFAXshyaAQLNuUilSN4PCtweVerJgLKYOog"
MODERATION_CHANNEL_ID = 1545900651492544602

intents = discord.Intents.default()
intents.members = True

bot = commands.Bot(command_prefix="!", intents=intents)

applications = {}

class ApplicationView(discord.ui.View):
    def __init__(self, application_id: int):
        super().__init__(timeout=None)
        self.application_id = application_id

    @discord.ui.button(label="Схвалити", style=discord.ButtonStyle.green, custom_id="approve_role")
    async def approve_button(self, interaction: discord.Interaction, button: discord.ui.Button):
        app_data = applications.get(self.application_id)
        if not app_data:
            await interaction.response.send_message("❌ Ця заявка вже застаріла або була видалена.", ephemeral=True)
            return

        guild = interaction.guild
        if not guild:
            return

        member = guild.get_member(app_data["user_id"])
        role = guild.get_role(app_data["role_id"])

        if not member or not role:
            await interaction.response.send_message("❌ Не вдалося знайти користувача або роль.", ephemeral=True)
            return

        try:
            if app_data["action"] == "add":
                await member.add_roles(role)
                action_desc = f"надано роль **{role.name}**"
            else:
                await member.remove_roles(role)
                action_desc = f"знято роль **{role.name}**"

            await interaction.response.send_message(f"✅ Заявку схвалено! Користувачу {member.mention} {action_desc}.", ephemeral=True)
            
            # Вимикаємо кнопки після натискання
            for child in self.children:
                child.disabled = True
            await interaction.message.edit(view=self)

            try:
                await member.send(f"✅ Твою заявку на **{role.name}** схвалено модератором {interaction.user.mention}!")
            except discord.Forbidden:
                pass

            applications.pop(self.application_id, None)

        except discord.Forbidden:
            await interaction.response.send_message("❌ У бота недостатньо прав для керування цією роллю.", ephemeral=True)

    @discord.ui.button(label="Відхилити", style=discord.ButtonStyle.red, custom_id="reject_role")
    async def reject_button(self, interaction: discord.Interaction, button: discord.ui.Button):
        app_data = applications.get(self.application_id)
        if not app_data:
            await interaction.response.send_message("❌ Ця заявка вже застаріла або була видалена.", ephemeral=True)
            return

        guild = interaction.guild
        if not guild:
            return

        member = guild.get_member(app_data["user_id"])
        role = guild.get_role(app_data["role_id"])

        await interaction.response.send_message("❌ Заявку відхилено.", ephemeral=True)

        for child in self.children:
            child.disabled = True
        await interaction.message.edit(view=self)

        if member and role:
            try:
                await member.send(f"❌ Твою заявку на **{role.name}** було відхилено модератором.")
            except discord.Forbidden:
                pass

        applications.pop(self.application_id, None)


# Фон-задача для відправки повідомлень без затримки команди
async def send_background_tasks(guild, moderation_channel_id, interaction, role, action_text, application_id):
    try:
        moderation_channel = guild.get_channel(moderation_channel_id)
        if moderation_channel:
            embed = discord.Embed(title="📥 Нова заявка", color=discord.Color.blurple())
            embed.add_field(name="👤 Користувач", value=f"{interaction.user.mention}\n`{interaction.user.id}`", inline=False)
            embed.add_field(name="📋 Дія", value=action_text, inline=True)
            embed.add_field(name="🎭 Роль", value=role.mention, inline=True)
            embed.add_field(name="🆔 ID заявки", value=f"`{application_id}`", inline=False)
            embed.set_footer(text="Використовуйте кнопки нижче для обробки заявки.")

            await moderation_channel.send(embed=embed, view=ApplicationView(application_id))

        try:
            await interaction.user.send(f"📥 Твоя заявка на **{action_text.lower()}** **{role.name}** була надіслана модерації.\n\nОчікуй рішення.")
        except discord.Forbidden:
            pass
    except Exception as e:
        print(f"Помилка у фоновій задачі: {e}")


# =========================================================
# /get_a_role
# =========================================================

@bot.tree.command(
    name="get_a_role",
    description="Подати заявку на отримання ролі"
)
@app_commands.describe(
    role="Роль, яку ти хочеш отримати"
)
async def get_a_role(
    interaction: discord.Interaction,
    role: discord.Role
):
    guild = interaction.guild
    if guild is None:
        await interaction.response.send_message("❌ Цю команду можна використовувати лише на сервері.", ephemeral=True)
        return

    if role >= guild.me.top_role:
        await interaction.response.send_message("❌ Бот не може керувати цією роллю. Підніми роль бота вище неї.", ephemeral=True)
        return

    if role in interaction.user.roles:
        await interaction.response.send_message("❌ У тебе вже є ця роль.", ephemeral=True)
        return

    moderation_channel = guild.get_channel(MODERATION_CHANNEL_ID)
    if moderation_channel is None:
        await interaction.response.send_message("❌ Канал модерації не знайдено.", ephemeral=True)
        return

    application_id = interaction.id
    applications[application_id] = {
        "user_id": interaction.user.id,
        "role_id": role.id,
        "action": "add"
    }

    action_text = "Отримання ролі"

    await interaction.response.send_message(f"✅ Заявку на **{action_text.lower()}** {role.mention} надіслано модерації.", ephemeral=True)
    asyncio.create_task(send_background_tasks(guild, MODERATION_CHANNEL_ID, interaction, role, action_text, application_id))


# =========================================================
# /remove_role
# =========================================================

@bot.tree.command(
    name="remove_role",
    description="Подати заявку на зняття ролі"
)
@app_commands.describe(
    role="Роль, яку ти хочеш зняти"
)
async def remove_role(
    interaction: discord.Interaction,
    role: discord.Role
):
    guild = interaction.guild
    if guild is None:
        await interaction.response.send_message("❌ Цю команду можна використовувати лише на сервері.", ephemeral=True)
        return

    if role >= guild.me.top_role:
        await interaction.response.send_message("❌ Бот не може керувати цією роллю. Підніми роль бота вище неї.", ephemeral=True)
        return

    if role not in interaction.user.roles:
        await interaction.response.send_message("❌ У тебе немає цієї ролі.", ephemeral=True)
        return

    moderation_channel = guild.get_channel(MODERATION_CHANNEL_ID)
    if moderation_channel is None:
        await interaction.response.send_message("❌ Канал модерації не знайдено.", ephemeral=True)
        return

    application_id = interaction.id
    applications[application_id] = {
        "user_id": interaction.user.id,
        "role_id": role.id,
        "action": "remove"
    }

    action_text = "Зняття ролі"

    await interaction.response.send_message(f"✅ Заявку на **{action_text.lower()}** {role.mention} надіслано модерації.", ephemeral=True)
    asyncio.create_task(send_background_tasks(guild, MODERATION_CHANNEL_ID, interaction, role, action_text, application_id))


# =========================================================
# ЗАПУСК
# =========================================================

@bot.event
async def on_ready():
    try:
        synced = await bot.tree.sync()
        print("--------------------------------")
        print(f"✅ Бот запущено: {bot.user}")
        print(f"🆔 ID бота: {bot.user.id}")
        print(f"📋 Команд синхронізовано: {len(synced)}")
        print("--------------------------------")
    except Exception as error:
        print(f"❌ Помилка синхронізації команд: {error}")


if __name__ == "__main__":
    bot.run(TOKEN)
    

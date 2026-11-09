import discord
from discord.ext import commands, tasks
import os
import time
from collections import defaultdict, deque

TOKEN = os.getenv("DISCORD_TOKEN")
WELCOME_CHANNEL_ID = int(os.getenv("WELCOME_CHANNEL_ID", "0"))
VERIFY_ROLE_ID = int(os.getenv("VERIFY_ROLE_ID", "0"))

intents = discord.Intents.default()
intents.members = True
intents.message_content = True

bot = commands.Bot(command_prefix="!", intents=intents)

# מערכת אימות
class VerifyButton(discord.ui.View):
    def __init__(self):
        super().__init__(timeout=None)

    @discord.ui.button(
        label="Verify",
        emoji="✅",
        style=discord.ButtonStyle.success,
        custom_id="neh_oray_verify"
    )
    async def verify(self, interaction: discord.Interaction, button: discord.ui.Button):
        role = interaction.guild.get_role(VERIFY_ROLE_ID)

        if role is None:
            return await interaction.response.send_message(
                "לא הוגדר רול אימות!", ephemeral=True
            )

        if role in interaction.user.roles:
            return await interaction.response.send_message(
                "אתה כבר מאומת!", ephemeral=True
            )

        await interaction.user.add_roles(role)
        await interaction.response.send_message(
            "✅ עברת אימות בהצלחה!", ephemeral=True
        )


@bot.event
async def setup_hook():
    bot.add_view(VerifyButton())


# כשהבוט מתחבר
@bot.event
async def on_ready():
    print(f"{bot.user} is online!")
    await bot.change_presence(
        activity=discord.Game(name="Nehoray | Bot")
    )


# ברוכים הבאים
@bot.event
async def on_member_join(member):
    channel = bot.get_channel(WELCOME_CHANNEL_ID)

    if channel:
        embed = discord.Embed(
            title="👋 Welcome!",
            description=f"ברוך הבא {member.mention} לשרת **{member.guild.name}**!",
            color=discord.Color.blue()
        )
        embed.set_thumbnail(url=member.display_avatar.url)
        await channel.send(embed=embed)


# אנטי ספאם בסיסי
messages = defaultdict(deque)

@bot.event
async def on_message(message):
    if message.author.bot:
        return

    now = time.time()
    user_messages = messages[message.author.id]

    while user_messages and now - user_messages[0] > 5:
        user_messages.popleft()

    user_messages.append(now)

    if len(user_messages) >= 6:
        if message.guild and message.author.guild_permissions.manage_messages:
            await bot.process_commands(message)
            return

        try:
            await message.delete()
            await message.author.timeout(
                discord.utils.utcnow() + discord.timedelta(seconds=30),
                reason="Anti-Spam"
            )
        except discord.Forbidden:
            pass

        return

    await bot.process_commands(message)


# פקודת בדיקת בוט
@bot.command()
async def ping(ctx):
    await ctx.send(f"🏓 Pong! {round(bot.latency * 1000)}ms")


# מחיקת הודעות
@bot.command()
@commands.has_permissions(manage_messages=True)
async def clear(ctx, amount: int = 5):
    if amount < 1 or amount > 100:
        return await ctx.send("בחר מספר בין 1 ל-100.")

    deleted = await ctx.channel.purge(limit=amount + 1)
    await ctx.send(f"🧹 נמחקו {len(deleted)-1} הודעות!")


# מערכת אימות
@bot.command()
@commands.has_permissions(administrator=True)
async def verifysetup(ctx):
    embed = discord.Embed(
        title="🔐 Server Verification",
        description="לחץ על הכפתור למטה כדי לקבל גישה לשרת!",
        color=discord.Color.blue()
    )
    await ctx.send(embed=embed, view=VerifyButton())


# הודעה אוטומטית לפי פקודה
@bot.command()
@commands.has_permissions(administrator=True)
async def announce(ctx, *, text):
    embed = discord.Embed(
        title="📢 Announcement",
        description=text,
        color=discord.Color.blue()
    )
    await ctx.send(embed=embed)


# טיפול בשגיאות הרשאה
@bot.event
async def on_command_error(ctx, error):
    if isinstance(error, commands.MissingPermissions):
        await ctx.send("❌ אין לך הרשאה להשתמש בפקודה הזאת.")
    elif isinstance(error, commands.CommandNotFound):
        pass
    else:
        print(f"Error: {error}")


if not TOKEN:
    raise RuntimeError("Missing DISCORD_TOKEN environment variable")

bot.run(TOKEN)

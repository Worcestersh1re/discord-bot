# discord-bot
# filename: role_select_member_bot.py
import os
import discord
from discord.ext import commands
from discord.ui import Select, View
from flask import Flask
from threading import Thread


# create bot and set prefix
intents = discord.Intents.default()
intents.message_content = True  # reading contents
intents.members = True  # loading members
bot = commands.Bot(command_prefix='!', intents=intents)

# when bot is ready, replying a notification
@bot.event
async def on_ready():
    print(f'Logged in as {bot.user}!')

# choose members
@bot.command(name='叫人')
async def select_role(ctx):
    # catch the role list, filter out `@everyone`
    roles = [role for role in ctx.guild.roles if role.name != "@everyone"]

    # create options
    options = [discord.SelectOption(label=role.name, value=str(role.id)) for role in roles]

    # create select menu to roles
    role_select = Select(placeholder="選擇身份", options=options)

    # callback of selectmenu
    async def role_select_callback(interaction):
        # catches the selected role id
        selected_role_id = int(role_select.values[0])
        selected_role = ctx.guild.get_role(selected_role_id)

        # filter the member list, only keep the member who has the selected role
        members = [member for member in ctx.guild.members if selected_role in member.roles and not member.bot]

        # check if there are members
        if not members:
            await interaction.response.send_message(f'没有人有 "{selected_role.name}" 這個身份。', ephemeral=True)
            return

        # limit the number of members to 25
        members = members[:25]

        # create a list of member names
        member_options = [discord.SelectOption(label=member.name, value=str(member.id)) for member in members]

        # create a new select menu to members
        member_select = Select(placeholder="看看有哪個小王八蛋", options=member_options, min_values=1, max_values=len(member_options))

        # callback of member select
        async def member_select_callback(interaction):
            # catches the selected members id
            selected_member_ids = member_select.values
            selected_members = [ctx.guild.get_member(int(member_id)) for member_id in selected_member_ids]

            # notify the selected members
            mentions = ", ".join([member.mention for member in selected_members])
            await interaction.response.send_message(f'{mentions} 雞丸 ')

        # matching the callback with the select members
        member_select.callback = member_select_callback

        # create a new view to hold the select menu
        member_view = View()
        member_view.add_item(member_select)

        # notify the selected members
        await interaction.response.send_message(f"'{selected_role.name}' ：", view=member_view, ephemeral=True)

    # callback of select roles
    role_select.callback = role_select_callback

    # create a view to hold the select role
    role_view = View()
    role_view.add_item(role_select)

    # sending a message to notify the user
    await ctx.send("請選擇身份：", view=role_view)


# Flask server setup for uptime monitoring
app = Flask('')

@app.route('/')
def home():
    return "Bot is running!"

def run():
    app.run(host='0.0.0.0', port=8080)

# Start the web server in a separate thread
t = Thread(target=run)
t.start()

# Run the Discord bot
bot.run(os.getenv('DISCORD_BOT_TOKEN'))

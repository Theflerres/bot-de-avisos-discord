import discord
from discord.ext import commands
import sqlite3
from datetime import datetime, timedelta
import os
import re
import tempfile
import asyncio
import atexit

# --- CARREGAR VARIÁVEIS DE AMBIENTE (APENAS PARA O TOKEN) ---
from dotenv import load_dotenv
load_dotenv() # Carrega variáveis do arquivo .env

# --- CONFIGURAÇÕES GERAIS ---
TOKEN = os.getenv("DISCORD_TOKEN")

# --- VERIFICAÇÃO INICIAL DO TOKEN ---
if not TOKEN:
    print("ERRO CRÍTICO: A variável de ambiente DISCORD_TOKEN não está definida no arquivo .env ou no ambiente.")
    print("Crie um arquivo .env na raiz do projeto com a linha: DISCORD_TOKEN='SEU_TOKEN_AQUI'")
    exit() # Termina o script se o token não for encontrado

intents = discord.Intents.default()
intents.message_content = True
intents.guilds = True
intents.members = True
bot = commands.Bot(command_prefix="!", intents=intents)

# --- CONFIGURAÇÕES HARDCODED (Exceto TOKEN) ---
# Adapte estes valores conforme sua necessidade
channels_to_notify = [1377042576787505204] # Adapte ou deixe vazio: [ID_CANAL_1, ID_CANAL_2]
bot_instance = None

ADMIN_APPROVAL_CHANNEL_ID = 1376724217747341322 # Adapte - Pode ser None se não quiser mais notificações de admin
USER_MUSIC_CHANNEL_ID = 1377042576787505204 # Adapte
MAX_SONG_SIZE_MB = 40

LOG_FOLDER = r"C:\Users\jg202\OneDrive\Área de Trabalho\database" # Adapte este caminho
os.makedirs(LOG_FOLDER, exist_ok=True)
DB_PATH = os.path.join(LOG_FOLDER, "bot.db")
LOG_FILE = os.path.join(LOG_FOLDER, "warns_log.txt")

ALLOWED_ROLES = [1282146997909979136, 1282147756814766132] # Adapte - Roles com permissão de "ADM" para músicas
ALLOWED_USER_ID = 299323165937500160 # Adapte - ID de usuário com permissão de "ADM"
CREATOR_NAME = "theflerres" # Adapte - Seu nome de usuário no Discord (sem #tag)

WARN_CHANNEL_ID = 1349002209794195526 # Adapte

DRIVE_FOLDER_ID = "1U8-Pz2YamB1OSP-wAaT8Wzw-3VOKM8Hc" # Adapte - ID da pasta no Google Drive

if not DRIVE_FOLDER_ID:
    print("AVISO: DRIVE_FOLDER_ID não está definido. Funcionalidades de música podem falhar.")
if not channels_to_notify:
    print("AVISO: Nenhum ID de canal de notificação foi configurado em 'channels_to_notify'. As notificações de início/manutenção/desligamento não serão enviadas.")


# --- CONEXÃO COM SQLITE ---
conn = sqlite3.connect(DB_PATH)
cursor = conn.cursor()

# Tabelas
cursor.execute(
    """CREATE TABLE IF NOT EXISTS warns (id INTEGER PRIMARY KEY AUTOINCREMENT, guild_id INTEGER NOT NULL, user_id INTEGER NOT NULL, moderator_id INTEGER NOT NULL, reason TEXT NOT NULL)"""
)
cursor.execute(
    """CREATE TABLE IF NOT EXISTS music_cache (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        youtube_url TEXT UNIQUE,
        drive_link TEXT,
        title TEXT,
        normalized_title TEXT,
        created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    )"""
)
try:
    cursor.execute("SELECT normalized_title FROM music_cache LIMIT 1")
except sqlite3.OperationalError:
    print("Adicionando coluna 'normalized_title' à tabela 'music_cache'.")
    cursor.execute("ALTER TABLE music_cache ADD COLUMN normalized_title TEXT")
    conn.commit()
conn.commit()


# --- NORMALIZAÇÃO DE TÍTULO ---
def normalize_title(title: str) -> str:
    if not title:
        return ""
    norm_title = title.lower()
    norm_title = re.sub(r'\([^)]*\)', '', norm_title)
    norm_title = re.sub(r'\[[^\]]*\]', '', norm_title)
    keywords_to_remove = [
        'official music video', 'music video', 'official video', 'official audio',
        'lyric video', 'lyrics', 'legendado', 'tradução', 'traduzido', 'hd', '4k',
        'hq', 'clipe oficial', 'vídeo oficial', 'áudio oficial', 'full album', 'ao vivo', 'live',
        '(', ')', '[', ']', '{', '}', '|', '-', '_', '"', "'"
    ] # Mantenha as aspas simples ao redor de ( ) [ ] etc. na lista
    for keyword in keywords_to_remove:
        norm_title = norm_title.replace(keyword, '')
    norm_title = re.sub(r'\s+', ' ', norm_title).strip()
    return norm_title

def _populate_normalized_titles_if_empty():
    print("Verificando e preenchendo 'normalized_title' para entradas existentes...")
    all_songs = cursor.execute("SELECT id, title FROM music_cache WHERE normalized_title IS NULL OR normalized_title = ''").fetchall()
    if not all_songs:
        print("Nenhuma entrada para normalizar.")
        return
    count = 0
    for song_id, song_title in all_songs:
        if song_title:
            norm_title = normalize_title(song_title)
            cursor.execute("UPDATE music_cache SET normalized_title = ? WHERE id = ?", (norm_title, song_id))
            count +=1
    if count > 0:
        conn.commit()
    print(f"Normalização de {count} títulos concluída.")

# --- PERMISSÕES ---
def has_permission(ctx_or_interaction): # Usado para checagens gerais de "ADM"
    author = None
    if isinstance(ctx_or_interaction, commands.Context):
        author = ctx_or_interaction.author
    elif isinstance(ctx_or_interaction, discord.Interaction):
        author = ctx_or_interaction.user
    else:
        return False

    if author.name == CREATOR_NAME: return True
    if ALLOWED_USER_ID and author.id == ALLOWED_USER_ID: return True
    # Verifica se o autor tem algum dos cargos permitidos
    # Adicionado hasattr para evitar erro se author.roles não existir (ex: User fora de um Guild)
    if hasattr(author, 'roles'):
        return any(role.id in ALLOWED_ROLES for role in author.roles)
    return False


def is_bot_creator(ctx: commands.Context) -> bool:
    return ctx.author.name == CREATOR_NAME

def is_warn_channel(ctx):
    if not WARN_CHANNEL_ID: return False # Se não houver canal de warn, não restringe
    return ctx.channel.id == WARN_CHANNEL_ID

def can_use_music_command_in_channel(ctx):
    if ctx.channel.id == USER_MUSIC_CHANNEL_ID: return True
    # Admins podem usar no canal de admin também (ou no canal de música, já coberto acima)
    if ADMIN_APPROVAL_CHANNEL_ID and ctx.channel.id == ADMIN_APPROVAL_CHANNEL_ID and has_permission(ctx): return True
    return False

# --- LOG DE ADVERTÊNCIAS ---
def add_warn_log(staff_name, user_name, reason):
    now = datetime.now()
    entry = (f"{now.strftime('%d/%m/%Y %H:%M:%S')}\nStaff: {staff_name}\n- Usuário: {user_name}\n- Motivo: {reason}\n" + "-"*30 + "\n")
    with open(LOG_FILE, "a", encoding="utf-8") as f: f.write(entry)

# --- GOOGLE DRIVE ---
CLIENT_SECRET_FILE = os.path.join(os.getcwd(), "client_secret.json")
CREDENTIALS_PATH = os.path.join(LOG_FOLDER, "credentials.json")
# DRIVE_FOLDER_ID já definido nas configurações hardcoded

# Imports que faltavam para o Google Drive e yt-dlp
from yt_dlp import YoutubeDL
from google.oauth2.credentials import Credentials
from google_auth_oauthlib.flow import InstalledAppFlow
from google.auth.transport.requests import Request
from googleapiclient.discovery import build
from googleapiclient.http import MediaFileUpload

def get_drive_service():
    scopes = ['https://www.googleapis.com/auth/drive.file']
    creds = None
    if os.path.exists(CREDENTIALS_PATH): creds = Credentials.from_authorized_user_file(CREDENTIALS_PATH, scopes)
    if not creds or not creds.valid:
        if creds and creds.expired and creds.refresh_token: creds.refresh(Request())
        else:
            if not os.path.exists(CLIENT_SECRET_FILE):
                print(f"ERRO CRÍTICO: '{CLIENT_SECRET_FILE}' não encontrado. Certifique-se de que está na raiz do projeto.")
                return None
            flow = InstalledAppFlow.from_client_secrets_file(CLIENT_SECRET_FILE, scopes)
            creds = flow.run_local_server(port=0)
        with open(CREDENTIALS_PATH, 'w') as token_file: token_file.write(creds.to_json())
    return build('drive', 'v3', credentials=creds)

def format_duration(seconds):
    if seconds is None: return "N/A"
    return f"{int(seconds // 60):02d}:{int(seconds % 60):02d}"

def format_filesize(bytes_size):
    if bytes_size is None or bytes_size == 0: return "N/A"
    return f"{bytes_size / (1024 * 1024):.2f} MB"

def get_song_info(youtube_url):
    ydl_opts = {'format': 'bestaudio/best', 'quiet': True, 'noplaylist': True}
    with YoutubeDL(ydl_opts) as ydl:
        try: info = ydl.extract_info(youtube_url, download=False)
        except Exception as e:
            print(f"[GET_SONG_INFO_ERROR] URL: {youtube_url}, Erro: {e}")
            return None, None, None, None
        title = info.get('title', 'Título Desconhecido')
        duration = info.get('duration')
        uploader = info.get('uploader', 'Uploader Desconhecido')
        filesize = next((f.get('filesize') or f.get('filesize_approx') for f in info.get('formats', []) if f.get('vcodec') == 'none' and f.get('acodec') != 'none' and (f.get('filesize') or f.get('filesize_approx'))), None)
        if filesize is None: filesize = info.get('filesize') or info.get('filesize_approx')
        return title, duration, filesize, uploader

def download_audio_file(youtube_url, temp_dir_path):
    ydl_opts = {
        'format': 'bestaudio/best',
        'outtmpl': os.path.join(temp_dir_path, '%(id)s.%(ext)s'),
        'postprocessors': [{'key': 'FFmpegExtractAudio', 'preferredcodec': 'mp3', 'preferredquality': '192'}],
        'quiet': True, 'noplaylist': True,
    }
    with YoutubeDL(ydl_opts) as ydl:
        info = ydl.extract_info(youtube_url, download=True)
        filename = ydl.prepare_filename(info)
        base, _ = os.path.splitext(filename)
        return base + ".mp3", info.get('title', 'audio')

def upload_to_drive(file_path, title):
    if not DRIVE_FOLDER_ID:
        raise Exception("ID da pasta do Google Drive (DRIVE_FOLDER_ID) não configurado.")
    service = get_drive_service()
    if not service: raise Exception("Serviço do Google Drive não autenticado.")
    file_name_on_drive = re.sub(r'[<>:"/\\|?*]', '', f"{title}.mp3") # Remove caracteres inválidos para nome de arquivo
    metadata = {'name': file_name_on_drive, 'parents': [DRIVE_FOLDER_ID]}
    media = MediaFileUpload(file_path, mimetype='audio/mpeg', resumable=True)
    file = service.files().create(body=metadata, media_body=media, fields='id').execute()
    fid = file.get('id')
    service.permissions().create(fileId=fid, body={'role':'reader','type':'anyone'}).execute()
    return f"https://drive.google.com/uc?export=download&id={fid}"

# --- VIEW DE APROVAÇÃO (Não será mais usada diretamente no fluxo principal, mas mantida caso queira reativar) ---
class ApprovalView(discord.ui.View):
    def __init__(self, requester: discord.Member = None, timeout_duration=1800): # Timeout de 30 minutos
        super().__init__(timeout=timeout_duration)
        self.approved = None
        self.interaction_user = None
        self.requester = requester
        self.message_to_edit = None # Mensagem que a view está anexada, para editar no timeout
    async def on_timeout(self):
        self.approved = False # Considera timeout como não aprovado
        for item in self.children: # Desabilita botões
            item.disabled = True
        if self.message_to_edit:
            try:
                requester_mention = self.requester.mention if self.requester else "o solicitante"
                # Adiciona aviso de timeout à mensagem original
                await self.message_to_edit.edit(content=self.message_to_edit.content + f"\n\n**Tempo esgotado!** A solicitação de {requester_mention} foi automaticamente rejeitada.", view=None) # Remove a view
            except discord.NotFound: # Mensagem pode ter sido deletada
                pass
        self.stop() # Pára a view

    @discord.ui.button(label="Aprovar ✅", style=discord.ButtonStyle.green)
    async def approve_button(self, interaction: discord.Interaction, button: discord.ui.Button):
        if not has_permission(interaction): # Somente usuários com permissão podem aprovar
            await interaction.response.send_message("Você não tem permissão para aprovar/rejeitar.", ephemeral=True)
            return
        self.approved = True
        self.interaction_user = interaction.user
        for item in self.children: item.disabled = True # Desabilita botões
        # Edita a mensagem original para mostrar quem aprovou
        await interaction.response.edit_message(content=interaction.message.content + f"\n\n**Aprovado por {interaction.user.mention}**", view=self) # Manter a view para desabilitar botões
        self.stop()

    @discord.ui.button(label="Rejeitar ❌", style=discord.ButtonStyle.red)
    async def reject_button(self, interaction: discord.Interaction, button: discord.ui.Button):
        if not has_permission(interaction): # Somente usuários com permissão podem rejeitar
            await interaction.response.send_message("Você não tem permissão para aprovar/rejeitar.", ephemeral=True)
            return
        self.approved = False
        self.interaction_user = interaction.user
        for item in self.children: item.disabled = True # Desabilita botões
        # Edita a mensagem original para mostrar quem rejeitou
        await interaction.response.edit_message(content=interaction.message.content + "\n\n**Solicitação Rejeitada.**", view=self) # Manter a view para desabilitar botões
        self.stop()

# --- PROCESSAMENTO CENTRAL DE MÚSICA ---
async def _perform_song_download_upload_cache(youtube_url: str, initial_title: str):
    print(f"[_PERFORM_SONG] Iniciando para: {youtube_url} (Título inicial: {initial_title})")
    actual_dl_title = initial_title # Começa com o título que temos
    drive_link = None
    normalized_final_title = ""

    with tempfile.TemporaryDirectory() as temp_dir:
        print(f"[_PERFORM_SONG] Diretório temporário: {temp_dir}")
        # download_audio_file retorna o caminho do arquivo e o título REAL do vídeo
        audio_path, downloaded_title = download_audio_file(youtube_url, temp_dir)
        if downloaded_title: # Se yt-dlp retornou um título, use esse como o "oficial"
            actual_dl_title = downloaded_title
        print(f"[_PERFORM_SONG] Download concluído: {audio_path} (Título real: {actual_dl_title})")

        print(f"[_PERFORM_SONG] Iniciando upload para o Drive: {actual_dl_title}")
        drive_link = upload_to_drive(audio_path, actual_dl_title) # Usa o título real para nomear no Drive
        print(f"[_PERFORM_SONG] Upload concluído. Link: {drive_link}")

    # Normaliza o título REAL do vídeo para armazenar no cache
    normalized_final_title = normalize_title(actual_dl_title)
    print(f"[_PERFORM_SONG] Adicionando ao cache: URL='{youtube_url}', Link='{drive_link}', Título='{actual_dl_title}', Título Normalizado='{normalized_final_title}'")
    cursor.execute(
        "INSERT INTO music_cache (youtube_url, drive_link, title, normalized_title) VALUES (?,?,?,?)",
        (youtube_url, drive_link, actual_dl_title, normalized_final_title)
    )
    conn.commit()
    print(f"[_PERFORM_SONG] Adicionado ao cache com sucesso.")

    return actual_dl_title, drive_link # Retorna o título real e o link do Drive

# MODIFIED FUNCTION: process_music_with_approval
async def process_music_with_approval(ctx: commands.Context, youtube_url: str, requesting_user: discord.Member, initial_title_from_info: str):
    admin_channel = bot.get_channel(ADMIN_APPROVAL_CHANNEL_ID) if ADMIN_APPROVAL_CHANNEL_ID else None

    title_for_display = initial_title_from_info
    _, duration_s, filesize_bytes, uploader = get_song_info(youtube_url)

    if title_for_display is None:
        await ctx.send("❌ Não foi possível obter informações da música (título nulo). Verifique o link ou tente novamente mais tarde.")
        return

    filesize_mb = (filesize_bytes / (1024 * 1024)) if filesize_bytes else 0
    print(f"[PROCESS_MUSIC] Info para processamento: Título='{title_for_display}', Duração={format_duration(duration_s)}, Tamanho={format_filesize(filesize_bytes)}, Uploader='{uploader}'")

    if filesize_mb > MAX_SONG_SIZE_MB:
        msg = (f"❌ A música **{title_for_display}** excede o limite de {MAX_SONG_SIZE_MB}MB ({format_filesize(filesize_bytes)}).")
        await ctx.send(msg)
        if admin_channel: # Notifica admin sobre a tentativa, mesmo que não haja aprovação
            await admin_channel.send(f"ℹ️ {requesting_user.mention} tentou adicionar '{title_for_display}' ({format_filesize(filesize_bytes)}) acima do limite. A adição foi bloqueada automaticamente.")
        return

    # --- MODIFICAÇÃO: REMOVER FLUXO DE APROVAÇÃO ---
    # Todas as solicitações válidas (que passam na checagem de tamanho) serão processadas diretamente.
    
    print(f"[PROCESS_MUSIC_DIRECT] Solicitação de {requesting_user.name} para '{title_for_display}'. Processando diretamente (sem aprovação).")
    
    # Mensagem para o usuário informando que o processamento começou
    status_message = await ctx.send(f"⏳ {requesting_user.mention}, processando sua solicitação para **{title_for_display}** ({format_filesize(filesize_bytes)})...")
    
    try:
        actual_title, drive_link = await _perform_song_download_upload_cache(youtube_url, title_for_display)
        await status_message.edit(content=f"🎶 Música **{actual_title}** pronta, {requesting_user.mention}!\n{drive_link}")
        
        # Notifica o canal de admin (se configurado) que uma música foi adicionada automaticamente.
        if admin_channel:
            # Não envia notificação duplicada se o comando foi usado no próprio canal de admin por um admin
            if not (ctx.channel.id == ADMIN_APPROVAL_CHANNEL_ID and has_permission(ctx)):
                 await admin_channel.send(f"ℹ️ {requesting_user.mention} adicionou **{actual_title}** diretamente (sem necessidade de aprovação).\nLink: {drive_link}")

    except Exception as e:
        error_msg_user = f"❌ Erro ao processar sua solicitação para **{title_for_display}**: {e}"
        await status_message.edit(content=error_msg_user)
        print(f"[ERROR_DIRECT_ADD] User: {requesting_user.name} - Song: {title_for_display} - Error: {e}\nTraceback: {e.__traceback__}")
        
        # Notifica o canal de admin sobre o erro.
        if admin_channel:
            # Não envia notificação duplicada se o comando foi usado no próprio canal de admin por um admin
            if not (ctx.channel.id == ADMIN_APPROVAL_CHANNEL_ID and has_permission(ctx)):
                await admin_channel.send(f"❌ Erro ao processar a adição direta de '{title_for_display}' por {requesting_user.mention}: {e}")
    return
    # --- FIM DA MODIFICAÇÃO ---

    # O código original do fluxo de aprovação abaixo não será mais executado.
    # is_admin_request = has_permission(ctx)
    # print(f"[PROCESS_MUSIC] Solicitante: {requesting_user.name}, É Admin? {is_admin_request}")

    # if is_admin_request:
    #     # ... (lógica original para admin) ...
    #     return

    # # ... (lógica original para fluxo de aprovação de usuário comum) ...


# --- COMANDOS ---
@bot.command(name="cmds")
async def cmds(ctx):
    embed = discord.Embed(title="📜 Comandos do Bot", color=discord.Color.blue())
    embed.add_field(name="!adv [usuário] [motivo]", value="Adverte um usuário.", inline=False)
    embed.add_field(name="!advs [usuário]", value="Mostra advertências de um usuário.", inline=False)
    embed.add_field(name="!remove-adv [usuário] [índice]", value="Remove uma advertência.", inline=False)
    embed.add_field(name="!remove-all-advs [usuário]", value="Remove todas advertências.", inline=False)

    music_channel_mention = f"<#{USER_MUSIC_CHANNEL_ID}>" if USER_MUSIC_CHANNEL_ID else "canal de música"
    admin_music_channel_mention = f"<#{ADMIN_APPROVAL_CHANNEL_ID}>" if ADMIN_APPROVAL_CHANNEL_ID else "canal de admin (se configurado)"

    embed.add_field(name=f"!cmusica [link_youtube] (no {music_channel_mention} ou {admin_music_channel_mention} para ADMs)", value="Adiciona música ao Drive.", inline=False) # Removido "Solicita/"
    embed.add_field(name=f"!bmusica [busca] (no {music_channel_mention} ou {admin_music_channel_mention} para ADMs)", value="Busca/Adiciona música ao Drive.", inline=False) # Removido "Solicita/"

    if is_bot_creator(ctx): # Mostra comandos de criador apenas para o criador
        embed.add_field(name="!manutencao (SOMENTE CRIADOR - DM)", value="Anuncia que o bot entrará em manutenção.", inline=False)
        embed.add_field(name="!desligando (SOMENTE CRIADOR - DM)", value="Anuncia que o bot está desligando e o desliga.", inline=False)

    await ctx.send(embed=embed)


@bot.command(name="adv")
@commands.check(has_permission)
@commands.check(is_warn_channel)
async def adv(ctx, user: discord.Member, *, reason: str):
    cursor.execute("INSERT INTO warns (guild_id, user_id, moderator_id, reason) VALUES (?, ?, ?, ?)", (ctx.guild.id, user.id, ctx.author.id, reason))
    conn.commit(); add_warn_log(ctx.author.name, user.name, reason)
    count = cursor.execute("SELECT COUNT(*) FROM warns WHERE guild_id=? AND user_id=?", (ctx.guild.id, user.id)).fetchone()[0]
    x_emojis = "❌ " * count
    embed_dm = discord.Embed(title="Você recebeu uma advertência", description=f"Motivo: {reason}\n{x_emojis}", color=discord.Color.red())
    try: await user.send(embed=embed_dm)
    except discord.Forbidden: await ctx.send(f"Não foi possível enviar DM para {user.mention}. Advertência registrada.")
    await ctx.send(embed=discord.Embed(title="Advertência registrada", description=f"Usuário: {user.mention}\nMotivo: {reason}\n{x_emojis}", color=discord.Color.blue()))

@adv.error
async def adv_error(ctx, error):
    if isinstance(error, commands.CheckFailure):
        warn_ch_mention = f"<#{WARN_CHANNEL_ID}>" if WARN_CHANNEL_ID else "canal de advertências configurado"
        await ctx.send(f"🚫 Este comando só pode ser usado no {warn_ch_mention} por usuários autorizados.")
    elif isinstance(error, commands.MissingRequiredArgument):
        await ctx.send(f"🚫 Uso incorreto. Falta argumento: `{error.param.name}`. Use `!adv @usuario <motivo>`")
    elif isinstance(error, commands.MemberNotFound):
        await ctx.send(f"🚫 Usuário `{error.argument}` não encontrado.")
    else:
        print(f"Erro não tratado em !adv: {error}") # Log para outros erros

@bot.command(name="advs")
@commands.check(has_permission)
@commands.check(is_warn_channel)
async def advs(ctx, user: discord.Member):
    rows = cursor.execute("SELECT moderator_id, reason FROM warns WHERE guild_id=? AND user_id=?", (ctx.guild.id, user.id)).fetchall()
    if not rows: return await ctx.send(f"{user.mention} não tem advertências.")
    embed = discord.Embed(title=f"Advertências de {user.name}", color=discord.Color.blue())
    for idx, (mod_id, reason) in enumerate(rows, start=1):
        mod = ctx.guild.get_member(mod_id) # Tenta encontrar o membro no servidor
        embed.add_field(name=f"{idx}.", value=f"Moderador: {mod.name if mod else f'ID {mod_id} (Fora do servidor?)'}\nMotivo: {reason}", inline=False)
    await ctx.send(embed=embed)

@advs.error
async def advs_error(ctx, error):
    if isinstance(error, commands.CheckFailure):
        warn_ch_mention = f"<#{WARN_CHANNEL_ID}>" if WARN_CHANNEL_ID else "canal de advertências configurado"
        await ctx.send(f"🚫 Este comando só pode ser usado no {warn_ch_mention} por usuários autorizados.")
    elif isinstance(error, commands.MissingRequiredArgument):
        await ctx.send(f"🚫 Uso incorreto. Falta argumento: `{error.param.name}`. Use `!advs @usuario`")
    elif isinstance(error, commands.MemberNotFound):
        await ctx.send(f"🚫 Usuário `{error.argument}` não encontrado.")
    else:
        print(f"Erro não tratado em !advs: {error}")

@bot.command(name="remove-adv")
@commands.check(has_permission)
@commands.check(is_warn_channel)
async def remove_adv(ctx, user: discord.Member, index: int):
    warns_data = cursor.execute("SELECT id FROM warns WHERE guild_id=? AND user_id=? ORDER BY id", (ctx.guild.id, user.id)).fetchall() # Adicionado ORDER BY
    if not warns_data or index < 1 or index > len(warns_data): return await ctx.send("Índice de advertência inválido.")
    cursor.execute("DELETE FROM warns WHERE id=?", (warns_data[index-1][0],)); conn.commit()
    await ctx.send(f"Advertência #{index} de {user.mention} removida.")

@remove_adv.error
async def remove_adv_error(ctx, error):
    if isinstance(error, commands.CheckFailure):
        warn_ch_mention = f"<#{WARN_CHANNEL_ID}>" if WARN_CHANNEL_ID else "canal de advertências configurado"
        await ctx.send(f"🚫 Este comando só pode ser usado no {warn_ch_mention} por usuários autorizados.")
    elif isinstance(error, commands.MissingRequiredArgument):
        await ctx.send(f"🚫 Uso incorreto. Faltam argumentos. Use `!remove-adv @usuario <índice>`")
    elif isinstance(error, commands.MemberNotFound):
        await ctx.send(f"🚫 Usuário `{error.argument}` não encontrado.")
    elif isinstance(error, commands.BadArgument): # Se o índice não for um número
        await ctx.send(f"🚫 O índice da advertência deve ser um número.")
    else:
        print(f"Erro não tratado em !remove-adv: {error}")


@bot.command(name="remove-all-advs")
@commands.check(has_permission)
@commands.check(is_warn_channel)
async def remove_all_advs(ctx, user: discord.Member):
    deleted_count = cursor.execute("DELETE FROM warns WHERE guild_id=? AND user_id=?", (ctx.guild.id, user.id)).rowcount
    conn.commit()
    if deleted_count > 0:
        await ctx.send(f"Todas as {deleted_count} advertências de {user.mention} foram removidas.")
    else:
        await ctx.send(f"{user.mention} não possuía advertências para remover.")

@remove_all_advs.error
async def remove_all_advs_error(ctx, error):
    if isinstance(error, commands.CheckFailure):
        warn_ch_mention = f"<#{WARN_CHANNEL_ID}>" if WARN_CHANNEL_ID else "canal de advertências configurado"
        await ctx.send(f"🚫 Este comando só pode ser usado no {warn_ch_mention} por usuários autorizados.")
    elif isinstance(error, commands.MissingRequiredArgument):
        await ctx.send(f"🚫 Uso incorreto. Falta argumento: `{error.param.name}`. Use `!remove-all-advs @usuario`")
    elif isinstance(error, commands.MemberNotFound):
        await ctx.send(f"🚫 Usuário `{error.argument}` não encontrado.")
    else:
        print(f"Erro não tratado em !remove-all-advs: {error}")


@bot.command(name="cmusica", aliases=["Cmusica"])
@commands.check(can_use_music_command_in_channel)
async def cmusica(ctx, youtube_url: str):
    if not re.match(r"https?://(www\.)?(youtube\.com/watch\?v=|youtu\.be/|music\.youtube\.com/watch\?v=)", youtube_url):
        return await ctx.send("❌ URL do YouTube inválida (aceita youtube.com, youtu.be, music.youtube.com).")

    row_url = cursor.execute("SELECT drive_link, title FROM music_cache WHERE youtube_url=?", (youtube_url,)).fetchone()
    if row_url:
        await ctx.send(f"🔁 Música **{row_url[1]}** já no Drive (encontrada por URL):\n{row_url[0]}")
        return

    title_from_info, _, _, _ = get_song_info(youtube_url)
    if not title_from_info:
        await ctx.send("❌ Não foi possível obter informações da música. Verifique o link ou tente novamente.")
        return

    normalized_title_from_info = normalize_title(title_from_info)
    if not normalized_title_from_info:
        print(f"[WARN_CMUSICA] Título normalizado vazio para '{title_from_info}' (URL: {youtube_url}). Prosseguindo sem checagem de título normalizado, mas isso pode levar a duplicatas não detectadas.")
    else:
        row_norm_title = cursor.execute("SELECT drive_link, title FROM music_cache WHERE normalized_title=?", (normalized_title_from_info,)).fetchone()
        if row_norm_title:
            await ctx.send(f"⚠️ Música similar **{row_norm_title[1]}** já parece estar no Drive (encontrada por título similar a '{title_from_info}').\nLink: {row_norm_title[0]}\nSe esta for uma música diferente, peça a um ADM para verificar ou contate o suporte. Por enquanto, esta solicitação não será processada para evitar duplicatas.")
            if ADMIN_APPROVAL_CHANNEL_ID and (admin_ch := bot.get_channel(ADMIN_APPROVAL_CHANNEL_ID)):
                 if not (ctx.channel.id == ADMIN_APPROVAL_CHANNEL_ID and has_permission(ctx)): # Não notifica se o admin já está no canal de admin
                    await admin_ch.send(f"ℹ️ Possível duplicata por título: {ctx.author.mention} tentou adicionar '{title_from_info}' ({youtube_url}), mas '{row_norm_title[1]}' ({row_norm_title[0]}) já existe com título normalizado similar. A adição foi prevenida.")
            return

    # Como a aprovação foi removida, chamamos diretamente o processamento.
    await process_music_with_approval(ctx, youtube_url, ctx.author, initial_title_from_info=title_from_info)

@cmusica.error # Tratador de erros específico para !cmusica
async def cmusica_error(ctx, error):
    if isinstance(error, commands.CheckFailure):
        music_ch_mention = f"<#{USER_MUSIC_CHANNEL_ID}>" if USER_MUSIC_CHANNEL_ID else "canal de música configurado"
        admin_ch_mention = f"<#{ADMIN_APPROVAL_CHANNEL_ID}>" if ADMIN_APPROVAL_CHANNEL_ID else "canal de administração musical (se configurado)"
        await ctx.send(f"🚫 O comando `!cmusica` só pode ser usado no {music_ch_mention} (ou no {admin_ch_mention} por ADMs).")
    elif isinstance(error, commands.MissingRequiredArgument):
        await ctx.send(f"🚫 Uso incorreto. Você precisa fornecer um link do YouTube. Ex: `!cmusica <link_youtube>`")
    else: # Outros erros não tratados especificamente aqui irão para o on_command_error geral
        print(f"Erro não tratado em !cmusica: {error}") # Log para outros erros


@bot.command(name="bmusica")
@commands.check(can_use_music_command_in_channel)
async def bmusica(ctx, *, search: str):
    normalized_search = normalize_title(search)
    query = """
    SELECT title, drive_link,
            CASE
                WHEN normalized_title LIKE ? THEN 1 -- Correspondência exata normalizada
                WHEN title LIKE ? THEN 2             -- Correspondência exata no título original
                WHEN normalized_title LIKE ? THEN 3 -- Correspondência parcial normalizada
                WHEN title LIKE ? THEN 4             -- Correspondência parcial no título original
                ELSE 5
            END as relevance
    FROM music_cache
    WHERE normalized_title LIKE ? OR title LIKE ? OR title LIKE ? OR normalized_title LIKE ? -- Expandido para cobrir mais casos
    ORDER BY relevance, title COLLATE NOCASE
    LIMIT 10
    """
    # Parâmetros para a query SQL, cobrindo exato e parcial
    exact_norm_search = normalized_search if normalized_search else search # se normalizado for vazio, usa busca original
    exact_orig_search = search
    like_normalized_search = f"%{exact_norm_search}%"
    like_original_search = f"%{exact_orig_search}%"

    rows = cursor.execute(query, (
        exact_norm_search, exact_orig_search, # for relevance CASE
        like_normalized_search, like_original_search, # for relevance CASE (parcial)
        exact_norm_search, exact_orig_search, # for WHERE clause (exato)
        like_normalized_search, like_original_search # for WHERE clause (parcial)
        )).fetchall()


    if rows:
        description_parts = [f"{i+1}. **{t}**\n   [Link do Drive]({l})" for i, (t, l, _) in enumerate(rows)]
        embed = discord.Embed(title=f"🎶 Músicas encontradas no cache para '{search}':", color=discord.Color.green(), description="\n".join(description_parts))
        
        total_found_query = "SELECT COUNT(*) FROM music_cache WHERE normalized_title LIKE ? OR title LIKE ? OR title LIKE ? OR normalized_title LIKE ?"
        total_count = cursor.execute(total_found_query, (exact_norm_search, exact_orig_search, like_normalized_search, like_original_search)).fetchone()[0]
        if total_count > 10: embed.set_footer(text=f"Mostrando {len(rows)} de {total_count} resultados. Seja mais específico se não encontrar.")
        else: embed.set_footer(text=f"Encontrados {len(rows)} resultado(s).")
        await ctx.send(embed=embed)
        return # Importante: Retorna após mostrar os resultados da busca

    # Se não encontrou nada no cache, pergunta pelo link.
    initial_response_msg = await ctx.send(f"🔍 Nenhuma música encontrada no cache para '**{search}**'.\nEnvie o link do YouTube desta música se desejar adicioná-la (válido por 60 segundos) ou digite 'cancelar':")
    
    def check_msg_author(msg): # Checa se a mensagem é do mesmo autor e canal
        if msg.author == ctx.author and msg.channel == ctx.channel:
            if msg.content.lower() == 'cancelar': return True
            # Valida se é um link do YouTube
            if re.match(r"https?://(www\.)?(youtube\.com/watch\?v=|youtu\.be/|music\.youtube\.com/watch\?v=)", msg.content): return True
        return False

    try:
        user_link_message = await bot.wait_for("message", timeout=60.0, check=check_msg_author)
        
        if user_link_message.content.lower() == 'cancelar':
            await initial_response_msg.edit(content=f"Solicitação de adição para '**{search}**' cancelada.")
            try: await user_link_message.delete() # Tenta deletar a mensagem "cancelar"
            except discord.HTTPException: pass
            return

        youtube_url = user_link_message.content
        try: # Tenta deletar a mensagem do usuário com o link e a mensagem do bot
            await user_link_message.delete()
            await initial_response_msg.delete() # Deleta a mensagem "Envie o link..."
        except discord.HTTPException: pass # Ignora se não conseguir deletar

        # Checa novamente se a URL já existe no cache (caso tenha sido adicionada enquanto o usuário procurava)
        row_url = cursor.execute("SELECT drive_link, title FROM music_cache WHERE youtube_url=?", (youtube_url,)).fetchone()
        if row_url:
            await ctx.send(f"🔁 Música **{row_url[1]}** já no Drive (encontrada por URL):\n{row_url[0]}")
            return

        title_from_info, _, _, _ = get_song_info(youtube_url)
        if not title_from_info:
            await ctx.send("❌ Não foi possível obter informações da música do link fornecido. Verifique o link ou tente novamente.")
            return

        normalized_title_from_info = normalize_title(title_from_info)
        if normalized_title_from_info: # Apenas checa título normalizado se ele não for vazio
            row_norm_title = cursor.execute("SELECT drive_link, title FROM music_cache WHERE normalized_title=?", (normalized_title_from_info,)).fetchone()
            if row_norm_title:
                await ctx.send(f"⚠️ Música similar **{row_norm_title[1]}** já parece estar no Drive (encontrada por título similar a '{title_from_info}').\nLink: {row_norm_title[0]}\nSe esta for uma música diferente, peça a um ADM para verificar ou use `!cmusica` com o link direto. Por enquanto, esta solicitação não será processada para evitar duplicatas.")
                if ADMIN_APPROVAL_CHANNEL_ID and (admin_ch := bot.get_channel(ADMIN_APPROVAL_CHANNEL_ID)):
                    if not (ctx.channel.id == ADMIN_APPROVAL_CHANNEL_ID and has_permission(ctx)):
                        await admin_ch.send(f"ℹ️ Possível duplicata por título (via bmusica): {ctx.author.mention} tentou adicionar '{title_from_info}' ({youtube_url}), mas '{row_norm_title[1]}' ({row_norm_title[0]}) já existe com título normalizado similar. A adição foi prevenida.")
                return
        
        # Como a aprovação foi removida, chamamos diretamente o processamento.
        await process_music_with_approval(ctx, youtube_url, ctx.author, initial_title_from_info=title_from_info)

    except asyncio.TimeoutError:
        await initial_response_msg.edit(content=f"⏳ Tempo esgotado para adicionar o link para '**{search}**'.")
    except Exception as e:
        print(f"[ERROR_BMUSICA_PROCESS] Erro durante o processamento de bmusica para '{search}': {e}\nTraceback: {e.__traceback__}")
        await ctx.send(f"❌ Ocorreu um erro inesperado ao tentar adicionar a música para '{search}'. Tente novamente mais tarde.")

@bmusica.error # Tratador de erros específico para !bmusica
async def bmusica_error(ctx, error):
    if isinstance(error, commands.CheckFailure):
        music_ch_mention = f"<#{USER_MUSIC_CHANNEL_ID}>" if USER_MUSIC_CHANNEL_ID else "canal de música configurado"
        admin_ch_mention = f"<#{ADMIN_APPROVAL_CHANNEL_ID}>" if ADMIN_APPROVAL_CHANNEL_ID else "canal de administração musical (se configurado)"
        await ctx.send(f"🚫 O comando `!bmusica` só pode ser usado no {music_ch_mention} (ou no {admin_ch_mention} por ADMs).")
    elif isinstance(error, commands.MissingRequiredArgument):
        await ctx.send(f"🚫 Uso incorreto. Você precisa fornecer um termo para busca. Ex: `!bmusica <nome da música>`")
    else: # Outros erros não tratados especificamente aqui irão para o on_command_error geral
        print(f"Erro não tratado em !bmusica: {error}") # Log para outros erros


@bot.command(name="manutencao")
@commands.dm_only()
@commands.check(is_bot_creator)
async def manutencao_cmd(ctx: commands.Context):
    """(SOMENTE CRIADOR - DM) Anuncia que o bot entrará em manutenção."""
    mensagem = "# =====**Bot em Manutenção**=====\nO bot ficará temporariamente offline para manutenção. Agradecemos a compreensão!"
    canais_notificados = 0
    canais_falha = 0

    if not channels_to_notify: # Verifica se a lista não está vazia
        await ctx.send("⚠️ Nenhum canal de notificação configurado em `channels_to_notify` para enviar o aviso.")
        return

    for channel_id in channels_to_notify:
        channel = bot.get_channel(channel_id)
        if channel:
            try:
                await channel.send(mensagem)
                canais_notificados += 1
            except discord.Forbidden:
                print(f"Erro: Sem permissão para enviar mensagem no canal {channel_id} (manutenção).")
                canais_falha += 1
            except Exception as e:
                print(f"Erro ao enviar mensagem para o canal {channel_id} (manutenção): {e}")
                canais_falha += 1
        else:
            print(f"Aviso: Canal {channel_id} (manutenção) não encontrado.")
            canais_falha += 1

    feedback_msg = f"📢 Anúncio de manutenção enviado!"
    if canais_notificados > 0:
        feedback_msg += f"\n✅ Notificado em {canais_notificados} canal(is)."
    if canais_falha > 0:
        feedback_msg += f"\n❌ Falha ao notificar {canais_falha} canal(is) (verifique o console)."
    await ctx.send(feedback_msg)

@manutencao_cmd.error
async def manutencao_cmd_error(ctx, error):
    if isinstance(error, commands.NoPrivateMessage):
        await ctx.author.send(f"🚫 O comando `!manutencao` só pode ser usado em Mensagem Direta (DM).")
    elif isinstance(error, commands.CheckFailure):
        await ctx.author.send(f"🚫 Apenas o criador do bot ({CREATOR_NAME}) pode usar este comando.")


@bot.command(name="desligando")
@commands.dm_only()
@commands.check(is_bot_creator)
async def desligando_cmd(ctx: commands.Context):
    """(SOMENTE CRIADOR - DM) Anuncia que o bot está desligando e o desliga."""
    mensagem = "# =====**Bot Desligando**=====\nO bot está sendo desligado agora. Até a próxima!"
    canais_notificados = 0
    canais_falha = 0

    if not channels_to_notify:
        await ctx.send("⚠️ Nenhum canal de notificação configurado. Desligando mesmo assim...")
    else:
        for channel_id in channels_to_notify:
            channel = bot.get_channel(channel_id)
            if channel:
                try:
                    await channel.send(mensagem)
                    canais_notificados += 1
                except discord.Forbidden:
                    print(f"Erro: Sem permissão para enviar mensagem no canal {channel_id} (desligando).")
                    canais_falha += 1
                except Exception as e:
                    print(f"Erro ao enviar mensagem para o canal {channel_id} (desligando): {e}")
                    canais_falha += 1
            else:
                print(f"Aviso: Canal {channel_id} (desligando) não encontrado.")
                canais_falha += 1

    feedback_msg = f"🔌 Anúncio de desligamento enviado. O bot será desligado agora."
    if canais_notificados > 0:
        feedback_msg += f"\n✅ Notificado em {canais_notificados} canal(is)."
    if canais_falha > 0:
        feedback_msg += f"\n❌ Falha ao notificar {canais_falha} canal(is) (verifique o console)."

    await ctx.send(feedback_msg)
    print(f"Comando !desligando recebido de {ctx.author.name}. Desligando o bot...")
    await bot.close() # Encerra a conexão do bot com o Discord

@desligando_cmd.error
async def desligando_cmd_error(ctx, error):
    if isinstance(error, commands.NoPrivateMessage):
        await ctx.author.send(f"🚫 O comando `!desligando` só pode ser usado em Mensagem Direta (DM).")
    elif isinstance(error, commands.CheckFailure):
        await ctx.author.send(f"🚫 Apenas o criador do bot ({CREATOR_NAME}) pode usar este comando.")


# --- EVENTOS ---
@bot.event
async def on_ready():
    global bot_instance
    bot_instance = bot # Para o shutdown_handler
    print(f"✅ Bot {bot.user.name} online e conectado ao Discord!")
    _populate_normalized_titles_if_empty() # Popula títulos normalizados ao iniciar

    if channels_to_notify: # Apenas tenta notificar se a lista não estiver vazia
        for channel_id in channels_to_notify:
            channel = bot.get_channel(channel_id)
            if channel:
                try:
                    # Usar create_task para não bloquear o on_ready
                    asyncio.create_task(channel.send("🟢 Bot iniciado e pronto para operar!"))
                except discord.Forbidden:
                    print(f"Erro on_ready: Sem permissão para enviar mensagem no canal {channel_id}.")
                except Exception as e:
                    print(f"Erro on_ready ao enviar mensagem para o canal {channel_id}: {e}")
            else:
                print(f"Aviso on_ready: Canal de notificação {channel_id} não encontrado.")
    else:
        print("Nenhum canal de notificação configurado para a mensagem de 'bot online'.")


@bot.event
async def on_command_error(ctx, error):
    cmd_name = ctx.command.name if ctx.command else "Comando desconhecido"

    if isinstance(error, commands.CheckFailure):
        # Checagens de permissão e canal já são tratadas nos tratadores de erro específicos dos comandos.
        # Se chegou aqui, é uma checagem não coberta ou um comando sem tratador específico.
        # Para !manutencao e !desligando, a checagem DM only e is_bot_creator já é tratada em seus .error
        if not isinstance(ctx.channel, discord.DMChannel) and cmd_name in ["manutencao", "desligando"]:
            # Isso não deveria acontecer se o @commands.dm_only() estiver funcionando, mas como fallback.
            pass # Já tratado pelo @commands.dm_only() ou pelo .error específico
        elif cmd_name in ["adv", "advs", "remove-adv", "remove-all-advs"] and not is_warn_channel(ctx):
             pass # Já tratado pelo .error específico
        elif cmd_name in ["cmusica", "bmusica"] and not can_use_music_command_in_channel(ctx):
            pass # Já tratado pelo .error específico
        else: # Permissão geral negada para outros comandos
            await ctx.send(f"🚫 Você não tem permissão para usar o comando `!{cmd_name}` ou não está no canal correto.")
        return # Evita mensagens duplicadas de erro

    elif isinstance(error, commands.MissingRequiredArgument):
        await ctx.send(f"🚫 Argumento faltando para o comando `!{cmd_name}`: `{error.param.name}`. Use `!cmds` para ver a lista de comandos ou verifique a sintaxe do comando.")
    elif isinstance(error, commands.CommandNotFound):
        pass # Ignora comandos não encontrados para não poluir o chat
    elif isinstance(error, commands.NoPrivateMessage):
        await ctx.author.send(f"🚫 O comando `!{cmd_name}` não pode ser usado em mensagens diretas (DM).")
    elif isinstance(error, commands.CommandInvokeError):
        original_error = error.original
        print(f"ERRO AO EXECUTAR COMANDO '{cmd_name}':")
        print(f"  Tipo do Erro Original: {type(original_error).__name__}")
        print(f"  Mensagem Original: {original_error}")
        import traceback
        print(f"  Traceback Completo:")
        traceback.print_exception(type(original_error), original_error, original_error.__traceback__)
        await ctx.send(f"🚫 Ocorreu um erro interno ao tentar executar o comando `!{cmd_name}`. O desenvolvedor foi notificado no console.")
    else:
        print(f"ERRO NÃO TRATADO (on_command_error) PARA COMANDO '{cmd_name}':")
        print(f"  Tipo do Erro: {type(error).__name__}")
        print(f"  Mensagem: {error}")
        import traceback
        print(f"  Traceback Completo:")
        traceback.print_exception(type(error), error, error.__traceback__)
        await ctx.send("🚫 Ocorreu um erro inesperado. O desenvolvedor foi notificado no console.")

# --- DESLIGAMENTO GRACIOSO ---
async def notify_shutdown_async():
    """Notifica canais configurados que o bot está desligando (exceto por !desligando)."""
    if bot_instance and channels_to_notify:
        # Evita notificação duplicada se o desligamento foi iniciado pelo comando !desligando
        # Esta é uma heurística; uma flag global seria mais robusta se necessário.
        # Por agora, se o bot estiver fechando sem ser pelo comando, envia esta mensagem.
        # (O comando !desligando já envia sua própria mensagem)
        
        # Para simplificar, vamos assumir que esta função só é chamada em desligamentos inesperados
        # ou quando o bot é fechado externamente, e não pelo comando !desligando.
        # O atexit.register pode ser chamado em vários cenários de término.
        
        print("Notificando canais sobre desligamento (atexit)...")
        for channel_id in channels_to_notify:
            channel = bot_instance.get_channel(channel_id)
            if channel:
                try:
                    await channel.send("🔴 Bot está sendo desligado (processo encerrado).")
                except Exception as e:
                    print(f"Erro ao notificar canal {channel_id} sobre desligamento (atexit): {e}")
            else:
                print(f"Canal {channel_id} não encontrado para notificação de desligamento (atexit).")

def shutdown_handler():
    """Handler para atexit, tenta notificar de forma síncrona se o loop já parou."""
    print("Executando shutdown_handler (atexit)...")
    if bot_instance and getattr(bot_instance, 'loop', None):
        loop = bot_instance.loop
        if loop.is_running():
            print("Loop está rodando, enviando notificação de desligamento de forma assíncrona.")
            # Cria uma future e agenda a corrotina no loop de eventos existente
            future = asyncio.run_coroutine_threadsafe(notify_shutdown_async(), loop)
            try:
                future.result(timeout=5) # Espera a conclusão por um curto período
                print("Notificação de desligamento (async) enviada.")
            except asyncio.TimeoutError:
                print("Timeout ao esperar notificação de desligamento (async).")
            except Exception as e:
                print(f"Erro ao executar notify_shutdown_async via atexit: {e}")
        else:
            print("Loop não está rodando. Não é possível enviar notificações de desligamento assíncronas.")
            # Aqui, você poderia tentar enviar uma mensagem de forma síncrona se tivesse uma maneira,
            # mas com discord.py, as operações de API são geralmente assíncronas.
    else:
        print("Instância do bot ou loop não disponível no shutdown_handler.")

atexit.register(shutdown_handler)


# --- INICIA BOT ---
if __name__ == "__main__":
    try:
        print("Iniciando o bot...")
        bot.run(TOKEN)
    except discord.PrivilegedIntentsRequired as e:
        print(f"ERRO CRÍTICO: {e}")
        print("Uma ou mais Intents Privilegiadas (Message Content, Server Members ou Presence) não estão habilitadas no Discord Developer Portal para este bot OU não foram especificadas no código.")
        print("Acesse https://discord.com/developers/applications, selecione seu bot, vá em 'Bot' e habilite as intents necessárias.")
        print("Verifique se 'intents.message_content = True', 'intents.guilds = True', 'intents.members = True' estão no código.")
    except discord.LoginFailure:
        print("ERRO CRÍTICO: Falha no login. Verifique se o DISCORD_TOKEN no arquivo .env (ou variável de ambiente) é válido e correto.")
    except Exception as e:
        print(f"Erro fatal ao tentar iniciar ou durante a execução do bot: {e}")
        import traceback
        traceback.print_exc()
    finally:
        print("Bot foi encerrado.")
        if conn:
            conn.close()
            print("Conexão com o banco de dados SQLite fechada.")

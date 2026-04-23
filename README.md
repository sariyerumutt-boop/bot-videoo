import os
import pickle
import time
import random
from moviepy.editor import VideoFileClip, AudioFileClip, TextClip, CompositeVideoClip
from moviepy.config import change_settings
from google_auth_oauthlib.flow import InstalledAppFlow
from google.auth.transport.requests import Request
from googleapiclient.discovery import build
from googleapiclient.http import MediaFileUpload

# --- 1. SİSTEM AYARLARI (GİTHUB/BULUT UYUMLU) ---
# Linux sunucularda ImageMagick yolu budur
IMAGEMAGICK_EXE = "/usr/bin/convert"
change_settings({"IMAGEMAGICK_BINARY": IMAGEMAGICK_EXE})

# Dosya yolları (Robot, dosyaları botla aynı klasörde arar)
CLIENT_SECRET = "client_secret.json"
VIDEO_IN = "video.mp4" 
AUDIO_IN = "muzik.mp3"
SOZLER_DOSYASI = "motivasyon_sozleri_1000.txt"
PROGRESS_FILE = "siradaki_soz.txt"

# --- 2. SÖZ SIRALAYICI ---
def siradaki_sozu_al():
    if not os.path.exists(PROGRESS_FILE):
        with open(PROGRESS_FILE, "w") as f: f.write("0")
    with open(PROGRESS_FILE, "r") as f:
        index = int(f.read().strip())
    
    if not os.path.exists(SOZLER_DOSYASI): return "PES ETME!"
    
    with open(SOZLER_DOSYASI, "r", encoding="utf-8") as f:
        lines = f.readlines()
    
    if index >= len(lines): index = 0 # 1000 söz bitince başa dön
        
    line = lines[index].strip()
    soz = line.split(". ", 1)[-1] if ". " in line else line
    
    with open(PROGRESS_FILE, "w") as f: f.write(str(index + 1))
    return soz.upper()

# --- 3. KURGU MOTORU ---
def kurgu_yap(soz):
    print(f"🎬 İşleniyor: {soz}")
    out_path = "final_video.mp4"
    
    # Dosya kontrolleri
    if not all(os.path.exists(f) for f in [VIDEO_IN, AUDIO_IN]):
        print("❌ Dosyalar eksik!")
        return None

    clip = VideoFileClip(VIDEO_IN)
    music = AudioFileClip(AUDIO_IN).subclip(0, clip.duration).volumex(0.3)
    
    txt = TextClip(soz, fontsize=70, color='white', font='Arial-Bold',
                   method='caption', size=(clip.w*0.8, None),
                   align='center').set_duration(clip.duration).set_position('center')
    
    final = CompositeVideoClip([clip, txt]).set_audio(music)
    
    try:
        final.write_videofile(out_path, codec="libx264", audio_codec="aac", fps=24, logger=None)
        clip.close()
        music.close()
        return out_path
    except Exception as e:
        print(f"⚠️ Kurgu hatası: {e}")
        return None

# --- 4. YOUTUBE YÜKLEME ---
def youtube_yukle(video_path, soz):
    creds = None
    if os.path.exists('token.json'):
        with open('token.json', 'rb') as token: creds = pickle.load(token)
            
    if not creds or not creds.valid:
        print("❌ Token hatası! GitHub'da manuel giriş yapılamaz.")
        return

    yt = build('youtube', 'v3', credentials=creds)
    body = {
        'snippet': {
            'title': f"{soz[:50]}... #motivasyon #shorts",
            'description': "Pes etmek senin lügatinde yok. #başarı #disiplin",
            'categoryId': '22'
        },
        'status': {'privacyStatus': 'public'}
    }
    media = MediaFileUpload(video_path, chunksize=-1, resumable=True)
    yt.videos().insert(part='snippet,status', body=body, media_body=media).execute()
    print("🚀 Video YouTube'a uçtu!")

# --- 5. ANA TETİKLEYİCİ (DÖNGÜSÜZ) ---
if __name__ == "__main__":
    soz = siradaki_sozu_al()
    if soz:
        video = kurgu_yap(soz)
        if video:
            youtube_yukle(video, soz)

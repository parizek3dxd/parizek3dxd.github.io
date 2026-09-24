import os
import io
import re
import tkinter as tk
from tkinter import ttk, messagebox
from mutagen.mp3 import MP3
from mutagen.id3 import ID3, APIC, TIT2, TPE1
from PIL import Image, ImageOps

# KOSTRA ADRESÁŘŮ
MUSIC_DIR = "./music"
SD_DIR = "./sd_card"
PREVIEWS_DIR = "./previews"
TARGET_SIZE = (80, 80)

os.makedirs(MUSIC_DIR, exist_ok=True)
os.makedirs(SD_DIR, exist_ok=True)
os.makedirs(PREVIEWS_DIR, exist_ok=True)

def parse_filename_fallback(filename):
    """Pokud chybí ID3 tagy, vytáhne interpreta a název z názvu souboru."""
    name_no_ext = os.path.splitext(filename)[0]
    if " - " in name_no_ext:
        parts = name_no_ext.split(" - ", 1)
        artist_raw, title = parts[0].strip(), parts[1].strip()
    else:
        artist_raw, title = "Unknown Artist", name_no_ext.strip()
    
    # Úprava autorů oddělených čárkou / feat / &
    artists = [a.strip() for a in re.split(r'[,&]|\bfeat\b|\bft\b', artist_raw, flags=re.IGNORECASE)]
    artist_formatted = ", ".join(artists)
    return artist_formatted, title

def convert_to_4gray_bytes(pil_img):
    """Převede obrázek na 4 úrovně šedi (2 bity na pixel = 1600 B pro 80x80 px)."""
    gray = pil_img.convert('L')
    gray_4 = gray.quantize(colors=4, method=Image.Quantize.MEDIANCUT, dither=Image.Dither.FLOYDSTEINBERG)
    
    pixels = list(gray_4.getdata())
    bin_bytes = bytearray()
    
    for i in range(0, len(pixels), 4):
        p0 = pixels[i] & 0x03
        p1 = pixels[i+1] & 0x03
        p2 = pixels[i+2] & 0x03
        p3 = pixels[i+3] & 0x03
        byte = (p0 << 6) | (p1 << 4) | (p2 << 2) | p3
        bin_bytes.append(byte)
        
    return bin_bytes, gray_4

def process_sync(log_callback, progress_callback):
    files = [f for f in os.listdir(MUSIC_DIR) if f.lower().endswith(".mp3")]
    total_files = len(files)
    
    if total_files == 0:
        log_callback("❌ Složka 'music' je prázdná! Vlož do ní MP3 soubory.")
        return

    log_callback(f"📂 Nalezeno {total_files} MP3 souborů. Začínám synchronizaci...\n")

    for idx, filename in enumerate(files):
        base_name = os.path.splitext(filename)[0]
        mp3_in = os.path.join(MUSIC_DIR, filename)
        
        # Cesty pro SD kartu (ČISTÉ BEZ PNG)
        mp3_out = os.path.join(SD_DIR, filename)
        bin_out = os.path.join(SD_DIR, f"{base_name}.bin")
        txt_out = os.path.join(SD_DIR, f"{base_name}.txt")
        
        # Cesta pro náhledy na PC (MIMO SD KARTU)
        png_out = os.path.join(PREVIEWS_DIR, f"{base_name}_preview.png")

        # KONTROLA SYNCU: Pokud už soubory na SD existují, přeskočíme
        if os.path.exists(mp3_out) and os.path.exists(bin_out) and os.path.exists(txt_out):
            log_callback(f"⏭️ Přeskočeno (již na SD): {filename}")
            progress_callback((idx + 1) / total_files * 100)
            continue

        try:
            # --- 1. ZÍSKÁNÍ METADAT ---
            audio = MP3(mp3_in, ID3=ID3)
            title, artist = None, None
            img_data = None

            if audio.tags:
                if 'TIT2' in audio.tags: title = str(audio.tags['TIT2'])
                if 'TPE1' in audio.tags: artist = str(audio.tags['TPE1'])
                for tag in audio.tags.values():
                    if isinstance(tag, APIC):
                        img_data = tag.data
                        break

            # Fallback z názvu souboru, pokud chybí ID3 tagy
            if not title or not artist:
                f_artist, f_title = parse_filename_fallback(filename)
                if not artist: artist = f_artist
                if not title: title = f_title

            # Formátování více autorů
            artists_list = [a.strip() for a in re.split(r'[,&]|\bfeat\b|\bft\b', artist, flags=re.IGNORECASE)]
            artist_clean = ", ".join(artists_list)

            # Úložka textových metadat pro Pico na SD
            with open(txt_out, "w", encoding="utf-8") as f:
                f.write(f"{title}\n{artist_clean}")

            # --- 2. ZPRACOVÁNÍ OBALU (80x80 px / 4 Grayscale) ---
            if img_data:
                img = Image.open(io.BytesIO(img_data))
            else:
                img = Image.new('RGB', TARGET_SIZE, color=(200, 200, 200))

            # Crop na čtverec
            w, h = img.size
            min_dim = min(w, h)
            img = img.crop(((w - min_dim)//2, (h - min_dim)//2, (w + min_dim)//2, (h + min_dim)//2))
            img = img.resize(TARGET_SIZE, Image.Resampling.LANCZOS)

            # Konverze na 4-bit binární šedotón pro ePaper
            raw_bytes, preview_img = convert_to_4gray_bytes(img)
            
            # .bin soubor uložíme na SD
            with open(bin_out, "wb") as f:
                f.write(raw_bytes)

            # PNG náhled uložíme DO SLOŽKY PREVIEWS NA PC
            preview_img.save(png_out)

            # --- 3. KOPÍROVÁNÍ MP3 NA SD ---
            with open(mp3_in, 'rb') as src, open(mp3_out, 'wb') as dst:
                dst.write(src.read())

            log_callback(f"✅ Zpracováno: {artist_clean} - {title}")

        except Exception as e:
            log_callback(f"⚠️ Chyba u {filename}: {e}")

        progress_callback((idx + 1) / total_files * 100)

    log_callback("\n🎉 SYNCHRONIZACE HOTOVA! Obsah složky 'sd_card' můžeš přetáhnout na SD.")

# --- GUI APLIKACE (Tkinter) ---
class WalkmanSyncApp:
    def __init__(self, root):
        self.root = root
        self.root.title("Walkman Pico Sync Tool (4-Grayscale 80x80)")
        self.root.geometry("520x420")
        self.root.resizable(False, False)

        # UI Prvky
        title_label = tk.Label(root, text="🎧 Walkman SD Synchronizátor", font=("Helvetica", 14, "bold"))
        title_label.pack(pady=10)

        info_label = tk.Label(root, text="Vlož MP3 do 'music/' a klikni na Sync. Náhledy budou v 'previews/'.", font=("Helvetica", 9))
        info_label.pack()

        self.btn_sync = tk.Button(root, text="🚀 Spustit synchronizaci", font=("Helvetica", 11, "bold"), bg="#4CAF50", fg="white", command=self.start_sync)
        self.btn_sync.pack(pady=10)

        self.progress = ttk.Progressbar(root, orient="horizontal", length=450, mode="determinate")
        self.progress.pack(pady=5)

        self.log_text = tk.Text(root, height=14, width=60, font=("Consolas", 8))
        self.log_text.pack(pady=10)

    def log(self, text):
        self.log_text.insert(tk.END, text + "\n")
        self.log_text.see(tk.END)
        self.root.update_idletasks()

    def set_progress(self, val):
        self.progress['value'] = val
        self.root.update_idletasks()

    def start_sync(self):
        self.btn_sync.config(state=tk.DISABLED)
        self.log_text.delete('1.0', tk.END)
        self.set_progress(0)
        
        process_sync(self.log, self.set_progress)
        
        self.btn_sync.config(state=tk.NORMAL)
        messagebox.showinfo("Hotovo", "Synchronizace byla dokončena! Složka 'sd_card' je připravena k přetažení.")

if __name__ == "__main__":
    root = tk.Tk()
    app = WalkmanSyncApp(root)
    root.mainloop()

import os
import io
from mutagen.mp3 import MP3
from mutagen.id3 import ID3, APIC
from PIL import Image, ImageOps

# NASTAVENÍ SLOŽEK A ROZMĚRU
MUSIC_DIR = "./hudba"
OUTPUT_BIN_DIR = "./prevedeno"
OUTPUT_PREVIEW_DIR = "./nahledy_png"
TARGET_SIZE = (96, 96)  # PŘEDĚLÁNO NA 96x96

# Vytvoření složek
os.makedirs(OUTPUT_BIN_DIR, exist_ok=True)
os.makedirs(OUTPUT_PREVIEW_DIR, exist_ok=True)

def process_mp3(file_path, filename):
    try:
        audio = MP3(file_path, ID3=ID3)
        img_data = None

        # 1. Najít obal v MP3
        for tag in audio.tags.values():
            if isinstance(tag, APIC):
                img_data = tag.data
                break

        if img_data:
            img = Image.open(io.BytesIO(img_data))
            
            # 2. Ořez na čtverec
            width, height = img.size
            if width > height:
                left = (width - height) / 2
                img = img.crop((left, 0, left + height, height))
            elif height > width:
                top = (height - width) / 2
                img = img.crop((0, top, width, top + width))

            # 3. Resize na 96x96 px
            img = img.resize(TARGET_SIZE, Image.Resampling.LANCZOS)

            # 4. Černobílý dithering
            img_bw = img.convert('1', dither=Image.Dither.FLOYDSTEINBERG)
            
            # 5. Export binárních dat pro ePaper (1152 B)
            img_inverted = ImageOps.invert(img_bw.convert('L')).convert('1')
            bin_data = img_inverted.tobytes()

            name_no_ext = os.path.splitext(filename)[0]

            # Uložení .bin
            bin_path = os.path.join(OUTPUT_BIN_DIR, f"{name_no_ext}.bin")
            with open(bin_path, "wb") as f:
                f.write(bin_data)

            # 6. Přímé uložení PNG náhledu do počítače
            preview_img = ImageOps.invert(img_inverted.convert('L'))
            preview_path = os.path.join(OUTPUT_PREVIEW_DIR, f"{name_no_ext}_96x96.png")
            preview_img.save(preview_path)
                
            print(f"[OK] Vytvořen 96x96 art + PNG náhled pro: {name_no_ext}")

        else:
            print(f"[--] Žádný obal v: {filename}")

    except Exception as e:
        print(f"[ERR] Chyba u {filename}: {e}")

# Spuštění
print("--- Začínám převod na 96x96 px ---")
if os.path.exists(MUSIC_DIR):
    for filename in os.listdir(MUSIC_DIR):
        if filename.lower().endswith(".mp3"):
            process_mp3(os.path.join(MUSIC_DIR, filename), filename)
    print("\n--- HOTOVO! Náhledy najdeš ve složce 'nahledy_png' ---")
else:
    print(f"[ERR] Složka '{MUSIC_DIR}' neexistuje! Vytvoř ji a vlož do ní MP3.")

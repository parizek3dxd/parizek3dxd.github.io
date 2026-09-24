import os
import io
from mutagen.mp3 import MP3
from mutagen.id3 import ID3, APIC
from PIL import Image, ImageOps, ImageEnhance, ImageFilter

MUSIC_DIR = "./hudba"
OUTPUT_BIN_DIR = "./prevedeno"
OUTPUT_PREVIEW_DIR = "./nahledy_png"
TARGET_SIZE = (96, 96)

os.makedirs(OUTPUT_BIN_DIR, exist_ok=True)
os.makedirs(OUTPUT_PREVIEW_DIR, exist_ok=True)

def process_mp3(file_path, filename):
    try:
        audio = MP3(file_path, ID3=ID3)
        img_data = None

        for tag in audio.tags.values():
            if isinstance(tag, APIC):
                img_data = tag.data
                break

        if img_data:
            img = Image.open(io.BytesIO(img_data)).convert('RGB')
            
            # 1. Ořez na čtverec z prostředka
            w, h = img.size
            min_dim = min(w, h)
            left = (w - min_dim) / 2
            top = (h - min_dim) / 2
            img = img.crop((left, top, left + min_dim, top + min_dim))

            # 2. Zmenšení na 96x96 s vysokou kvalitou
            img = img.resize(TARGET_SIZE, Image.Resampling.LANCZOS)

            # 3. Mírné zvýšení kontrastu a doostření pro ePaper
            img = ImageEnhance.Contrast(img).enhance(1.3)
            img = img.filter(ImageFilter.UnsharpMask(radius=1, percentage=120, threshold=3))

            # 4. Převedení na šedou a jemný dithering (Atkinson / Ordered style)
            img_gray = img.convert('L')
            
            # Použijeme nastavení, které neshlukuje tečky do mraveniště
            img_bw = img_gray.convert('1', dither=Image.Dither.FLOYDSTEINBERG)

            # 5. Uložení binárních dat pro Pico
            img_inverted = ImageOps.invert(img_bw.convert('L')).convert('1')
            bin_data = img_inverted.tobytes()

            name_no_ext = os.path.splitext(filename)[0]

            bin_path = os.path.join(OUTPUT_BIN_DIR, f"{name_no_ext}.bin")
            with open(bin_path, "wb") as f:
                f.write(bin_data)

            # 6. Uložení PNG pro rychlou kontrolu v PC
            preview_path = os.path.join(OUTPUT_PREVIEW_DIR, f"{name_no_ext}_preview.png")
            img_bw.save(preview_path)
                
            print(f"[OK] Zpracováno: {name_no_ext}")

        else:
            print(f"[--] Bez obalu: {filename}")

    except Exception as e:
        print(f"[ERR] Chyba: {e}")

print("--- Předělávám na finální podobu ---")
if os.path.exists(MUSIC_DIR):
    for filename in os.listdir(MUSIC_DIR):
        if filename.lower().endswith(".mp3"):
            process_mp3(os.path.join(MUSIC_DIR, filename), filename)

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
            # 1. Načtení obrázku
            img = Image.open(io.BytesIO(img_data)).convert('RGB')
            
            # 2. Ořez na čtverec ze středu
            w, h = img.size
            min_dim = min(w, h)
            left = (w - min_dim) / 2
            top = (h - min_dim) / 2
            img = img.crop((left, top, left + min_dim, top + min_dim))

            # 3. Zmenšení na 96x96 px
            img = img.resize(TARGET_SIZE, Image.Resampling.LANCZOS)

            # 4. Úprava kontrastu a doostření pro ePaper
            img = ImageEnhance.Contrast(img).enhance(1.4)
            img = img.filter(ImageFilter.UnsharpMask(radius=1, percentage=150, threshold=3))

            # 5. Převod na černobílou s ditheringem (kompatibilní se všemi verzemi Pillow)
            img_bw = img.convert('1')

            # 6. Uložení binárních dat pro Pico (1152 B)
            img_inverted = ImageOps.invert(img_bw.convert('L')).convert('1')
            bin_data = img_inverted.tobytes()

            name_no_ext = os.path.splitext(filename)[0]

            bin_path = os.path.join(OUTPUT_BIN_DIR, f"{name_no_ext}.bin")
            with open(bin_path, "wb") as f:
                f.write(bin_data)

            # 7. Uložení PNG náhledu pro kontrolu v PC
            preview_path = os.path.join(OUTPUT_PREVIEW_DIR, f"{name_no_ext}_preview.png")
            img_bw.save(preview_path)
                
            print(f"[OK] Zpracováno: {name_no_ext}")

        else:
            print(f"[--] Bez obalu: {filename}")

    except Exception as e:
        print(f"[ERR] Chyba u {filename}: {e}")

print("--- Spouštím převod obalů ---")
if os.path.exists(MUSIC_DIR):
    for filename in os.listdir(MUSIC_DIR):
        if filename.lower().endswith(".mp3"):
            process_mp3(os.path.join(MUSIC_DIR, filename), filename)
    print("\n--- HOTOVO! Náhledy najdeš ve složce 'nahledy_png' ---")
else:
    print(f"[ERR] Složka '{MUSIC_DIR}' neexistuje! Vytvoř ji a vlož do ní MP3.")

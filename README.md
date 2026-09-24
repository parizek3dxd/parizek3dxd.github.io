import os
import io
from mutagen.mp3 import MP3
from mutagen.id3 import ID3, APIC
from PIL import Image, ImageOps

# NASTAVENÍ SLOŽEK A ROZMĚRU
MUSIC_DIR = "./hudba"
OUTPUT_DIR = "./prevedeno"
TARGET_SIZE = (80, 80)

# Vytvoření výstupní složky
os.makedirs(OUTPUT_DIR, exist_ok=True)

def process_mp3(file_path, filename):
    try:
        audio = MP3(file_path, ID3=ID3)
        img_data = None

        # 1. Hledání cover artu v ID3 tazích
        for tag in audio.tags.values():
            if isinstance(tag, APIC):
                img_data = tag.data
                break

        if img_data:
            # 2. Načtení obrázku
            img = Image.open(io.BytesIO(img_data))
            
            # 3. Ořez na čtverec (Center Crop)
            width, height = img.size
            if width > height:
                left = (width - height) / 2
                img = img.crop((left, 0, left + height, height))
            elif height > width:
                top = (height - width) / 2
                img = img.crop((0, top, width, top + width))

            # 4. Zmenšení na 80x80 px
            img = img.resize(TARGET_SIZE, Image.Resampling.LANCZOS)

            # 5. Převedení na 1-bitový černobílý obrázek s ditheringem
            img = img.convert('1', dither=Image.Dither.FLOYDSTEINBERG)
            
            # 6. Inverze barev (1 = bílá, 0 = černá pro ePaper) a export bajtů
            img_inverted = ImageOps.invert(img.convert('L')).convert('1')
            bin_data = img_inverted.tobytes()

            # 7. Uložení .bin souboru (přesně 800 B pro 80x80 px)
            name_no_ext = os.path.splitext(filename)[0]
            out_path = os.path.join(OUTPUT_DIR, f"{name_no_ext}.bin")
            with open(out_path, "wb") as f:
                f.write(bin_data)
                
            print(f"[OK] Vytvořen art: {name_no_ext}.bin")

        else:
            print(f"[--] MP3 neobsahuje obal: {filename}")

    except Exception as e:
        print(f"[ERR] Chyba u {filename}: {e}")

# Spuštění
print("--- Začínám převod obalů ---")
for filename in os.listdir(MUSIC_DIR):
    if filename.lower().endswith(".mp3"):
        process_mp3(os.path.join(MUSIC_DIR, filename), filename)
print("\n--- HOTOVO! Všechny .bin soubory najdeš ve složce 'prevedeno' ---")

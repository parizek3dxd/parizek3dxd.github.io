import os
import io
from mutagen.mp3 import MP3
from mutagen.id3 import ID3, APIC
from PIL import Image, ImageOps, ImageEnhance, ImageFilter

# NASTAVENÍ SLOŽEK A ROZMĚRU
MUSIC_DIR = "./hudba"
OUTPUT_BIN_DIR = "./prevedeno"
OUTPUT_PREVIEW_DIR = "./nahledy_png"
TARGET_SIZE = (96, 96)

# PRAH ČERNOBÍLÉ (128 je střed; čím nižší číslo, tím méně černé; čím vyšší, tím více černé)
THRESHOLD = 135 

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
            img = Image.open(io.BytesIO(img_data))
            
            # 1. Ořez na čtverec
            width, height = img.size
            if width > height:
                left = (width - height) / 2
                img = img.crop((left, 0, left + height, height))
            elif height > width:
                top = (height - width) / 2
                img = img.crop((0, top, width, top + width))

            # 2. Úprava kontrastu a ostrosti PROŠTĚPENÍ TEXTU
            img = img.convert('L') # Převedení na odstíny šedi
            
            # Zvýšení kontrastu (2.0 = dvojnásobný kontrast)
            enhancer = ImageEnhance.Contrast(img)
            img = enhancer.enhance(1.8)
            
            # Doostření hran
            img = img.filter(ImageFilter.SHARPEN)

            # 3. Resize na 96x96 px
            img = img.resize(TARGET_SIZE, Image.Resampling.LANCZOS)

            # 4. OSTRÝ ČERNOBÍLÝ PřEVOD (Bez ditheringu!)
            # Pixely tmavší než THRESHOLD budou černé (0), ostatní bílé (255)
            img_bw = img.point(lambda p: 255 if p > THRESHOLD else 0, mode='1')

            # 5. Export binárních dat pro ePaper
            img_inverted = ImageOps.invert(img_bw.convert('L')).convert('1')
            bin_data = img_inverted.tobytes()

            name_no_ext = os.path.splitext(filename)[0]

            # Uložení .bin
            bin_path = os.path.join(OUTPUT_BIN_DIR, f"{name_no_ext}.bin")
            with open(bin_path, "wb") as f:
                f.write(bin_data)

            # 6. Uložení PNG náhledu pro kontrolu na PC
            preview_img = ImageOps.invert(img_inverted.convert('L'))
            preview_path = os.path.join(OUTPUT_PREVIEW_DIR, f"{name_no_ext}_clean.png")
            preview_img.save(preview_path)
                
            print(f"[OK] Čistý 96x96 art vytvořen pro: {name_no_ext}")

        else:
            print(f"[--] Žádný obal v: {filename}")

    except Exception as e:
        print(f"[ERR] Chyba u {filename}: {e}")

# Spuštění
print("--- Začínám ostrý převod bez ditheringu ---")
if os.path.exists(MUSIC_DIR):
    for filename in os.listdir(MUSIC_DIR):
        if filename.lower().endswith(".mp3"):
            process_mp3(os.path.join(MUSIC_DIR, filename), filename)
    print("\n--- HOTOVO! Mrkni do 'nahledy_png' na výsledek ---")
else:
    print(f"[ERR] Složka '{MUSIC_DIR}' neexistuje!")

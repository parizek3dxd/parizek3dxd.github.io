import os
from PIL import Image, ImageOps

INPUT_DIR = "./prevedeno"  # Složka s .bin soubory
OUTPUT_DIR = "./nahledy_png"  # Sem se uloží klasické PNGčka
SIZE = (80, 80)

os.makedirs(OUTPUT_DIR, exist_ok=True)

print("--- Načítám .bin soubory a převádím na PNG ---")

for filename in os.listdir(INPUT_DIR):
    if filename.endswith(".bin"):
        bin_path = os.path.join(INPUT_DIR, filename)
        
        with open(bin_path, "rb") as f:
            raw_bytes = f.read()
            
        # Převedení bajtů zpátky na 1-bitový obrázek
        img = Image.frombytes('1', SIZE, raw_bytes)
        
        # Vrácení původní inverze barev pro náhled na monitoru
        img_preview = ImageOps.invert(img.convert('L'))
        
        # Uložení jako klasický PNG obrázek
        png_name = os.path.splitext(filename)[0] + "_preview.png"
        img_preview.save(os.path.join(OUTPUT_DIR, png_name))
        
        print(f"[OK] Převedeno: {png_name}")

print("\n--- Hotovo! Otevři složku 'nahledy_png' a podívej se, jak budou obaly vypadat na ePaperu. ---")

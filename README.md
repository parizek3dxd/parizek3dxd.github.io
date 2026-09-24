import os
from PIL import Image, ImageOps

INPUT_DIR = "./prevedeno"
OUTPUT_DIR = "./nahledy_png"
SIZE = (96, 96)
EXPECTED_BYTES = (96 * 96) // 8  # 1152 bajtů

os.makedirs(OUTPUT_DIR, exist_ok=True)

if not os.path.exists(INPUT_DIR):
    print(f"[ERR] Složka '{INPUT_DIR}' neexistuje!")
    exit()

bin_files = [f for f in os.listdir(INPUT_DIR) if f.endswith(".bin")]

if not bin_files:
    print(f"[ERR] Ve složce '{INPUT_DIR}' nejsou žádné .bin soubory.")
    exit()

print(f"--- Převádím {len(bin_files)} .bin souborů na PNG náhledy (96x96 px) ---")

for filename in bin_files:
    bin_path = os.path.join(INPUT_DIR, filename)
    
    with open(bin_path, "rb") as f:
        raw_bytes = f.read()
        
    if len(raw_bytes) != EXPECTED_BYTES:
        print(f"[!] {filename} má špatnou velikost ({len(raw_bytes)} B namísto {EXPECTED_BYTES} B). Přeskakuji.")
        continue

    # Převedení 1152 bajtů na 1-bitový obrázek
    img = Image.frombytes('1', SIZE, raw_bytes)
    
    # Otočení inverze barev pro náhled na monitoru
    img_preview = ImageOps.invert(img.convert('L'))
    
    png_name = os.path.splitext(filename)[0] + "_preview.png"
    out_path = os.path.join(OUTPUT_DIR, png_name)
    img_preview.save(out_path)
    
    print(f"[OK] Náhled vytvořen: {png_name}")

print("\n--- HOTOVO! Otevři složku 'nahledy_png' ---")

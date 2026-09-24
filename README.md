import os
from PIL import Image, ImageOps

# Nastavení složek
INPUT_DIR = "./prevedeno"
OUTPUT_DIR = "./nahledy_png"
SIZE = (80, 80)

# Kontrola, zda složka s .bin vůbec existuje
if not os.path.exists(INPUT_DIR):
    print(f"[ERR] Složka '{INPUT_DIR}' neexistuje!")
    print("-> Nejdříve spusť 'convert_art.py', aby se vytvořily .bin soubory.")
    exit()

bin_files = [f for f in os.listdir(INPUT_DIR) if f.endswith(".bin")]

if not bin_files:
    print(f"[ERR] Ve složce '{INPUT_DIR}' nejsou žádné .bin soubory.")
    print("-> Nakopíruj do složky 'hudba' nějaké MP3 s obalem a spusť 'convert_art.py'.")
    exit()

os.makedirs(OUTPUT_DIR, exist_ok=True)
print(f"--- Nalezeno {len(bin_files)} .bin souborů. Převádím na PNG náhledy ---")

for filename in bin_files:
    bin_path = os.path.join(INPUT_DIR, filename)
    
    with open(bin_path, "rb") as f:
        raw_bytes = f.read()
        
    # Kontrola správné velikosti (80x80 px / 8 = 800 bajtů)
    if len(raw_bytes) != 800:
        print(f"[!] Soubor {filename} má špatnou velikost ({len(raw_bytes)} B namísto 800 B). Přeskakuji.")
        continue

    # Převedení bajtů na obrázek
    img = Image.frombytes('1', SIZE, raw_bytes)
    
    # Invertování barev pro zobrazení na monitoru (černé pixely na bílém podkladu)
    img_preview = ImageOps.invert(img.convert('L'))
    
    # Uložení jako PNG
    png_name = os.path.splitext(filename)[0] + "_preview.png"
    out_path = os.path.join(OUTPUT_DIR, png_name)
    img_preview.save(out_path)
    
    print(f"[OK] Vytvořen náhled: {out_path}")

print("\n--- HOTOVO! Otevři složku 'nahledy_png' a prohlédni si výsledky. ---")

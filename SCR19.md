import os
import random

print("Paste your config links below, then press Ctrl+D (EOF):")
lines = []
try:
    while True:
        line = input()
        if line.strip():
            lines.append(line.strip())
except EOFError:
    pass

vless_links = [l for l in lines if l.startswith("vless://")]
trojan_links = [l for l in lines if l.startswith("trojan://")]
ss_links = [l for l in lines if l.startswith("ss://") or l.startswith("ssss://")]

if not (vless_links or trojan_links or ss_links):
    print("❌ No valid links found.")
    exit()

output_dir = "/sdcard/Download/mix98"
os.makedirs(output_dir, exist_ok=True)

# فیدبک داخلی برای نگه داشتن ترکیب 2 تایی ساخته شده توسط گزینه 2
internal_feedback_2combo = []

def generate_3combo():
    combos = []
    all_links = vless_links + trojan_links + ss_links
    max_count = min(50, len(all_links))
    for _ in range(max_count):
        a = random.choice(all_links)
        b = random.choice(all_links)
        c = random.choice(all_links)
        # حذف پروتکل سوم و اضافه کردن ss//
        if "://" in c:
            c = c.split("://", 1)[1]
        combo = f"{a}+{b}+ss//{c}"
        combos.append(combo)
    return combos

def generate_2combo_from_3combo_file(file_path):
    # خواندن فایل خروجی 3 تایی و حذف قسمت سوم ss// (مخفی انجام میشه)
    combos_2 = []
    try:
        with open(file_path, "r") as f:
            lines = f.readlines()
            for line in lines:
                line = line.strip()
                if line:
                    parts = line.split('+')
                    if len(parts) >= 2:
                        # فقط دو قسمت اول نگه داشته شود
                        combo2 = f"{parts[0]}+{parts[1]}"
                        combos_2.append(combo2)
    except Exception as e:
        print(f"❌ Error reading file for 2-combo generation: {e}")
        return []

    return combos_2

def save_2combo_to_file(combos_2, filename):
    output_path = os.path.join(output_dir, filename)
    with open(output_path, "w") as f:
        f.write("\n".join(combos_2))
    print(f"✅ {len(combos_2)} two-combo saved to: {output_path}")

while True:
    print("\nChoose combination mode:")
    print("1. Generate new 3-combo")
    print("2. Generate hidden 2-combo from previous 3-combo file (internal feedback)")
    print("3. Save 2-combo from internal feedback to file")

    choice = input("> ").strip()

    if choice == "1":
        combos_3 = generate_3combo()
        print(f"\nGenerated {len(combos_3)} three-combo combinations:\n")
        for combo in combos_3:
            print(combo)

        # فایل خروجی نام بپرس
        output_name = input("\nEnter output filename for 3-combo (e.g., result3.txt): ").strip()
        if not output_name.endswith(".txt"):
            output_name += ".txt"

        output_path = os.path.join(output_dir, output_name)
        with open(output_path, "w") as f:
            f.write("\n".join(combos_3))
        print(f"✅ 3-combo saved to: {output_path}")

        # برای گزینه 2 ذخیره مسیر فایل 3 تایی جهت استفاده در فیدبک داخلی
        last_3combo_file = output_path

    elif choice == "2":
        # اگر قبلاً فایل 3 ترکیب وجود ندارد، خطا بده
        if 'last_3combo_file' not in locals():
            print("❌ No previous 3-combo file found. Please generate 3-combo first (option 1).")
            continue
        # مخفی بودن: هیچ چیزی چاپ نکن، فقط فیدبک داخلی بساز
        internal_feedback_2combo = generate_2combo_from_3combo_file(last_3combo_file)
        # هیچ چاپی نکن، عملیات مخفی است

    elif choice == "3":
        if not internal_feedback_2combo:
            print("❌ No internal 2-combo feedback found. Generate it first using option 2.")
            continue
        output_name = input("Enter output filename to save 2-combo (e.g., result2.txt): ").strip()
        if not output_name.endswith(".txt"):
            output_name += ".txt"
        save_2combo_to_file(internal_feedback_2combo, output_name)

    else:
        print("❌ Invalid choice. Try again.")
        continue

    again = input("\nDo you want to perform another operation? (y/n): ").strip().lower()
    if again != 'y':
        break

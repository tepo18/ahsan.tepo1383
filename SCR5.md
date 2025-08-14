import os
import random

# مسیر ورودی و خروجی
input_file = "input.txt"
output_dir = "/sdcard/Download/mix98"

# خواندن لینک‌ها از فایل
with open(input_file, "r") as f:
    lines = f.read().splitlines()

# جداسازی لینک‌ها بر اساس نوع
vless_links = [line for line in lines if line.startswith("vless://")]
trojan_links = [line for line in lines if line.startswith("trojan://")]
ss_links = [line for line in lines if line.startswith("ss://") or line.startswith("ssss://")]

# بررسی وجود حداقل یک لینک
if not (vless_links or trojan_links or ss_links):
    print("❌ هیچ لینکی شناسایی نشد.")
    exit()

# ترکیب لینک‌ها به صورت دسته‌های سه‌تایی
def generate_combinations():
    combinations = []
    all_links = vless_links + trojan_links + ss_links
    max_count = min(50, len(all_links))  # حداکثر 50 ترکیب

    for _ in range(max_count):
        a = random.choice(all_links)
        b = random.choice(all_links)
        c = random.choice(all_links)
        combo = f"{a}+{b}+ss/{c}"
        combinations.append(combo)

    return combinations

# دریافت نام فایل خروجی
output_name = input("📤 Enter output filename (e.g., output.txt): ").strip()
if not output_name.endswith(".txt"):
    output_name += ".txt"

# ساخت پوشه خروجی اگر وجود نداشت
os.makedirs(output_dir, exist_ok=True)

# تولید ترکیب‌ها و ذخیره در فایل
combos = generate_combinations()
output_path = os.path.join(output_dir, output_name)
with open(output_path, "w") as f:
    f.write("\n".join(combos))

print(f"✅ {len(combos)} combinations generated and saved to: {output_path}")

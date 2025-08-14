import os
import random

# Output directory path
output_dir = "/sdcard/Download/mix98"

# Get links from user input
print("📥 Paste your links one by one and press Enter. When you're done, press Enter on an empty line or Ctrl+D:")
lines = []
try:
    while True:
        line = input()
        if line.strip() == "":
            break
        lines.append(line.strip())
except EOFError:
    pass

# Separate links by type
vless_links = [line for line in lines if line.startswith("vless://")]
trojan_links = [line for line in lines if line.startswith("trojan://")]
ss_links = [line for line in lines if line.startswith("ss://") or line.startswith("ssss://")]

# Check if there is at least one link
if not (vless_links or trojan_links or ss_links):
    print("❌ No valid links detected.")
    exit()

# Combine links in triplets
def generate_combinations():
    combinations = []
    all_links = vless_links + trojan_links + ss_links
    max_count = min(50, len(all_links))  # Max 50 combos

    for _ in range(max_count):
        a = random.choice(all_links)
        b = random.choice(all_links)
        c = random.choice(all_links)

        # Remove protocol prefix from third link
        for prefix in ["vless://", "trojan://", "ss://", "ssss://"]:
            if c.startswith(prefix):
                c = c[len(prefix):]
                break

        combo = f"{a}+{b}+ss//{c}"
        combinations.append(combo)

    return combinations

# Get output file name
output_name = input("📤 Enter output file name (e.g. output.txt): ").strip()
if not output_name.endswith(".txt"):
    output_name += ".txt"

# Create output directory if it doesn't exist
os.makedirs(output_dir, exist_ok=True)

# Generate and save combinations
combos = generate_combinations()
output_path = os.path.join(output_dir, output_name)
with open(output_path, "w") as f:
    f.write("\n".join(combos))

print(f"✅ {len(combos)} combinations saved to: {output_path}")

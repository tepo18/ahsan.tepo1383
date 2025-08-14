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

# internal feedback list for 2-combo generated internally
internal_feedback_2combo = []

# Keep track of used configs indexes to remove later
used_configs_indexes = set()

def generate_3combo():
    combos = []
    all_links = vless_links + trojan_links + ss_links
    max_count = min(50, len(all_links))
    for _ in range(max_count):
        a_index = random.randint(0, len(all_links) - 1)
        b_index = random.randint(0, len(all_links) - 1)
        c_index = random.randint(0, len(all_links) - 1)

        a = all_links[a_index]
        b = all_links[b_index]
        c = all_links[c_index]

        # Keep track of used config indexes for removal later
        used_configs_indexes.update([a_index, b_index, c_index])

        # Remove protocol from c and add ss//
        if "://" in c:
            c = c.split("://", 1)[1]
        combo = f"{a}+{b}+ss//{c}"
        combos.append(combo)
    return combos

def generate_2combo_from_3combo_file(file_path):
    combos_2 = []
    try:
        with open(file_path, "r") as f:
            lines = f.readlines()
            for line in lines:
                line = line.strip()
                if line:
                    parts = line.split('+')
                    if len(parts) >= 2:
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

def remove_used_configs():
    global vless_links, trojan_links, ss_links
    all_links = vless_links + trojan_links + ss_links
    # Sort indexes descending to pop correctly
    for idx in sorted(used_configs_indexes, reverse=True):
        if idx < len(all_links):
            all_links.pop(idx)
    # Update original lists accordingly
    vless_links = [l for l in all_links if l.startswith("vless://")]
    trojan_links = [l for l in all_links if l.startswith("trojan://")]
    ss_links = [l for l in all_links if l.startswith("ss://") or l.startswith("ssss://")]
    used_configs_indexes.clear()

last_3combo_file = None

while True:
    print("\nChoose combination mode:")
    print("1. Generate new 3-combo")
    print("2. Generate hidden 2-combo from previous 3-combo file (internal feedback)")
    print("3. Save 2-combo from internal feedback to file")
    print("4. End current cycle and go to removal/continue prompt")

    choice = input("> ").strip()

    if choice == "1":
        combos_3 = generate_3combo()
        print(f"\nGenerated {len(combos_3)} three-combo combinations.\n")

        output_name = input("Enter output filename for 3-combo (e.g., result3.txt): ").strip()
        if not output_name.endswith(".txt"):
            output_name += ".txt"
        output_path = os.path.join(output_dir, output_name)
        with open(output_path, "w") as f:
            f.write("\n".join(combos_3))
        print(f"✅ 3-combo saved to: {output_path}")

        last_3combo_file = output_path

    elif choice == "2":
        if not last_3combo_file:
            print("❌ No previous 3-combo file found. Generate 3-combo first (option 1).")
            continue
        internal_feedback_2combo = generate_2combo_from_3combo_file(last_3combo_file)

    elif choice == "3":
        if not internal_feedback_2combo:
            print("❌ No internal 2-combo feedback found. Generate it first using option 2.")
            continue
        output_name = input("Enter output filename to save 2-combo (e.g., result2.txt): ").strip()
        if not output_name.endswith(".txt"):
            output_name += ".txt"
        save_2combo_to_file(internal_feedback_2combo, output_name)

    elif choice == "4":
        if not used_configs_indexes:
            print("No used configs to remove.")
        else:
            remove_choice = input("Do you want to remove used configs from input pool? (y/n): ").strip().lower()
            if remove_choice == 'y':
                remove_used_configs()
                print("✅ Used configs removed from input pool.")
            else:
                print("Used configs kept in input pool.")

        cont_choice = input("Do you want to continue with new operations? (y/n): ").strip().lower()
        if cont_choice != 'y':
            print("Goodbye!")
            break
        else:
            # reset internal feedback and used configs tracking for new cycle
            internal_feedback_2combo = []
            used_configs_indexes.clear()
            last_3combo_file = None
    else:
        print("❌ Invalid choice. Try again.")

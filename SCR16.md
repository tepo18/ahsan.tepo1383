#!/usr/bin/env python3
import os

GREEN = "\033[92m"
YELLOW = "\033[93m"
BLUE = "\033[94m"
RESET = "\033[0m"

def parse_link(link):
    # اینجا فقط vless رو قبول میکنیم برای نمونه
    return link.startswith("vless://")

def get_manual_input():
    print(BLUE + "Enter your configs manually. Press Ctrl+D (EOF) when done:" + RESET)
    lines = []
    try:
        while True:
            line = input()
            if line.strip():
                lines.append(line.strip())
    except EOFError:
        pass
    return lines

def get_subscription_input():
    filepath = input(BLUE + "Enter path to subscription file:" + RESET).strip()
    if not os.path.isfile(filepath):
        print(YELLOW + "File not found!" + RESET)
        return []
    with open(filepath, "r", encoding="utf-8") as f:
        lines = [line.strip() for line in f if line.strip()]
    print(GREEN + f"Read {len(lines)} lines from subscription." + RESET)
    return lines

def main():
    print(GREEN + "Select input method:" + RESET)
    print("1. Manual input")
    print("2. Load from subscription file")

    choice = input("Enter choice (1 or 2): ").strip()
    if choice == "1":
        lines = get_manual_input()
    elif choice == "2":
        lines = get_subscription_input()
    else:
        print(YELLOW + "Invalid choice, exiting." + RESET)
        return

    valid_links = [l for l in lines if parse_link(l)]

    if not valid_links:
        print(YELLOW + "❌ No valid links found." + RESET)
        return

    print(GREEN + f"✅ Found {len(valid_links)} valid vless links." + RESET)

    print("\nSelect output type:")
    print("1. Fragmented Config (V2Ray)")
    print("2. Raw JSON")

    out_choice = input("Enter your choice (1 or 2): ").strip()
    if out_choice not in ("1", "2"):
        print(YELLOW + "Invalid choice, exiting." + RESET)
        return

    filename = input("Enter output file name (without extension): ").strip()
    if not filename:
        filename = "output"

    outdir = "/sdcard/Download/almasi98"
    os.makedirs(outdir, exist_ok=True)
    filepath = os.path.join(outdir, f"{filename}.json")

    with open(filepath, "w", encoding="utf-8") as f:
        if out_choice == "1":
            f.write("// Fragmented Config placeholder\n")
        f.write("\n".join(valid_links))

    print(GREEN + f"✅ Saved to: {filepath}" + RESET)

if __name__ == "__main__":
    main()

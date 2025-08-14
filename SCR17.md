import os
import random
import re
import subprocess
import requests
from rich import print

OUTPUT_DIR = "/storage/emulated/0/Download/almasi98"
os.makedirs(OUTPUT_DIR, exist_ok=True)

protocols = ["vmess://", "vless://", "trojan://", "ss://", "ssss://"]

def extract_host(config):
    match = re.search(r"@([^:/?#]+)", config)
    return match.group(1) if match else None

def ping_host(host):
    try:
        output = subprocess.check_output(["ping", "-c", "3", "-W", "1", host], stderr=subprocess.DEVNULL).decode()
        match = re.search(r"rtt min/avg/max/mdev = [\d.]+/([\d.]+)", output)
        return float(match.group(1)) if match else None
    except:
        return None

def classify_ping(ping):
    if ping is None:
        return "red", "[bold red][BAD][/bold red]"
    elif ping < 150:
        return "green", "[bold green][GOOD][/bold green]"
    elif ping < 300:
        return "yellow", "[bold yellow][WARN][/bold yellow]"
    else:
        return "red", "[bold red][BAD][/bold red]"

def fetch_sub_link(url):
    try:
        r = requests.get(url, timeout=15)
        r.raise_for_status()
        content = r.text
        content = ''.join([c for c in content if ord(c) < 128])
        return content
    except Exception as e:
        print(f"[red]❌ Failed to fetch: {url}\nError: {e}[/red]")
        return None

def parse_configs(raw_data):
    lines = raw_data.splitlines()
    configs = [line.strip() for line in lines if line.strip()]
    return configs

def categorize_configs(configs):
    vless = []
    trojan = []
    ss = []
    others = []

    for cfg in configs:
        lower = cfg.lower()
        if lower.startswith("vless://"):
            vless.append(cfg)
        elif lower.startswith("trojan://"):
            trojan.append(cfg)
        elif lower.startswith("ss://") or lower.startswith("ssss://"):
            ss.append(cfg)
        else:
            others.append(cfg)

    return vless, trojan, ss, others

def save_to_file(filepath, data):
    with open(filepath, "w", encoding="utf-8") as f:
        for d in data:
            f.write(d + "\n")

def generate_combinations_from_list(config_list, count=50):
    combos = []
    n = len(config_list)
    if n < 3:
        print("[red]Not enough configs to generate combos (need at least 3).[/red]")
        return combos

    attempts = 0
    max_attempts = count * 10

    while len(combos) < count and attempts < max_attempts:
        a, b, c = random.sample(config_list, 3)

        # Remove protocol from third link to keep consistent combo format:
        c_no_proto = c
        for prefix in protocols:
            if c_no_proto.startswith(prefix):
                c_no_proto = c_no_proto[len(prefix):]
                break

        combo = f"{a}+{b}+ss//{c_no_proto}"
        if combo not in combos:
            combos.append(combo)
        attempts += 1

    return combos

def script1():
    print("[cyan]Script 1: Generate combos from input file[/cyan]")
    filename = input("Enter input filename (txt file with configs line by line): ").strip()
    if not filename:
        print("[red]Filename cannot be empty.[/red]")
        return

    if not os.path.isfile(filename):
        print(f"[red]File '{filename}' does not exist.[/red]")
        return

    with open(filename, "r", encoding="utf-8") as f:
        lines = [line.strip() for line in f if line.strip()]

    if not lines:
        print("[red]No configs found in the file.[/red]")
        return

    print("[cyan]Pinging hosts to filter best configs (green)...[/cyan]")
    green_configs = []
    for config in lines:
        host = extract_host(config)
        if not host:
            continue
        ping = ping_host(host)
        status, _ = classify_ping(ping)
        if status == "green":
            green_configs.append(config)

    if not green_configs:
        print("[red]No green ping configs found. Using all configs instead.[/red]")
        green_configs = lines

    combos = generate_combinations_from_list(green_configs, count=50)

    if not combos:
        print("[red]Failed to generate combos.[/red]")
        return

    out_filename = input("Enter output filename (default: combos_output.txt): ").strip()
    if not out_filename:
        out_filename = "combos_output.txt"
    if not out_filename.endswith(".txt"):
        out_filename += ".txt"

    out_path = os.path.join(OUTPUT_DIR, out_filename)
    save_to_file(out_path, combos)
    print(f"[green]✅ {len(combos)} combos generated and saved to: {out_path}[/green]")

def script2():
    print("[cyan]Script 2: Fetch, ping test, categorize configs[/cyan]")
    all_configs = []

    print("Choose input method:")
    print("1) Manual input (Ctrl+D to finish)")
    print("2) Fetch from subscription URL")

    choice = input("Enter choice (1 or 2): ").strip()

    if choice == "1":
        print("[cyan]Paste your configs line by line. Press Ctrl+D (or Ctrl+Z on Windows) to finish:[/cyan]")
        try:
            while True:
                line = input()
                if line.strip():
                    all_configs.append(line.strip())
        except EOFError:
            pass
    elif choice == "2":
        while True:
            url = input("Enter subscription URL: ").strip()
            data = fetch_sub_link(url)
            if data:
                new_cfgs = parse_configs(data)
                all_configs.extend(new_cfgs)
                print(f"[green]✅ Fetched {len(new_cfgs)} configs[/green]")
            else:
                print("[red]No configs fetched.[/red]")
            more = input("Add another link? (y/n): ").strip().lower()
            if more != "y":
                break
    else:
        print("[red]Invalid choice.[/red]")
        return

    all_configs = list(set(all_configs))
    if not all_configs:
        print("[red]No configs found.[/red]")
        return

    vless, trojan, ss, others = categorize_configs(all_configs)

    print(f"\nTotal configs: {len(all_configs)}")
    print(f"VLESS: {len(vless)}")
    print(f"TROJAN: {len(trojan)}")
    print(f"SHADOWSOCKS: {len(ss)}")
    print(f"Others: {len(others)}")

    green, yellow, red = [], [], []
    print("\n[cyan]Performing ping checks...[/cyan]")
    for config in all_configs:
        host = extract_host(config)
        if not host:
            print(f"[red][INVALID] {config}[/red]")
            continue
        ping = ping_host(host)
        status, label = classify_ping(ping)
        print(f"{label} {host} - {ping if ping else 'NO REPLY'}ms")
        if status == "green":
            green.append(config)
        elif status == "yellow":
            yellow.append(config)
        else:
            red.append(config)

    print("\nPing Summary:")
    print(f"[green]GREEN:[/green] {len(green)}")
    print(f"[yellow]YELLOW:[/yellow] {len(yellow)}")
    print(f"[red]RED:[/red] {len(red)}")

    while True:
        print("\nChoose output to save:")
        print("1) VLESS only")
        print("2) TROJAN only")
        print("3) SHADOWSOCKS only")
        print("4) All configs")
        print("5) Combo from green VLESS + TROJAN + SHADOWSOCKS")
        print("0) Exit")

        out_choice = input("Your choice: ").strip()

        if out_choice == "1":
            path = os.path.join(OUTPUT_DIR, "vless.txt")
            save_to_file(path, vless)
            print(f"[green]Saved VLESS configs to {path}[/green]")
        elif out_choice == "2":
            path = os.path.join(OUTPUT_DIR, "trojan.txt")
            save_to_file(path, trojan)
            print(f"[green]Saved TROJAN configs to {path}[/green]")
        elif out_choice == "3":
            path = os.path.join(OUTPUT_DIR, "shadowsocks.txt")
            save_to_file(path, ss)
            print(f"[green]Saved SHADOWSOCKS configs to {path}[/green]")
        elif out_choice == "4":
            path = os.path.join(OUTPUT_DIR, "all_configs.txt")
            save_to_file(path, all_configs)
            print(f"[green]Saved ALL configs to {path}[/green]")
        elif out_choice == "5":
            green_combined = [cfg for cfg in green if any(cfg.lower().startswith(p) for p in ["vless://", "trojan://", "ss://", "ssss://"])]
            if len(green_combined) < 3:
                print("[red]Not enough green configs for combos.[/red]")
                continue
            combos = generate_combinations_from_list(green_combined, 50)
            path = os.path.join(OUTPUT_DIR, "green_combos.txt")
            save_to_file(path, combos)
            print(f"[green]Saved 50 combos from green configs to {path}[/green]")
        elif out_choice == "0":
            print("[cyan]Exiting output menu.[/cyan]")
            break
        else:
            print("[red]Invalid choice.[/red]")

def main():
    while True:
        print("\nChoose script to run:")
        print("1) Script 1 (Generate combos from input file)")
        print("2) Script 2 (Fetch, ping test, categorize configs)")
        print("0) Exit")
        choice = input("Your choice: ").strip()
        if choice == "1":
            script1()
        elif choice == "2":
            script2()
        elif choice == "0":
            print("[cyan]Goodbye![/cyan]")
            break
        else:
            print("[red]Invalid choice. Try again.[/red]")

if __name__ == "__main__":
    main()

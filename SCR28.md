import os
import re
import subprocess
import requests
import json
import yaml
from urllib.parse import quote
from rich import print

protocols = ["vmess://", "vless://", "trojan://", "ss://"]

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
        if "vless://" in lower:
            vless.append(cfg)
        elif "trojan://" in lower:
            trojan.append(cfg)
        elif "ss://" in lower:
            ss.append(cfg)
        else:
            others.append(cfg)

    return vless, trojan, ss, others

def save_to_file(filepath, data):
    with open(filepath, "w", encoding="utf-8") as f:
        for d in data:
            f.write(d + "\n")

def save_json(filepath, data):
    with open(filepath, "w", encoding="utf-8") as f:
        json.dump(data, f, ensure_ascii=False, indent=2)

def save_yaml(filepath, data):
    with open(filepath, "w", encoding="utf-8") as f:
        yaml.dump(data, f, allow_unicode=True)

def main():
    all_configs = []

    print("[cyan]Choose input method:[/cyan]")
    print("1) Manual input (Ctrl+D to finish)")
    print("2) Fetch from subscription URL")
    choice = input("Enter choice (1 or 2): ").strip()

    if choice == "1":
        print("[cyan]Paste your configs line by line. Press Ctrl+D to finish:[/cyan]")
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
                print("No configs fetched.")
            more = input("Add another link? (y/n): ").strip().lower()
            if more != "y":
                break
    else:
        print("Invalid choice.")
        return

    all_configs = list(set(all_configs))
    vless, trojan, ss, others = categorize_configs(all_configs)

    print(f"\n[bold]Total configs:[/bold] {len(all_configs)}")
    print(f"VLESS: {len(vless)}")
    print(f"TROJAN: {len(trojan)}")
    print(f"SHADOWSOCKS: {len(ss)}")
    print(f"Others: {len(others)}")

    green, yellow, red = [], [], []
    ping_results = {}

    print("\n[bold cyan]Performing ping checks...[/bold cyan]")
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
        ping_results[config] = (status, ping)

    print("\n[bold]Ping Summary:[/bold]")
    print(f"[green]GREEN:[/green] {len(green)}")
    print(f"[yellow]YELLOW:[/yellow] {len(yellow)}")
    print(f"[red]RED:[/red] {len(red)}")

    while True:
        print("\nChoose output:")
        print("1) VLESS only")
        print("2) TROJAN only")
        print("3) SHADOWSOCKS only")
        print("4) All configs")
        print("5) Combo (VLESS + TROJAN)")
        print("6) Combo (VLESS + TROJAN + SS)")
        print("7) Green ping only")
        print("8) Green + Yellow ping")
        print("9) Save all configs with custom filename")
        print("10) Save all as JSON")
        print("11) Convert to YAML for Clash Meta")
        print("12) Fragmented outputs (each config in own file)")
        print("13) Combo VLESS + SS")
        print("14) Combo TROJAN + SS")
        print("0) Exit")

        option = input("Enter choice number [0-14]: ").strip()
        folder = "/storage/emulated/0/Download/Akbar98"
        os.makedirs(folder, exist_ok=True)

        if option == "0":
            print("Exiting...")
            break

        elif option == "1":
            name = input("File name: ").strip() or "output_vless"
            save_to_file(os.path.join(folder, name + ".txt"), vless)

        elif option == "2":
            name = input("File name: ").strip() or "output_trojan"
            save_to_file(os.path.join(folder, name + ".txt"), trojan)

        elif option == "3":
            name = input("File name: ").strip() or "output_ss"
            save_to_file(os.path.join(folder, name + ".txt"), ss)

        elif option == "4":
            name = input("File name: ").strip() or "output_all"
            save_to_file(os.path.join(folder, name + ".txt"), all_configs)

        elif option == "5":
            combo = vless + trojan
            name = input("File name: ").strip() or "combo_vless_trojan"
            save_to_file(os.path.join(folder, name + ".txt"), combo)

        elif option == "6":
            combo = vless + trojan + ss
            name = input("File name: ").strip() or "combo_all"
            save_to_file(os.path.join(folder, name + ".txt"), combo)

        elif option == "7":
            name = input("File name: ").strip() or "green_ping"
            save_to_file(os.path.join(folder, name + ".txt"), green)

        elif option == "8":
            name = input("File name: ").strip() or "green_yellow_ping"
            save_to_file(os.path.join(folder, name + ".txt"), green + yellow)

        elif option == "9":
            fname = input("Enter custom filename: ").strip() or "all_configs"
            save_to_file(os.path.join(folder, fname + ".txt"), all_configs)

        elif option == "10":
            fname = input("JSON file name: ").strip() or "output_all"
            save_json(os.path.join(folder, fname + ".json"), all_configs)

        elif option == "11":
            file_name = input("YAML file name: ").strip() or "clash_config"
            yaml_data = {
                "proxies": all_configs,
                "name": "Clash Meta Config",
                "type": "mixed"
            }
            save_yaml(os.path.join(folder, file_name + ".yaml"), yaml_data)
            link = f"https://mago81.ahsan-tepo1383online.workers.dev/clash/akbar98/{quote(file_name)}.yaml"
            print(f"[bold green]YAML saved and link ready:[/bold green] {link}")

        elif option == "12":
            for idx, cfg in enumerate(all_configs, 1):
                save_to_file(os.path.join(folder, f"fragment_{idx}.txt"), [cfg])
            print(f"[cyan]Saved {len(all_configs)} fragments to {folder}[/cyan]")

        elif option == "13":
            combo = vless + ss
            name = input("File name: ").strip() or "combo_vless_ss"
            save_to_file(os.path.join(folder, name + ".txt"), combo)

        elif option == "14":
            combo = trojan + ss
            name = input("File name: ").strip() or "combo_trojan_ss"
            save_to_file(os.path.join(folder, name + ".txt"), combo)

        else:
            print("[red]Invalid option.[/red]")

if __name__ == "__main__":
    main()

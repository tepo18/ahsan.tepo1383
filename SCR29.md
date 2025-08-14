import os
import re
import subprocess
from urllib.parse import quote
from rich import print

protocols = ["vmess://", "vless://", "trojan://", "ss://"]

def extract_host(config):
    match = re.search(r"@([^:/?#]+)", config)
    return match.group(1) if match else None

def ping_host(host):
    try:
        output = subprocess.check_output(
            ["ping", "-c", "3", "-W", "1", host],
            stderr=subprocess.DEVNULL
        ).decode()
        match = re.search(r"rtt min/avg/max/mdev = [\d.]+/([\d.]+)", output)
        return float(match.group(1)) if match else None
    except:
        return None

def classify_ping(ping):
    if ping is None:
        return "bad", "[bold red][BAD][/bold red]"
    elif ping < 150:
        return "good", "[bold green][GOOD][/bold green]"
    elif ping < 300:
        return "warn", "[bold yellow][WARN][/bold yellow]"
    else:
        return "bad", "[bold red][BAD][/bold red]"

def main():
    print("[bold cyan]Enter your configs line by line. Press Enter for empty line (ignored). To finish input, press Ctrl+D:[/bold cyan]")
    lines = []
    while True:
        try:
            line = input()
            if line.strip() == "":
                continue
            lines.append(line.strip())
        except EOFError:
            break

    good_configs = []
    for config in lines:
        if not any(config.startswith(p) for p in protocols):
            continue

        host = extract_host(config)
        if not host:
            print(f"[bold red][INVALID][/bold red] {config}")
            continue

        ping = ping_host(host)
        status, label = classify_ping(ping)
        print(f"{label} {host} - {ping if ping else 'NO REPLY'}ms")

        if status == "good":
            good_configs.append(config)

    if not good_configs:
        print("[bold red]No good configs found.[/bold red]")
        return

    folder = "/storage/emulated/0/Download/Akbar98"
    os.makedirs(folder, exist_ok=True)

    while True:
        print("\nChoose output:")
        print("1) Show in terminal")
        print("2) Save to Akbar98/.txt")
        print("3) Convert to YAML for Clash Meta")
        print("4) Exit")

        choice = input("Enter choice (1/2/3/4): ").strip()

        if choice == "1":
            print("\nFiltered configs:")
            for conf in good_configs:
                print(conf)

        elif choice == "2":
            file_name = input("Enter file name (without .txt): ").strip() or "good"
            path = os.path.join(folder, file_name + ".txt")
            with open(path, "w") as f:
                for conf in good_configs:
                    f.write(conf + "\n")
            print(f"[cyan]Saved to {path}[/cyan]")

        elif choice == "3":
            file_name = input("Enter YAML file name (without .yaml): ").strip() or "clash_conf"
            path = os.path.join(folder, file_name + ".yaml")
            with open(path, "w") as f:
                f.write("proxies:\n")
                for i, conf in enumerate(good_configs, 1):
                    host = extract_host(conf)
                    f.write(f"  - name: proxy{i}\n    type: custom\n    server: {host}\n    raw: '{conf}'\n")

            sublink = f"https://mago81.ahsan-tepo1383online.workers.dev/clash/akbar98/{quote(file_name)}.yaml"
            print(f"[green]✅ YAML saved at: {path}[/green]")
            print(f"[bold cyan]Clash Meta SubLink:[/bold cyan] {sublink}")

        elif choice == "4":
            print("Exiting...")
            break

        else:
            print("[bold red]Invalid choice, please try again.[/bold red]")
            continue

        more = input("Do you want another output? (y/n): ").strip().lower()
        if more != "y":
            break

if __name__ == "__main__":
    main()

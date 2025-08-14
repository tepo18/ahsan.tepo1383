import os
import subprocess
import requests
import re
import json
import base64
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
        return "bad", "[bold red][BAD][/bold red]"
    elif ping < 150:
        return "good", "[bold green][GOOD][/bold green]"
    elif ping < 300:
        return "warn", "[bold yellow][WARN][/bold yellow]"
    else:
        return "bad", "[bold red][BAD][/bold red]"

def fetch_config_from_sublink(url):
    try:
        r = requests.get(url)
        if r.status_code == 200:
            return r.text.splitlines()
    except:
        pass
    return []

def convert_to_clash_yaml(configs):
    yaml_lines = ["proxies:"]
    for i, cfg in enumerate(configs, start=1):
        yaml_lines.append(f"  - name: proxy{i}")
        yaml_lines.append("    type: vmess")
        yaml_lines.append(f"    server: {extract_host(cfg) or 'unknown'}")
        yaml_lines.append("    port: 443")
        yaml_lines.append("    uuid: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx")
        yaml_lines.append("    alterId: 0")
        yaml_lines.append("    cipher: auto")
        yaml_lines.append("    network: ws")
        yaml_lines.append("    ws-path: /")
        yaml_lines.append("    tls: true")
        yaml_lines.append("")
    return "\n".join(yaml_lines)

def main():
    # ---------- ورودی ----------
    print("[bold cyan]Select input method:[/bold cyan]")
    print("1) Manual input")
    print("2) Sublink URL")
    choice = input("[bold yellow]Choose 1 or 2: [/bold yellow]").strip()

    configs = []
    if choice == "1":
        print("[bold cyan]Enter configs (empty line to finish):[/bold cyan]")
        while True:
            line = input()
            if not line.strip(): break
            configs.append(line.strip())
    elif choice == "2":
        url = input("[bold yellow]Enter sublink URL: [/bold yellow]").strip()
        configs = fetch_config_from_sublink(url)
    else:
        print("[bold red]Invalid choice[/bold red]")
        return

    if not configs:
        print("[bold red]No configs provided.[/bold red]")
        return

    # ---------- پینگ گیری ----------
    good_configs = []
    print("\n[bold cyan]Pinging servers...[/bold cyan]")
    for cfg in configs:
        if not any(cfg.startswith(p) for p in protocols):
            continue
        host = extract_host(cfg)
        ping = ping_host(host)
        status, label = classify_ping(ping)
        print(f"{label} {host or 'unknown'} - {ping if ping else 'NO REPLY'}ms")
        if status == "good":
            good_configs.append(cfg)

    if not good_configs:
        print("[bold red]No good configs found.[/bold red]")
        return

    # ---------- خروجی ----------
    folder = "/storage/emulated/0/Download/tepo98"
    os.makedirs(folder, exist_ok=True)

    print("\n[bold green]Select output type:[/bold green]")
    print("1) Normal combined list")
    print("2) Fragmented Config JSON")
    print("3) Simple Raw JSON")
    print("4) Clash YAML Meta")
    print("5) Base64 per line")
    print("6) Full Base64")
    out_choice = input("[bold yellow]Choose 1-6: [/bold yellow]").strip()

    if out_choice == "1":
        path = os.path.join(folder, "output_combined.txt")
        with open(path, "w") as f:
            f.write("\n".join(good_configs))
    elif out_choice == "2":
        path = os.path.join(folder, "output_fragmented.json")
        frag_json = [{"proxy": cfg} for cfg in good_configs]
        with open(path, "w") as f:
            json.dump(frag_json, f, indent=2)
    elif out_choice == "3":
        path = os.path.join(folder, "output_raw.json")
        with open(path, "w") as f:
            json.dump(good_configs, f, indent=2)
    elif out_choice == "4":
        path = os.path.join(folder, "output_clash.yaml")
        yaml_data = convert_to_clash_yaml(good_configs)
        with open(path, "w") as f:
            f.write(yaml_data)
    elif out_choice == "5":
        path = os.path.join(folder, "output_base64_lines.txt")
        with open(path, "w") as f:
            for cfg in good_configs:
                f.write(base64.b64encode(cfg.encode()).decode() + "\n")
    elif out_choice == "6":
        path = os.path.join(folder, "output_base64_all.txt")
        joined = "\n".join(good_configs)
        with open(path, "w") as f:
            f.write(base64.b64encode(joined.encode()).decode())
    else:
        print("[bold red]Invalid output choice[/bold red]")
        return

    print(f"[bold green]Output saved to {path}[/bold green]")

if __name__ == "__main__":
    main()

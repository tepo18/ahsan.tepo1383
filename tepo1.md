import os
import re
import subprocess
from urllib.parse import quote
from rich import print, box
from rich.table import Table
from rich.prompt import Prompt
import requests

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
    except Exception:
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

def combine_configs(*lists):
    combined = []
    length = min(len(lst) for lst in lists)
    for i in range(length):
        combined.append("+".join(lst[i] for lst in lists))
    return combined

def save_to_file(configs, folder, filename):
    os.makedirs(folder, exist_ok=True)
    path = os.path.join(folder, filename)
    with open(path, "w") as f:
        for conf in configs:
            f.write(conf + "\n")
    return path

def get_input_configs():
    print("[bold cyan]Choose input method:[/bold cyan]")
    print("1) Enter configs manually")
    print("2) Enter a subscription link")
    choice = Prompt.ask("Enter 1 or 2", choices=["1","2"])
    
    configs = []
    if choice == "1":
        print("[bold cyan]Enter your configs line by line. Press Enter on empty line to ignore. Ctrl+D to finish input.[/bold cyan]")
        while True:
            try:
                line = input()
                if line.strip() == "":
                    continue
                configs.append(line.strip())
            except EOFError:
                break
    else:
        sub_link = Prompt.ask("Enter subscription URL")
        try:
            r = requests.get(sub_link)
            if r.status_code == 200:
                # Split lines and decode base64 if needed
                raw_lines = r.text.splitlines()
                for line in raw_lines:
                    line = line.strip()
                    if any(line.startswith(p) for p in protocols):
                        configs.append(line)
            else:
                print(f"[red]Failed to fetch subscription. Status code: {r.status_code}[/red]")
        except Exception as e:
            print(f"[red]Error fetching subscription: {e}[/red]")
    return configs

def display_summary(all_configs, VLESS, TROJAN, SS, OTHER, green_configs, yellow_configs, red_configs):
    table = Table(title="Configs Summary", box=box.ROUNDED)
    table.add_column("Category", style="cyan", no_wrap=True)
    table.add_column("Count", justify="right", style="magenta")
    table.add_row("All valid configs", str(len(all_configs)))
    table.add_row("VLESS", str(len(VLESS)))
    table.add_row("TROJAN", str(len(TROJAN)))
    table.add_row("SHADOWSOCKS", str(len(SS)))
    table.add_row("Others", str(len(OTHER)))
    table.add_row("Green ping", str(len(green_configs)))
    table.add_row("Yellow ping", str(len(yellow_configs)))
    table.add_row("Red ping", str(len(red_configs)))
    print()
    print(table)

def main():
    all_configs = get_input_configs()
    valid_configs = [c for c in all_configs if any(c.startswith(p) for p in protocols)]
    if not valid_configs:
        print("[bold red]No valid configs entered.[/bold red]")
        return

    # Categorize
    VLESS, TROJAN, SS, OTHER = [], [], [], []
    green_configs, yellow_configs, red_configs = [], [], []
    ping_results = {}

    print()
    for conf in valid_configs:
        host = extract_host(conf)
        if not host:
            print(f"[bold red][INVALID HOST][/bold red] {conf}")
            continue
        ping = ping_host(host)
        status, label = classify_ping(ping)
        ping_results[conf] = (ping, status)
        print(f"{label} {host} - {str(ping)+' ms' if ping is not None else 'NO REPLY'}")
        
        if conf.startswith("vless://"): VLESS.append(conf)
        elif conf.startswith("trojan://"): TROJAN.append(conf)
        elif conf.startswith("ss://"): SS.append(conf)
        else: OTHER.append(conf)
        
        if status == "good": green_configs.append(conf)
        elif status == "warn": yellow_configs.append(conf)
        else: red_configs.append(conf)

    folder = "/storage/emulated/0/Download/Akbar98"

    while True:
        display_summary(valid_configs, VLESS, TROJAN, SS, OTHER, green_configs, yellow_configs, red_configs)
        print("\n[bold cyan]Select output option:[/bold cyan]")
        print("1) VLESS only\n2) TROJAN only\n3) SHADOWSOCKS only\n4) All configs\n5) Combo (VLESS + TROJAN)\n6) Combo (VLESS + TROJAN + SS)\n7) Green ping only\n8) Green + Yellow ping\n9) Save all configs with custom filename\n10) Upload all configs to GitHub")
        choice = Prompt.ask("Enter choice number", choices=[str(i) for i in range(1, 11)])
        output_configs = []

        if choice == "1": output_configs = VLESS
        elif choice == "2": output_configs = TROJAN
        elif choice == "3": output_configs = SS
        elif choice == "4": output_configs = valid_configs
        elif choice == "5": output_configs = combine_configs(VLESS, TROJAN)
        elif choice == "6": output_configs = combine_configs(VLESS, TROJAN, SS)
        elif choice == "7": output_configs = green_configs
        elif choice == "8": output_configs = green_configs + yellow_configs
        elif choice == "9":
            file_name = Prompt.ask("Enter filename (without extension)", default="good_configs")
            path = save_to_file(valid_configs, folder, file_name+".txt")
            print(f"[green]✅ Saved all configs to {path}[/green]")
            output_configs = []
        elif choice == "10":
            import base64, json
            repo = Prompt.ask("GitHub repo (username/repo)", default="yourusername/yourrepo")
            filename = Prompt.ask("Filename in repo (e.g. configs.txt)", default="configs.txt")
            token = Prompt.ask("GitHub Token (keep private)")
            temp_path = os.path.join(folder, "temp_upload.txt")
            save_to_file(valid_configs, folder, "temp_upload.txt")
            with open(temp_path, "r") as f: content = f.read()
            content_b64 = base64.b64encode(content.encode()).decode()
            url = f"https://api.github.com/repos/{repo}/contents/{filename}"
            headers = {"Authorization": f"token {token}"}
            r = requests.get(url, headers=headers)
            sha = r.json()["sha"] if r.status_code==200 else None
            data = {"message":"Upload configs via script","content":content_b64,"branch":"main"}
            if sha: data["sha"]=sha
            response = requests.put(url, headers=headers, json=data)
            if response.status_code in [200,201]: print(f"[green]✅ Successfully uploaded to GitHub repo {repo} as {filename}[/green]")
            else: print(f"[red]❌ Failed to upload. Status code: {response.status_code}[/red]")
            output_configs=[]
        
        if output_configs:
            print("\nChoose output method:\n1) Show in terminal\n2) Save to file")
            out_method = Prompt.ask("Enter choice", choices=["1","2"], default="1")
            if out_method=="1":
                print("\n[bold green]Filtered configs:[/bold green]")
                for conf in output_configs: print(conf)
            else:
                file_name = Prompt.ask("Enter filename (without extension)", default="output")
                path = save_to_file(output_configs, folder, file_name+".txt")
                print(f"[green]✅ Saved to {path}[/green]")

        again = Prompt.ask("\nDo you want to get another output? (y/n)", choices=["y","n"], default="n")
        if again=="n": break

if __name__ == "__main__":
    main()

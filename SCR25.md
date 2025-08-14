import os
import re
import subprocess
from urllib.parse import quote
from rich import print, box
from rich.table import Table
from rich.prompt import Prompt

protocols = ["vmess://", "vless://", "trojan://", "ss://"]

def extract_host(config):
    """Extract host from config string using regex"""
    match = re.search(r"@([^:/?#]+)", config)
    return match.group(1) if match else None

def ping_host(host):
    """Ping the host 3 times with timeout 1s, return average RTT in ms or None"""
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
    """Classify ping result and return (status_key, rich_colored_label)"""
    if ping is None:
        return "bad", "[bold red][BAD][/bold red]"
    elif ping < 150:
        return "good", "[bold green][GOOD][/bold green]"
    elif ping < 300:
        return "warn", "[bold yellow][WARN][/bold yellow]"
    else:
        return "bad", "[bold red][BAD][/bold red]"

def combine_configs(*lists):
    """Combine configs element-wise (only up to the shortest list length)"""
    combined = []
    length = min(len(lst) for lst in lists)
    for i in range(length):
        combined.append("+".join(lst[i] for lst in lists))
    return combined

def save_to_file(configs, folder, filename):
    """Save list of configs to a text file in folder with filename"""
    os.makedirs(folder, exist_ok=True)
    path = os.path.join(folder, filename)
    with open(path, "w") as f:
        for conf in configs:
            f.write(conf + "\n")
    return path

def main():
    print("[bold cyan]Enter your configs line by line. Press Enter on empty line to ignore. Ctrl+D to finish input.[/bold cyan]")
    lines = []
    while True:
        try:
            line = input()
            if line.strip() == "":
                continue
            lines.append(line.strip())
        except EOFError:
            break

    # Filter valid protocol configs
    valid_configs = [c for c in lines if any(c.startswith(p) for p in protocols)]
    if not valid_configs:
        print("[bold red]No valid configs entered.[/bold red]")
        return

    # Categorize configs by protocol
    VLESS = []
    TROJAN = []
    SS = []
    OTHER = []

    # For ping status groups
    green_configs = []
    yellow_configs = []
    red_configs = []

    # Store ping results for display
    ping_results = {}

    # Ping hosts and classify
    print()
    for conf in valid_configs:
        host = extract_host(conf)
        if not host:
            print(f"[bold red][INVALID HOST][/bold red] {conf}")
            continue
        ping = ping_host(host)
        status, label = classify_ping(ping)
        ping_results[conf] = (ping, status)
        print(f"{label} {host} - {str(ping) + ' ms' if ping is not None else 'NO REPLY'}")

        # Append to protocol list
        if conf.startswith("vless://"):
            VLESS.append(conf)
        elif conf.startswith("trojan://"):
            TROJAN.append(conf)
        elif conf.startswith("ss://"):
            SS.append(conf)
        else:
            OTHER.append(conf)

        # Append to ping color groups
        if status == "good":
            green_configs.append(conf)
        elif status == "warn":
            yellow_configs.append(conf)
        else:
            red_configs.append(conf)

    all_configs = valid_configs

    # Show summary table
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

    # Output options menu
    print("\n[bold cyan]Select output option:[/bold cyan]")
    print("1) VLESS only")
    print("2) TROJAN only")
    print("3) SHADOWSOCKS only")
    print("4) All configs")
    print("5) Combo (VLESS + TROJAN)")
    print("6) Combo (VLESS + TROJAN + SS)")
    print("7) Green ping only")
    print("8) Green + Yellow ping")
    print("9) Save all configs with custom filename")
    print("10) Upload all configs to GitHub (requires token)")

    choice = Prompt.ask("Enter choice number", choices=[str(i) for i in range(1, 11)])

    folder = "/storage/emulated/0/Download/Akbar98"
    output_configs = []

    if choice == "1":
        output_configs = VLESS
    elif choice == "2":
        output_configs = TROJAN
    elif choice == "3":
        output_configs = SS
    elif choice == "4":
        output_configs = all_configs
    elif choice == "5":
        # Combo VLESS + TROJAN
        output_configs = combine_configs(VLESS, TROJAN)
    elif choice == "6":
        # Combo VLESS + TROJAN + SS
        output_configs = combine_configs(VLESS, TROJAN, SS)
    elif choice == "7":
        output_configs = green_configs
    elif choice == "8":
        output_configs = green_configs + yellow_configs
    elif choice == "9":
        # Save all with custom filename
        file_name = Prompt.ask("Enter filename (without extension)", default="good_configs")
        path = save_to_file(all_configs, folder, file_name + ".txt")
        print(f"[green]✅ Saved all configs to {path}[/green]")
        return
    elif choice == "10":
        # Upload to GitHub
        import base64
        import json
        import requests

        repo = Prompt.ask("GitHub repo (username/repo)", default="yourusername/yourrepo")
        filename = Prompt.ask("Filename in repo (e.g. configs.txt)", default="configs.txt")
        token = Prompt.ask("GitHub Token (keep private)")

        # Save all configs temporarily
        temp_path = os.path.join(folder, "temp_upload.txt")
        save_to_file(all_configs, folder, "temp_upload.txt")

        # Prepare content base64 encoded
        with open(temp_path, "r") as f:
            content = f.read()
        content_b64 = base64.b64encode(content.encode()).decode()

        # Prepare GitHub API URL
        url = f"https://api.github.com/repos/{repo}/contents/{filename}"

        # Get current file sha if exists (to update)
        headers = {"Authorization": f"token {token}"}
        r = requests.get(url, headers=headers)
        if r.status_code == 200:
            sha = r.json()["sha"]
        else:
            sha = None

        data = {
            "message": "Upload configs via script",
            "content": content_b64,
            "branch": "main"
        }
        if sha:
            data["sha"] = sha

        response = requests.put(url, headers=headers, json=data)
        if response.status_code in [200, 201]:
            print(f"[green]✅ Successfully uploaded to GitHub repo {repo} as {filename}[/green]")
        else:
            print(f"[red]❌ Failed to upload. Status code: {response.status_code}[/red]")
        return

    # If we reached here, output_configs is selected to show or save
    if not output_configs:
        print("[red]No configs to output.[/red]")
        return

    # Ask user whether to print or save
    print("\nChoose output method:")
    print("1) Show in terminal")
    print("2) Save to file")

    out_method = Prompt.ask("Enter choice", choices=["1", "2"], default="1")

    if out_method == "1":
        print("\n[bold green]Filtered configs:[/bold green]")
        for conf in output_configs:
            print(conf)
    else:
        file_name = Prompt.ask("Enter filename (without extension)", default="output")
        path = save_to_file(output_configs, folder, file_name + ".txt")
        print(f"[green]✅ Saved to {path}[/green]")


if __name__ == "__main__":
    main()

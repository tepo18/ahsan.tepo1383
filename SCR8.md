import os
import base64
import json
import re
import subprocess
from urllib.parse import urlparse, parse_qs, unquote
from rich import print
from rich.prompt import Prompt

# ========================================
# ==== بخش حذف ایموجی (برای نام‌ها) =====
def remove_emojis(text):
    emoji_pattern = re.compile(
        "["
        "\U0001F600-\U0001F64F"  # Emoticons
        "\U0001F300-\U0001F5FF"  # Symbols & Pictographs
        "\U0001F680-\U0001F6FF"  # Transport & Map
        "\U0001F1E0-\U0001F1FF"  # Flags
        "\U00002700-\U000027BF"  # Dingbats
        "\U0001F900-\U0001F9FF"  # Supplemental Symbols and Pictographs
        "\U00002600-\U000026FF"  # Misc symbols
        "]+", flags=re.UNICODE)
    return emoji_pattern.sub(r'', text)

# ========================================
# ==== تبدیل و استخراج اطلاعات کانفیگ
def parse_config_line(config, index):
    protocol = "vmess"
    server = "example.com"
    port = "443"
    uuid = "00000000-0000-0000-0000-000000000000"
    name = f"proxy{index}"
    tls = False
    network = "tcp"
    password = ""
    extra = {}

    config = config.strip()
    proto_match = re.match(r"^(vmess|vless|trojan|ss|shadowsocks)://", config, re.I)
    if proto_match:
        protocol = proto_match.group(1).lower()

    try:
        if protocol == "vmess":
            # اضافه = برای decode بیس64 در صورت ناقص بودن
            decoded = base64.b64decode(config[len("vmess://"):].strip() + "===").decode(errors="ignore")
            data = json.loads(decoded)
            server = data.get("add", server)
            port = str(data.get("port", port))
            uuid = data.get("id", uuid)
            name = data.get("ps", name)
            tls = data.get("tls", "none") != "none"
            network = data.get("net", "tcp")
        elif protocol in ["vless", "trojan"]:
            parsed = urlparse(config)
            server = parsed.hostname or server
            port = str(parsed.port or port)
            uuid = parsed.username or uuid
            name = unquote(parsed.fragment or name)
            params = parse_qs(parsed.query)
            tls = params.get("security", ["none"])[0] == "tls"
            network = params.get("type", [network])[0]
            extra = {
                "servername": params.get("sni", [""])[0],
                "alpn": params.get("alpn", []),
                "skip-cert-verify": params.get("skip-cert-verify", ["false"])[0] == "true",
                "udp": True
            }
        elif protocol in ["ss", "shadowsocks"]:
            parsed = urlparse(config)
            name = unquote(parsed.fragment or name)
            server = parsed.hostname or server
            port = str(parsed.port or port)
            extra["password"] = parsed.password or ""
    except Exception as e:
        print(f"[bold red]Error parsing config {index}: {e}[/bold red]")

    return {
        "name": name,
        "type": protocol,
        "server": server,
        "port": int(port),
        "uuid": uuid,
        "tls": tls,
        "network": network,
        **extra
    }

# ========================================
# ==== تبدیل لیست کانفیگ‌ها به فرمت کلش YAML
def convert_to_clash_yaml(configs):
    proxies = []
    names_seen = set()

    for idx, config in enumerate(configs, 1):
        proxy = parse_config_line(config, idx)

        original_name = proxy["name"]
        count = 1
        while proxy["name"] in names_seen:
            proxy["name"] = f"{original_name}_{count}"
            count += 1
        names_seen.add(proxy["name"])

        lines = [
            f"  - name: \"{proxy['name']}\"",
            f"    type: {proxy['type']}",
            f"    server: {proxy['server']}",
            f"    port: {proxy['port']}",
        ]
        if proxy["type"] in ["vmess", "vless"]:
            lines.append(f"    uuid: {proxy['uuid']}")
        if proxy.get("password"):
            lines.append(f"    password: {proxy['password']}")
        if proxy["type"] != "ss":
            lines.append(f"    network: {proxy['network']}")
            lines.append(f"    tls: {str(proxy['tls']).lower()}")
        if proxy.get("servername"):
            lines.append(f"    servername: {proxy['servername']}")
        if proxy.get("alpn"):
            lines.append(f"    alpn: {json.dumps(proxy['alpn'])}")
        if proxy.get("skip-cert-verify") is not None:
            lines.append(f"    skip-cert-verify: {str(proxy['skip-cert-verify']).lower()}")
        if proxy.get("udp"):
            lines.append(f"    udp: true")

        proxies.append("\n".join(lines))

    return "proxies:\n" + "\n\n".join(proxies)

# ========================================
# ==== تابع استخراج هاست برای پینگ
def extract_host(config):
    match = re.search(r"@([^:/?#]+)", config)
    return match.group(1) if match else None

# ========================================
# ==== تابع پینگ رنگی و دسته‌بندی شده
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

# ========================================
# ==== ورودی دستی
def get_manual_input():
    print("[bold blue]Enter your configs manually. Press Ctrl+D (EOF) when done:[/bold blue]")
    lines = []
    try:
        while True:
            line = input()
            if line.strip():
                lines.append(line.strip())
    except EOFError:
        pass
    return lines

# ========================================
# ==== ورودی از فایل ساب‌اسکریپشن
def get_subscription_input():
    filepath = input("[bold blue]Enter path to subscription file:[/bold blue] ").strip()
    if not os.path.isfile(filepath):
        print("[bold yellow]File not found![/bold yellow]")
        return []
    with open(filepath, "r", encoding="utf-8") as f:
        lines = [line.strip() for line in f if line.strip()]
    print(f"[bold green]Read {len(lines)} lines from subscription.[/bold green]")
    return lines

# ========================================
# ==== تابع اصلی برنامه
def main():
    print("[bold green]Select input method:[/bold green]")
    print("1. Manual input")
    print("2. Load from subscription file")

    choice = input("[bold yellow]Enter choice (1 or 2):[/bold yellow] ").strip()
    if choice == "1":
        configs = get_manual_input()
    elif choice == "2":
        configs = get_subscription_input()
    else:
        print("[bold yellow]Invalid choice, exiting.[/bold yellow]")
        return

    if not configs:
        print("[bold red]No configs entered.[/bold red]")
        return

    # فیلتر فقط کانفیگ‌های معتبر (شروع با پروتکل)
    valid_protocols = ["vmess://", "vless://", "trojan://", "ss://"]
    configs = [c for c in configs if any(c.startswith(p) for p in valid_protocols)]
    if not configs:
        print("[bold red]No valid configs found.[/bold red]")
        return

    # ======= بخش پینگ و فیلتر کانفیگ‌های خوب =======
    print("\n[bold cyan]Checking ping for each config...[/bold cyan]")
    good_configs = []
    for idx, config in enumerate(configs, 1):
        host = extract_host(config)
        if not host:
            print(f"[bold red][INVALID][/bold red] {config}")
            continue

        ping = ping_host(host)
        status, label = classify_ping(ping)
        ping_str = f"{ping:.1f}ms" if ping is not None else "NO REPLY"
        print(f"{label} {host} - {ping_str}")

        if status == "good":
            good_configs.append(config)

    if not good_configs:
        print("[bold red]No good configs found after ping check.[/bold red]")
        return

    # ======= منوی خروجی با همه گزینه‌ها =======
    while True:
        print("\n[bold green]Output Options:[/bold green]")
        print("1) Show good configs here")
        print("2) Save good configs as TXT")
        print("3) Save good configs as Base64 (per line)")
        print("4) Save good configs as Full Base64 (combined)")
        print("5) Save good configs as Clash YAML")
        print("6) Save good configs as JSON proxies list")
        print("7) Save good configs as Fragmented Config (V2Ray)")
        print("8) Save good configs as Raw JSON")

        choice = input("[bold yellow]Select option (1-8):[/bold yellow] ").strip()
        folder = "/storage/emulated/0/Download/tepo98"
        os.makedirs(folder, exist_ok=True)

        if choice == "1":
            print("\n[bold blue]Good configs:[/bold blue]")
            for conf in good_configs:
                print(conf)
        elif choice == "2":
            name = input("Enter file name: ").strip() or "good_configs.txt"
            path = os.path.join(folder, name)
            with open(path, "w", encoding="utf-8") as f:
                f.write("\n".join(good_configs))
            print(f"[bold cyan]Saved to {path}[/bold cyan]")
        elif choice == "3":
            name = input("Enter file name: ").strip() or "good_configs_base64.txt"
            path = os.path.join(folder, name)
            with open(path, "w", encoding="utf-8") as f:
                for conf in good_configs:
                    f.write(base64.b64encode(conf.encode()).decode() + "\n")
            print(f"[bold cyan]Saved to {path}[/bold cyan]")
        elif choice == "4":
            name = input("Enter file name: ").strip() or "good_configs_full_base64.txt"
            combined = base64.b64encode("\n".join(good_configs).encode()).decode()
            path = os.path.join(folder, name)
            with open(path, "w", encoding="utf-8") as f:
                f.write(combined)
            print(f"[bold cyan]Saved to {path}[/bold cyan]")
        elif choice == "5":
            name = input("Enter YAML file name: ").strip() or "good_configs.yaml"
            yaml_data = convert_to_clash_yaml(good_configs)
            path = os.path.join(folder, name)
            with open(path, "w", encoding="utf-8") as f:
                f.write(yaml_data)
            print(f"[bold cyan]Saved to {path}[/bold cyan]")
        elif choice == "6":
            name = input("Enter JSON file name: ").strip() or "good_configs.json"
            proxies = []
            for i, c in enumerate(good_configs, 1):
                proxy = parse_config_line(c, i)
                base_name = remove_emojis(proxy["name"]).strip()
                proxy["name"] = f"{base_name} {i}"
                proxies.append(proxy)
            path = os.path.join(folder, name)
            with open(path, "w", encoding="utf-8") as f:
                json.dump({"proxies": proxies}, f, indent=2)
            print(f"[bold cyan]Saved to {path}[/bold cyan]")
        elif choice == "7":
            # Fragmented Config - نمونه ساده (همانند هدف 4)
            name = input("Enter file name: ").strip() or "fragmented_config.txt"
            path = os.path.join(folder, name)
            with open(path, "w", encoding="utf-8") as f:
                f.write("// Fragmented Config placeholder\n")
                f.write("\n".join(good_configs))
            print(f"[bold cyan]Saved to {path}[/bold cyan]")
        elif choice == "8":
            # Raw JSON خروجی مستقیم لینک‌ها به صورت آرایه
            name = input("Enter JSON file name: ").strip() or "raw_configs.json"
            path = os.path.join(folder, name)
            with open(path, "w", encoding="utf-8") as f:
                json.dump(good_configs, f, indent=2)
            print(f"[bold cyan]Saved to {path}[/bold cyan]")
   :
            print("[bold red]Invalid option.[/bold red]")
            continue
vv
        again = input("\n[bold magenta]Do you want to continue? (y/n):[/bold magenta] ").strip().lower()
        if again != "y":
            break

if __name__ == "__main__":
    main()

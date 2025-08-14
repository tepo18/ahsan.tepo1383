import os
import re
import subprocess
import requests
import json
import yaml
from urllib.parse import quote
from rich import print

# ---------- تنظیمات ----------
SAVE_DIR = "/storage/emulated/0/Download/king-pro"
os.makedirs(SAVE_DIR, exist_ok=True)

protocols = ["vmess://", "vless://", "trojan://", "ss://"]

# ---------- ابزار پینگ ----------
def extract_host(config):
    match = re.search(r"@([^:/?#]+)", config)
    return match.group(1) if match else None

def ping_host(host):
    try:
        output = subprocess.check_output(
            ["ping", "-c", "3", "-W", "1", host],
            stderr=subprocess.DEVNULL
        ).decode(errors="ignore")
        match = re.search(r"rtt min/avg/max/(?:mdev|stddev) = [\d.]+/([\d.]+)", output)
        return float(match.group(1)) if match else None
    except Exception:
        return None

def classify_ping(p):
    if p is None:
        return "red", "[bold red][BAD][/bold red]"
    elif p < 150:
        return "green", "[bold green][GOOD][/bold green]"
    elif p < 300:
        return "yellow", "[bold yellow][WARN][/bold yellow]"
    else:
        return "red", "[bold red][BAD][/bold red]"

# ---------- ورودی از ساب‌لینک ----------
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
    return [line.strip() for line in lines if line.strip()]

# ---------- تشخیص نوع/تبدیل JSON به vless ----------
def is_valid_uuid(u):
    return re.fullmatch(r'[0-9a-fA-F-]{36}', u) is not None

def detect_config_type(text):
    if text.startswith("vless://"):
        return "vless"
    elif text.startswith("vmess://"):
        return "vmess"
    elif text.startswith("trojan://"):
        return "trojan"
    elif text.startswith("ss://"):
        return "shadowsocks"
    elif '"protocol":"vless"' in text and '"vnext"' in text:
        return "json_vless"
    elif '"protocol":"vmess"' in text:
        return "json_vmess"
    elif '"protocol":"trojan"' in text:
        return "json_trojan"
    elif '"protocol":"shadowsocks"' in text:
        return "json_shadowsocks"
    elif '"protocol":"wireguard"' in text:
        return "json_wireguard"
    return "unknown"

def extract_json_blocks(text):
    # بلاک‌های JSON دارای "protocol": "..."
    return re.findall(r'{.*?"protocol"\s*:\s*".*?".*?}', text, re.DOTALL)

def convert_json_to_vless(js):
    try:
        cfg = json.loads(js)
        out = cfg.get("outbounds", [{}])[0]
        vnext = out.get("settings", {}).get("vnext", [{}])[0]
        user = vnext.get("users", [{}])[0]
        uuid = user.get("id", "")
        address = vnext.get("address", "")
        port = vnext.get("port", "")
        stream = out.get("streamSettings", {})
        net = stream.get("network", "tcp")
        sec = stream.get("security", "tls")
        tls = stream.get("tlsSettings", {}) or {}
        sni = tls.get("serverName", "")
        fp = tls.get("fingerprint", "")
        path = ""
        if net == "ws":
            path = stream.get("wsSettings", {}).get("path", "")
        elif net == "grpc":
            path = stream.get("grpcSettings", {}).get("serviceName", "")
        if not (uuid and address and port and is_valid_uuid(uuid)):
            return None
        return f"vless://{uuid}@{address}:{port}?encryption=none&type={net}&security={sec}&sni={sni}&fp={fp}&path={path}#{address}"
    except Exception:
        return None

# ---------- دسته‌بندی کلی ----------
def categorize_configs(configs):
    vless, trojan, ss, others = [], [], [], []
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

def save_to_file(filename, data):
    with open(filename, "w", encoding="utf-8") as f:
        for d in data:
            f.write(d + "\n")

def save_json(filename, data):
    with open(filename, "w", encoding="utf-8") as f:
        json.dump(data, f, ensure_ascii=False, indent=2)

def save_yaml(filename, data):
    with open(filename, "w", encoding="utf-8") as f:
        yaml.dump(data, f, allow_unicode=True)

# ---------- دو خروجیِ مثل کد دوم (VLESS only) ----------
def parse_vless_link(link):
    """فقط VLESS مثل کد دوم"""
    link = link.strip()
    if not link.startswith("vless://"):
        return None
    try:
        main_part = link.split("://", 1)[1]
        if "#" in main_part:
            main_part, remark = main_part.split("#", 1)
        else:
            remark = ""
        if "?" in main_part:
            addr_port, params = main_part.split("?", 1)
            from urllib.parse import parse_qs, unquote
            query = {k: v[0] for k, v in parse_qs(params).items()}
        else:
            addr_port, query, unquote = main_part, {}, lambda s: s
        if '@' in addr_port:
            user, host_port = addr_port.split("@", 1)
        else:
            user, host_port = "", addr_port
        if ':' in host_port:
            address, port = host_port.split(":", 1)
        else:
            address, port = host_port, "443"
        return {
            "protocol": "vless",
            "address": address,
            "port": int(port) if port.isdigit() else 443,
            "id": user,
            "params": query,
            "remark": remark
        }
    except Exception:
        return None

def build_fragment_list(vless_objs):
    """لیست کانفیگ‌های فرگمنت مانند کد دوم (ساده‌شده)"""
    fragments = []
    for i, cfg in enumerate(vless_objs):
        outbound = {
            "tag": "proxy",
            "protocol": "vless",
            "settings": {
                "vnext": [{
                    "address": cfg["address"],
                    "port": cfg["port"],
                    "users": [{
                        "id": cfg["id"],
                        "encryption": cfg["params"].get("encryption", "none"),
                        "flow": cfg["params"].get("flow", "")
                    }]
                }]
            },
            "streamSettings": {
                "network": cfg["params"].get("type", "ws"),
                "security": cfg["params"].get("security", "tls"),
                "sockopt": {"dialerProxy": "fragment"},
                "tlsSettings": {
                    "serverName": cfg["params"].get("sni", cfg["params"].get("host", cfg["address"])),
                    "fingerprint": cfg["params"].get("fp", "chrome"),
                    "alpn": cfg["params"].get("alpn", "").split(",") if "alpn" in cfg["params"] else ["http/1.1"]
                },
                "wsSettings": {
                    "path": cfg["params"].get("path", "/"),
                    "headers": {"Host": cfg["params"].get("host", cfg["address"])}
                }
            }
        }
        fragment = {
            "remarks": cfg.get("remark") or f"Config {i+1}",
            "log": {"loglevel": "warning"},
            "dns": {},
            "inbounds": [],
            "outbounds": [outbound],
            "routing": {}
        }
        fragments.append(fragment)
    return fragments

# ---------- برنامه اصلی ----------
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

    # حذف تکراری‌ها
    all_configs = list(dict.fromkeys([c.strip() for c in all_configs if c.strip()]))

    # آمار اولیه
    vless, trojan, ss, others = categorize_configs(all_configs)
    print(f"\n[bold]Total configs:[/bold] {len(all_configs)}")
    print(f"VLESS: {len(vless)}")
    print(f"TROJAN: {len(trojan)}")
    print(f"SHADOWSOCKS: {len(ss)}")
    print(f"Others: {len(others)}")

    # پینگ‌گیری
    green, yellow, red = [], [], []
    print("\n[bold cyan]Performing ping checks...[/bold cyan]")
    for config in all_configs:
        host = extract_host(config)
        if not host:
            print(f"[red][INVALID] {config}[/red]")
            continue
        p = ping_host(host)
        status, label = classify_ping(p)
        print(f"{label} {host} - {p if p is not None else 'NO REPLY'}ms")
        if status == "green":
            green.append(config)
        elif status == "yellow":
            yellow.append(config)
        else:
            red.append(config)

    print("\n[bold]Ping Summary:[/bold]")
    print(f"[green]GREEN:[/green] {len(green)}")
    print(f"[yellow]YELLOW:[/yellow] {len(yellow)}")
    print(f"[red]RED:[/red] {len(red)}")

    # دسته‌بندی دقیق + تبدیل JSONهای vless به vless://
    configs = all_configs[:]  # پس از پینگ همان ورودی‌ها
    input_count = len(configs)

    detected = {
        "vless": [],
        "vmess": [],
        "trojan": [],
        "shadowsocks": [],
        "wireguard": [],
        "json_vless": [],
        "json_vmess": [],
        "json_trojan": [],
        "json_shadowsocks": [],
        "json_wireguard": [],
        "unknown": [],
        "errors": []
    }

    for line in configs:
        t = detect_config_type(line)
        if t in ["vless", "vmess", "trojan", "shadowsocks", "wireguard"]:
            detected[t].append(line)
        elif t.startswith("json_"):
            if t == "json_vless":
                cv = convert_json_to_vless(line)
                if cv:
                    detected["vless"].append(cv)
                else:
                    detected["errors"].append(line)
            else:
                detected[t].append(line)
        elif "outbounds" in line:
            blocks = extract_json_blocks(line)
            for block in blocks:
                cv = convert_json_to_vless(block)
                if cv:
                    detected["vless"].append(cv)
                else:
                    detected["errors"].append(block)
        else:
            detected["unknown"].append(line)

    total_output = (
        detected["vless"] + detected["vmess"] + detected["trojan"] +
        detected["shadowsocks"] + detected["wireguard"] +
        detected["json_vmess"] + detected["json_trojan"] +
        detected["json_shadowsocks"] + detected["json_wireguard"]
    )

    print(f"\nکانفیگ‌های فیلتر شده (بدون singbox): {input_count}")
    print("دسته بندی‌ها:")
    for k in ["vless", "vmess", "trojan", "shadowsocks", "wireguard",
              "json_vless", "json_vmess", "json_trojan", "json_shadowsocks", "json_wireguard", "unknown"]:
        print(f"  - {k}: {len(detected[k])}")

    if len(total_output) == 0:
        print("\nهیچ کانفیگی قابل تبدیل نیست.")
        return

    # --------- منوی خروجی‌ها (اصلی + دو خروجی اضافه) با امکان تکرار ---------
    while True:
        print("\n🔄 نوع خروجی را انتخاب کنید:")
        print("1) VLESS fragment list (vless:// ...)")
        print("2) VMess fragment list")
        print("3) Shadowsocks fragment list")
        print("4) WireGuard fragment list")
        print("5) All as JSON array (raw strings)")
        print("6) YAML fragments (vless/vmess/ss/wg lists)")
        print("7) Plain text (all lines)")
        print("8) Fragmented Config JSON (VLESS only, like code #2)")
        print("9) Simple Raw JSON (parsed VLESS objects, like code #2)")
        print("0) خروج")

        opt = input("شماره گزینه (0-9): ").strip()
        if opt == "0":
            print("[cyan]خروج.[/cyan]")
            break

        # نام فایل برای هر خروجی
        name = input("📄 نام فایل (بدون پسوند): ").strip() or "output"

        try:
            if opt == "1":
                path = os.path.join(SAVE_DIR, f"{name}_vless.txt")
                save_to_file(path, detected["vless"])
            elif opt == "2":
                path = os.path.join(SAVE_DIR, f"{name}_vmess.txt")
                save_to_file(path, detected["vmess"])
            elif opt == "3":
                path = os.path.join(SAVE_DIR, f"{name}_ss.txt")
                save_to_file(path, detected["shadowsocks"])
            elif opt == "4":
                path = os.path.join(SAVE_DIR, f"{name}_wg.txt")
                save_to_file(path, detected["wireguard"])
            elif opt == "5":
                path = os.path.join(SAVE_DIR, f"{name}_all.json")
                save_json(path, total_output)
            elif opt == "6":
                path = os.path.join(SAVE_DIR, f"{name}_fragments.yaml")
                yaml_data = {
                    "vless": detected["vless"],
                    "vmess": detected["vmess"],
                    "shadowsocks": detected["shadowsocks"],
                    "wireguard": detected["wireguard"]
                }
                save_yaml(path, yaml_data)
            elif opt == "7":
                path = os.path.join(SAVE_DIR, f"{name}_all.txt")
                save_to_file(path, total_output)
            elif opt == "8":
                # Fragmented Config JSON (VLESS only)
                vless_objs = []
                for s in detected["vless"]:
                    obj = parse_vless_link(s)
                    if obj:
                        vless_objs.append(obj)
                if not vless_objs:
                    print("[red]❌ هیچ VLESS معتبری برای ساخت فرگمنت یافت نشد.[/red]")
                    continue
                fragments = build_fragment_list(vless_objs)
                path = os.path.join(SAVE_DIR, f"{name}_fragmented.json")
                save_json(path, fragments)
            elif opt == "9":
                # Simple Raw JSON of parsed VLESS objects
                vless_objs = []
                for s in detected["vless"]:
                    obj = parse_vless_link(s)
                    if obj:
                        vless_objs.append(obj)
                if not vless_objs:
                    print("[red]❌ هیچ VLESS معتبری برای ساخت JSON ساده یافت نشد.[/red]")
                    continue
                path = os.path.join(SAVE_DIR, f"{name}_vless_parsed.json")
                save_json(path, vless_objs)
            else:
                print("[red]گزینه نامعتبر.[/red]")
                continue

            print(f"[bold green]✅ ذخیره شد:[/bold green] {path}")

        except Exception as e:
            print(f"[red]❌ خطا در ذخیره خروجی: {e}[/red]")

        again = input("آیا خروجی دیگری با همین ورودی بگیرم؟ (y/n): ").strip().lower()
        if again != "y":
            break

if __name__ == "__main__":
    main()

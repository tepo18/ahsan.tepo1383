import os
import sys
import requests
import itertools
import json
import urllib.parse
from colorama import init, Fore

init(autoreset=True)

SAVE_PATH = "/storage/emulated/0/Download/king-pro"
os.makedirs(SAVE_PATH, exist_ok=True)

def detect_type(link):
    if link.startswith("vless://"):
        return "vless"
    elif link.startswith("trojan://"):
        return "trojan"
    elif link.startswith("ss://"):
        return "ss"
    return "unknown"

def get_configs_from_input():
    print(Fore.CYAN + "\nPaste your configs (one per line), then press Enter and Ctrl+D (or Ctrl+Z on Windows):\n")
    lines = []
    try:
        while True:
            line = input()
            if line.strip():
                lines.append(line.strip())
    except EOFError:
        pass
    return lines

def get_configs_from_subscription():
    url = input(Fore.CYAN + "🔗 Enter subscription URL: ").strip()
    try:
        response = requests.get(url, timeout=10)
        if response.status_code == 200:
            content = response.text.strip()
            if "+" in content:
                return content.split("+")
            elif "\n" in content:
                return content.splitlines()
            else:
                return [content]
        else:
            print(Fore.RED + f"❌ Failed to fetch: {response.status_code}")
            return []
    except Exception as e:
        print(Fore.RED + f"❌ Error: {e}")
        return []

# ----------- Fragment/JSON building from second code -----------

def parse_link(link):
    link = link.strip()
    parsed = {}
    if link.startswith("vless://"):
        protocol = "vless"
    else:
        return None, None

    try:
        main_part = link.split("://")[1]
        if "#" in main_part:
            main_part, remark = main_part.split("#", 1)
        else:
            remark = ""

        if "?" in main_part:
            addr_port, params = main_part.split("?", 1)
            query = urllib.parse.parse_qs(params)
        else:
            addr_port = main_part
            query = {}

        if '@' in addr_port:
            user, host_port = addr_port.split("@", 1)
        else:
            user = ""
            host_port = addr_port

        if ':' in host_port:
            address, port = host_port.split(":", 1)
        else:
            address, port = host_port, "443"

        parsed = {
            "protocol": protocol,
            "address": address,
            "port": int(port) if port.isdigit() else 443,
            "id": user,
            "params": {k: v[0] for k, v in query.items()},
            "remark": urllib.parse.unquote(remark)
        }
        return protocol, parsed

    except Exception:
        return None, None

def build_fragment_list(configs):
    fragments = []
    for i, cfg in enumerate(configs):
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
                "sockopt": {
                    "dialerProxy": "fragment"
                },
                "tlsSettings": {
                    "serverName": cfg["params"].get("sni", cfg["params"].get("host", cfg["address"])),
                    "fingerprint": cfg["params"].get("fp", "chrome"),
                    "alpn": cfg["params"].get("alpn", "").split(",") if "alpn" in cfg["params"] else ["http/1.1"]
                },
                "wsSettings": {
                    "path": cfg["params"].get("path", "/"),
                    "headers": {
                        "Host": cfg["params"].get("host", cfg["address"])
                    }
                }
            }
        }

        fragment = {
            "remarks": cfg["remark"] or f"💦 {i+1} - VLESS",
            "log": {"loglevel": "warning"},
            "dns": {},
            "inbounds": [],
            "outbounds": [outbound],
            "routing": {}
        }
        fragments.append(fragment)

    return fragments

def save_all_outputs(combinations):
    while True:
        print("\nSelect output type:\n1. Normal combined list\n2. Fragmented Config (JSON)\n3. Simple Raw JSON")
        choice = input("Enter your choice: ").strip()
        filename = input(Fore.CYAN + "📁 Enter output filename (without .txt/.json): ").strip()

        if choice == "1":
            combo_file = os.path.join(SAVE_PATH, f"{filename}_combined.txt")
            with open(combo_file, "w") as f:
                for item in combinations:
                    f.write(item + '\n')
            print(Fore.GREEN + f"\n✅ Saved list to: {combo_file}")

        else:
            parsed_configs = []
            for line in combinations:
                parts = line.split("+")
                for p in parts:
                    proto, parsed = parse_link(p)
                    if proto:
                        parsed_configs.append(parsed)

            if not parsed_configs:
                print(Fore.RED + "❌ No valid configs for JSON output.")
                return

            save_path = SAVE_PATH
            if choice == "2":
                output = build_fragment_list(parsed_configs)
            else:
                output = parsed_configs

            full_path = os.path.join(save_path, f"{filename}.json")
            with open(full_path, "w", encoding="utf-8") as f:
                json.dump(output, f, indent=2, ensure_ascii=False)
            print(Fore.GREEN + f"\n✅ Saved JSON to: {full_path}")

        again = input("\nDo you want to export again with same input? (y/n): ").strip().lower()
        if again != "y":
            break

def main():
    print(Fore.YELLOW + "\n🧩 Config Combination Generator")
    print(Fore.BLUE + "\nSelect input method:")
    print("1️⃣  Manual paste")
    print("2️⃣  Subscription URL")
    choice = input(Fore.GREEN + "Enter 1 or 2: ").strip()

    if choice == "1":
        lines = get_configs_from_input()
    elif choice == "2":
        lines = get_configs_from_subscription()
    else:
        print(Fore.RED + "❌ Invalid choice.")
        return

    lines = [l.strip() for l in lines if l.strip()]
    types = {"vless": [], "trojan": [], "ss": [], "unknown": []}

    for line in lines:
        types[detect_type(line)].append(line)

    valid_types = {k: v for k, v in types.items() if v and k != "unknown"}
    type_keys = list(valid_types.keys())

    if not valid_types:
        print(Fore.RED + "❌ No valid configs detected.")
        return

    print(Fore.GREEN + f"\n✅ Detected types: {', '.join(type_keys)}")

    combinations = []
    if len(valid_types) == 1:
        only = type_keys[0]
        combos = itertools.combinations(valid_types[only], 3)
        combinations = ['+'.join(c) for c in combos]
    elif len(valid_types) == 2:
        t1, t2 = type_keys
        combinations = [f"{a}+{b}" for a in valid_types[t1] for b in valid_types[t2]]
    elif len(valid_types) >= 3:
        t1, t2, t3 = type_keys[:3]
        combinations = [f"{a}+{b}+{c}" for a in valid_types[t1] for b in valid_types[t2] for c in valid_types[t3]]

    combinations = combinations[:20]
    if not combinations:
        print(Fore.RED + "❌ No combinations to save.")
        return

    save_all_outputs(combinations)

if __name__ == "__main__":
    main()

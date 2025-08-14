import os
import json
import urllib.parse

class Colors:
    BLUE = '\033[94m'
    YELLOW = '\033[93m'
    GREEN = '\033[92m'
    END = '\033[0m'

def cprint(text, color):
    print(f"{color}{text}{Colors.END}")

def parse_link(link):
    link = link.strip()
    if not link.startswith("vless://"):
        return None, None
    try:
        main_part = link.split("://", 1)[1]
        remark = ""
        if "#" in main_part:
            main_part, remark = main_part.split("#", 1)
        if "?" in main_part:
            addr_port, params = main_part.split("?", 1)
            query = urllib.parse.parse_qs(params)
        else:
            addr_port = main_part
            query = {}
        if "@" in addr_port:
            user, host_port = addr_port.split("@", 1)
        else:
            user = ""
            host_port = addr_port
        if ":" in host_port:
            address, port = host_port.split(":", 1)
        else:
            address = host_port
            port = "443"
        return "vless", {
            "protocol": "vless",
            "address": address,
            "port": int(port) if port.isdigit() else 443,
            "id": user,
            "params": {k: v[0] for k, v in query.items()},
            "remark": urllib.parse.unquote(remark)
        }
    except:
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
                "sockopt": {"dialerProxy": "fragment"},
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
            "remarks": cfg["remark"] or f"Config {i+1}",
            "log": {"loglevel": "warning"},
            "dns": {},
            "inbounds": [],
            "outbounds": [outbound],
            "routing": {}
        }
        fragments.append(fragment)
    return fragments

def save_output(configs):
    cprint("\nSelect output type:", Colors.BLUE)
    cprint("1. Fragmented Config (V2Ray)", Colors.YELLOW)
    cprint("2. Raw JSON", Colors.YELLOW)
    choice = input("Enter your choice (1 or 2): ").strip()

    file_name = input("Enter output file name (without extension): ").strip()
    save_path = "/sdcard/Download/almasi98"
    os.makedirs(save_path, exist_ok=True)
    full_path = os.path.join(save_path, f"{file_name}.json")

    if choice == "1":
        output = build_fragment_list(configs)
    else:
        output = configs

    with open(full_path, "w", encoding="utf-8") as f:
        json.dump(output, f, indent=2, ensure_ascii=False)

    cprint(f"\n✅ Saved to: {full_path}", Colors.GREEN)

def main():
    cprint("=" * 40, Colors.BLUE)
    cprint("Paste your configs (Ctrl+D to finish):", Colors.YELLOW)
    lines = []
    try:
        while True:
            line = input()
            if line.strip():
                lines.append(line.strip())
    except EOFError:
        pass

    parsed_configs = []
    for line in lines:
        proto, parsed = parse_link(line)
        if proto:
            parsed_configs.append(parsed)

    if not parsed_configs:
        cprint("❌ No valid configs found.", Colors.YELLOW)
        return

    save_output(parsed_configs)

if __name__ == "__main__":
    main()

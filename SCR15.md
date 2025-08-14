import json
import sys
import base64
import re
import yaml
import os

GREEN = "\033[92m"
YELLOW = "\033[93m"
BLUE = "\033[94m"
RESET = "\033[0m"

def is_valid_uuid(uuid):
    return re.fullmatch(r'[0-9a-fA-F-]{36}', uuid) is not None

def detect_config_type(line):
    line = line.strip()
    if line.startswith("vless://"):
        return "vless"
    elif line.startswith("vmess://"):
        return "vmess"
    elif line.startswith("trojan://"):
        return "trojan"
    elif line.startswith("ss://"):
        return "shadowsocks"
    elif line.startswith("wg://") or line.startswith("wireguard://"):
        return "wireguard"
    try:
        js = json.loads(line)
        protocol = js.get("protocol", "").lower()
        if protocol == "vless":
            return "json_vless"
        elif protocol == "vmess":
            return "json_vmess"
        elif protocol == "trojan":
            return "json_trojan"
        elif protocol == "shadowsocks":
            return "json_shadowsocks"
        elif protocol == "wireguard":
            return "json_wireguard"
    except:
        pass
    return "unknown"

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
        sni = stream.get("tlsSettings", {}).get("serverName", "")
        path = ""
        if net == "ws":
            path = stream.get("wsSettings", {}).get("path", "")
        elif net == "grpc":
            path = stream.get("grpcSettings", {}).get("serviceName", "")
        if not (uuid and address and port and is_valid_uuid(uuid)):
            return None
        return f"vless://{uuid}@{address}:{port}?encryption=none&type={net}&security={sec}&sni={sni}&path={path}#{address}"
    except:
        return None

def save_output(choice, detected, base_path):
    filename = input(GREEN + "Enter file name (without extension): " + RESET).strip()
    if not filename:
        print(YELLOW + "Invalid file name." + RESET)
        return None

    extension = "txt"
    content = []

    if choice == "1":
        content = detected["vless"] + detected["json_vless"]
    elif choice == "2":
        content = detected["vmess"] + detected["json_vmess"]
    elif choice == "3":
        content = detected["shadowsocks"] + detected["json_shadowsocks"]
    elif choice == "4":
        content = detected["wireguard"] + detected["json_wireguard"]
    elif choice == "5":
        extension = "json"
        content = []
        for key in detected:
            content.extend(detected[key])
    elif choice == "6":
        extension = "yaml"
        content = {}
        for key in ["vless", "vmess", "shadowsocks", "wireguard"]:
            content[key] = detected[key]
    elif choice == "7":
        content = []
        for key in detected:
            content.extend(detected[key])
    elif choice == "8":
        fragment_links = []
        all_vless = detected["vless"] + detected["json_vless"]
        for i in range(0, len(all_vless), 3):
            combo = "+".join(all_vless[i:i+3])
            if combo:
                fragment_links.append(combo)
        content = fragment_links
    elif choice == "9":
        extension = "json"
        content = detected
    else:
        print(YELLOW + "Invalid option." + RESET)
        return None

    os.makedirs(base_path, exist_ok=True)
    result_path = os.path.join(base_path, f"{filename}.{extension}")

    try:
        with open(result_path, "w", encoding="utf-8") as f:
            if choice in ["5", "9"]:
                json.dump(content, f, ensure_ascii=False, indent=2)
            elif choice == "6":
                yaml.dump(content, f, allow_unicode=True)
            else:
                for item in content:
                    f.write(item + "\n")
        print(GREEN + f"\nSaved output to: {result_path}" + RESET)
        return result_path
    except Exception as e:
        print(YELLOW + f"Error writing file: {e}" + RESET)
        return None

def main():
    print(BLUE + "Choose input method:" + RESET)
    print("1) Paste configs manually")
    print("2) Load configs from subscription file (full path)")

    input_method = input("Select input method (1 or 2): ").strip()

    if input_method == "1":
        print(BLUE + "Paste your configs (links or JSON), then press Enter and Ctrl+D:\n" + RESET)
        input_text = sys.stdin.read().strip()
        if not input_text:
            print(YELLOW + "No input detected." + RESET)
            return
        lines = [l.strip() for l in input_text.splitlines() if l.strip()]
    elif input_method == "2":
        file_path = input(GREEN + "Enter subscription file path: " + RESET).strip()
        if not os.path.isfile(file_path):
            print(YELLOW + "File not found." + RESET)
            return
        with open(file_path, "r", encoding="utf-8") as f:
            lines = [l.strip() for l in f if l.strip()]
    else:
        print(YELLOW + "Invalid input method." + RESET)
        return

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
        "unknown": []
    }

    for line in lines:
        t = detect_config_type(line)
        if t.startswith("json_"):
            if t == "json_vless":
                conv = convert_json_to_vless(line)
                if conv:
                    detected["json_vless"].append(conv)
                else:
                    detected["unknown"].append(line)
            else:
                detected[t].append(line)
        else:
            detected[t].append(line)

    total_count = sum(len(v) for v in detected.values())

    print(BLUE + f"\nDetected configs: {total_count}" + RESET)
    for key in detected:
        print(f"  {key}: {len(detected[key])}")

    if total_count == 0:
        print(YELLOW + "No configs detected." + RESET)
        return

    while True:
        print(BLUE + "\nChoose output format:" + RESET)
        print("1) VLESS fragment links")
        print("2) VMess links")
        print("3) Shadowsocks links")
        print("4) WireGuard links")
        print("5) JSON (all configs)")
        print("6) YAML fragment")
        print("7) Plain text fragments")
        print("8) Fragmented VLESS combo links (3 links combined)")
        print("9) Raw JSON detected configs")

        choice = input("Option (1-9): ").strip()
        save = input("Save output to /storage/emulated/0/Download/tepo81/? (y/n): ").strip().lower()
        base_path = "/storage/emulated/0/Download/tepo81" if save == "y" else "."

        saved_file = save_output(choice, detected, base_path)

        if not saved_file:
            print(YELLOW + "No file saved." + RESET)

        again = input("\nDo you want another output? (y/n): ").strip().lower()
        if again != "y":
            print(GREEN + "Exiting." + RESET)
            break

if __name__ == "__main__":
    main()

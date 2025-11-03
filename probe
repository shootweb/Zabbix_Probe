#!/usr/bin/env python3
"""
zabbix_trapper_probe.py

Probe Zabbix trapper endpoints using the local zabbix_sender binary.
Use only with explicit written authorization for the target(s).

Produces:
 - found_trapper.txt (successful host/key combos where server processed > 0 and failed == 0)
 - logs/<target>_<timestamp>.txt (raw zabbix_sender output for each target)

Example:
 python3 zabbix_trapper_probe.py --targets 10.0.0.5 10.0.0.6 --port 10052

Note:
 This wrapper invokes the zabbix_sender program. Install it if missing (apt install zabbix-sender or equivalent).
"""
import argparse
import os
import shutil
import subprocess
import sys
import time
from datetime import datetime
from pathlib import Path

# defaults
DEFAULT_PORT = 10051
DEFAULT_HOST_GUESSES = [
    "localhost",
    "localhost.localdomain",
    "server",
    "server01",
    "host01",
    "zabbix-server",
    "zabbix",
    "web01",
    "app01",
]
DEFAULT_KEYS = [
    "test.key",
    "system.uptime",
    "agent.ping",
    "custom.test"
]

OUT_DIR = Path("logs")
OUT_DIR.mkdir(exist_ok=True)

def which_zabbix_sender():
    binname = shutil.which("zabbix_sender")
    if not binname:
        print("[!] zabbix_sender binary not found in PATH. Install it and re-run.")
        return None
    return binname

def run_sender(binpath, target_ip, port, hostname, key, value="1", timeout=10, proxy=None):
    # Build command
    cmd = [binpath, "-z", target_ip, "-p", str(port), "-s", hostname, "-k", key, "-o", value, "-vv"]
    env = os.environ.copy()
    if proxy:
        env["http_proxy"] = proxy
        env["https_proxy"] = proxy

    # run and capture output
    start = time.time()
    try:
        proc = subprocess.run(cmd, stdout=subprocess.PIPE, stderr=subprocess.STDOUT, env=env, timeout=timeout, check=False)
        out = proc.stdout.decode(errors="replace")
    except subprocess.TimeoutExpired:
        out = "[!] zabbix_sender timed out"
    except Exception as e:
        out = f"[!] zabbix_sender invocation failed: {e}"
    duration = time.time() - start
    return out, duration

def parse_sender_output(output):
    """
    Parse typical zabbix_sender outputs.
    Will try to find:
     - sent: N
     - processed: N
     - failed: N
    Returns dict with keys: sent, processed, failed (ints or None)
    """
    out = {"sent": None, "processed": None, "failed": None}
    # common text patterns
    for line in output.splitlines():
        line = line.strip()
        # patterns: "info: processed: 1; failed: 0; total: 1; seconds spent: 0.000046"
        if "processed:" in line and "failed:" in line:
            # try to extract numbers
            try:
                parts = line.replace(";", " ").replace(",", " ").split()
                for i,p in enumerate(parts):
                    if p.startswith("processed:"):
                        out["processed"] = int(parts[i+1])
                    if p.startswith("failed:"):
                        out["failed"] = int(parts[i+1])
            except Exception:
                pass
        # pattern: "sent: 1"
        if line.startswith("sent:"):
            try:
                out["sent"] = int(line.split(":")[1].strip())
            except Exception:
                pass
        # Sometimes alternative formatting: "info from server: processed: 1; failed: 0"
        if line.startswith("info from server:"):
            try:
                # collapse punctuation
                clean = line.replace("info from server:", "").replace(";", " ")
                tokens = clean.split()
                for i,t in enumerate(tokens):
                    if t == "processed:":
                        out["processed"] = int(tokens[i+1])
                    if t == "failed:":
                        out["failed"] = int(tokens[i+1])
            except Exception:
                pass
    return out

def safe_filename(s):
    return "".join(c if c.isalnum() or c in "._" else "_" for c in s)

def probe_targets(targets, port, host_guesses, keys, delay, proxy):
    binpath = which_zabbix_sender()
    if not binpath:
        return 1
    found_file = Path("found_trapper.txt")
    with found_file.open("a", encoding="utf-8") as fh:
        for target in targets:
            timestamp = datetime.utcnow().strftime("%Y%m%dT%H%M%SZ")
            logfile = OUT_DIR / f"{safe_filename(target)}_{timestamp}.log"
            print(f"[+] Target {target} port {port}")
            for host in host_guesses:
                for key in keys:
                    print(f"  -> try host {host} key {key}", end=" ", flush=True)
                    out, dur = run_sender(binpath, target, port, host, key, timeout=10, proxy=proxy)
                    parsed = parse_sender_output(out)
                    # write raw output to per target log
                    with logfile.open("a", encoding="utf-8") as lf:
                        lf.write(f"### {datetime.utcnow().isoformat()} host={host} key={key} duration={dur:.2f}s\n")
                        lf.write(out + "\n\n")
                    # determine success heuristics
                    success = False
                    # prefer processed/failed if available
                    if parsed.get("processed") is not None and parsed.get("failed") is not None:
                        if parsed["processed"] > 0 and parsed["failed"] == 0:
                            success = True
                    # fallback to sent and failed
                    elif parsed.get("sent") is not None and parsed.get("failed") is not None:
                        if parsed["sent"] > 0 and parsed["failed"] == 0:
                            success = True
                    # print short summary
                    print(f"(duration {dur:.2f}s) parsed={parsed} {'SUCCESS' if success else 'no'}")
                    if success:
                        entry = f"{datetime.utcnow().isoformat()} target={target} port={port} host={host} key={key} parsed={parsed}\n"
                        fh.write(entry)
                    # gentle delay between attempts
                    time.sleep(delay)
    print("[*] Done. See found_trapper.txt and logs/ for raw outputs.")
    return 0

def main():
    parser = argparse.ArgumentParser(description="Probe Zabbix trapper using zabbix_sender")
    parser.add_argument("--targets", nargs="+", required=True, help="one or more target IPs or hostnames")
    parser.add_argument("--port", type=int, default=DEFAULT_PORT, help="target trapper port (default 10051)")
    parser.add_argument("--hosts-file", help="file with hostnames to try one per line")
    parser.add_argument("--keys-file", help="file with item keys to try one per line")
    parser.add_argument("--delay", type=float, default=0.5, help="delay seconds between attempts")
    parser.add_argument("--proxy", help="HTTP proxy for debugging (http://127.0.0.1:8080)")
    parser.add_argument("--extra-hosts", nargs="*", help="additional host guesses")
    parser.add_argument("--extra-keys", nargs="*", help="additional keys to try")
    args = parser.parse_args()

    host_guesses = list(DEFAULT_HOST_GUESSES)
    keys = list(DEFAULT_KEYS)

    if args.hosts_file:
        p = Path(args.hosts_file)
        if p.exists():
            host_guesses = [line.strip() for line in p.read_text(encoding="utf-8").splitlines() if line.strip()]
    if args.keys_file:
        p = Path(args.keys_file)
        if p.exists():
            keys = [line.strip() for line in p.read_text(encoding="utf-8").splitlines() if line.strip()]
    if args.extra_hosts:
        host_guesses = args.extra_hosts + host_guesses
    if args.extra_keys:
        keys = args.extra_keys + keys

    # final run
    return_code = probe_targets(args.targets, args.port, host_guesses, keys, args.delay, args.proxy)
    sys.exit(return_code)

if __name__ == "__main__":
    main()

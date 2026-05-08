
>[!info]
>This script is a small wrapper that makes Masscan easier and safer to use for initial reconnaissance. It provides sensible defaults, clear long‑form flags, and simple port profiles so you can quickly identify interesting hosts and services before handing results off to Nmap or AutoRecon for deeper enumeration.

 >[!warning]+ Requirements
> masscan and nmap must be installed and available in PATH.
> Nmap is used to generate Top 100 / Top 1000 port lists for Masscan.

```
masscan_wrapper_recon.sh
```

```
#!/usr/bin/env bash
set -e

# Defaults (no default subnet on purpose)
SUBNET=""
OUTDIR="scans/masscan"
RATE=300
OPEN_ONLY=true
PORT_PROFILE="top100"

UDP_SCAN=false
UDP_PROFILE="top20"

usage() {
  cat << EOF
Usage: $0 --subnet <HOST|CIDR> [OPTIONS]

Required:
  --subnet <HOST|CIDR>      Target host or subnet (required)

Options:
  --rate <number>           Masscan rate (default: 300)
  --open-only <true|false>  Show only open ports (default: true)
  --ports <profile>         TCP ports: top100 | top1000 | full
                            (default: top100)
  --udp <true|false>        Enable UDP scanning (default: false)
  --udp-ports <profile>    UDP ports: top20 | top50
                            (default: top20)
  --outdir <path>           Output directory (default: scans/masscan)
  --help                    Show this help

Notes:
  UDP scanning is best-effort discovery only.
  Results must be validated with Nmap.
EOF
}

# Parse arguments
while [[ $# -gt 0 ]]; do
  case "$1" in
    --subnet) SUBNET="$2"; shift 2 ;;
    --rate) RATE="$2"; shift 2 ;;
    --open-only) OPEN_ONLY="$2"; shift 2 ;;
    --ports) PORT_PROFILE="$2"; shift 2 ;;
    --udp) UDP_SCAN="$2"; shift 2 ;;
    --udp-ports) UDP_PROFILE="$2"; shift 2 ;;
    --outdir) OUTDIR="$2"; shift 2 ;;
    --help) usage; exit 0 ;;
    *)
      echo "Unknown option: $1"
      usage
      exit 1
      ;;
  esac
done

# Enforce required subnet
if [[ -z "$SUBNET" ]]; then
  echo "ERROR: --subnet is required"
  usage
  exit 1
fi

# Requirements
for cmd in nmap masscan; do
  if ! command -v "$cmd" >/dev/null 2>&1; then
    echo "ERROR: $cmd is required but not found in PATH"
    exit 1
  fi
done

mkdir -p "$OUTDIR"

SAFE_SUBNET=$(echo "$SUBNET" | tr '/.' '_')
OUTBASE="${OUTDIR}/${SAFE_SUBNET}"

OPEN_FLAG=""
[[ "$OPEN_ONLY" == "true" ]] && OPEN_FLAG="--open-only"

# TCP ports
case "$PORT_PROFILE" in
  top100)
    TCP_PORTS="$(nmap --top-ports 100 -p- 2>/dev/null \
      | sed -n 's/.*Ports: //p' \
      | tr ',' '\n' \
      | cut -d'/' -f1 \
      | tr '\n' ',' \
      | sed 's/,$//')"
    ;;
  top1000)
    TCP_PORTS="$(nmap --top-ports 1000 -p- 2>/dev/null \
      | sed -n 's/.*Ports: //p' \
      | tr ',' '\n' \
      | cut -d'/' -f1 \
      | tr '\n' ',' \
      | sed 's/,$//')"
    ;;
  full)
    TCP_PORTS="1-65535"
    ;;
  *)
    echo "Invalid TCP port profile"
    exit 1
    ;;
esac

echo "[*] Running TCP scan"
sudo masscan "$SUBNET" -p"$TCP_PORTS" \
  --rate "$RATE" \
  $OPEN_FLAG \
  -oG "${OUTBASE}_tcp.grep" \
  -oJ "${OUTBASE}_tcp.json" \
  -oL "${OUTBASE}_tcp.list" \
  -oB "${OUTBASE}_tcp.bin"

# UDP scan (optional)
if [[ "$UDP_SCAN" == "true" ]]; then
  case "$UDP_PROFILE" in
    top20)
      UDP_PORTS="53,67,68,69,123,137,138,161,162,500,514,520,623,1194,1900,4500,5353,11211,1701,1812"
      ;;
    top50)
      UDP_PORTS="53,67,68,69,123,137,138,161,162,500,514,520,623,1194,1900,4500,5353,11211,1701,1812,2049,5060,3478,1645,1646,4444,9876,9999"
      ;;
    *)
      echo "Invalid UDP port profile"
      exit 1
      ;;
  esac

  echo "[*] Running UDP scan"
  sudo masscan "$SUBNET" -pU:"$UDP_PORTS" \
    --rate "$RATE" \
    $OPEN_FLAG \
    -oG "${OUTBASE}_udp.grep" \
    -oJ "${OUTBASE}_udp.json" \
    -oL "${OUTBASE}_udp.list" \
    -oB "${OUTBASE}_udp.bin"
fi
```
# Usage

```
# Basic TCP recon (Top 100 ports, open-only)
./masscan_recon.sh --subnet 10.10.0.0/16

# Faster TCP recon
./masscan_recon.sh --subnet 10.10.0.0/16 --rate 1000

# TCP Top 1000 ports
./masscan_recon.sh --subnet 10.10.0.0/16 --ports top1000

# Full TCP port scan
./masscan_recon.sh --subnet 10.10.0.0/16 --ports full

# Include closed TCP ports
./masscan_recon.sh --subnet 10.10.0.0/16 --open-only false

# Single host scan
./masscan_recon.sh --subnet 10.10.10.5

# Custom output directory
./masscan_recon.sh --subnet 10.10.0.0/16 --outdir scans/internal

# TCP + UDP (Top 20 UDP ports)
./masscan_recon.sh --subnet 10.10.0.0/16 --udp true

# TCP + expanded UDP discovery
./masscan_recon.sh --subnet 10.10.0.0/16 --udp true --udp-ports top50

# Slower, quieter TCP + UDP scan
./masscan_recon.sh --subnet 10.10.0.0/16 --rate 150 --udp true

# Fully explicit configuration
./masscan_recon.sh \
  --subnet 10.10.0.0/16 \
  --rate 500 \
  --ports top1000 \
  --open-only true \
  --udp true \
  --udp-ports top50 \
  --outdir scans/custom
```


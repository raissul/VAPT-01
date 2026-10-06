# Automated Web VAPT: ONUS + KP WebScanner

This guide installs and runs two automated web VAPT tools on Linux:

1. **ONUS** — dashboard + deterministic CVSS v3.1 scoring + PDF report.
2. **KP WebScanner** — Docker-based 18-step pipeline + consolidated HTML report.

> **Authorization:** Only scan websites/assets that you own or have explicit written permission to test. Both tools perform active security testing; KP WebScanner includes SQL injection, XSS, fuzzing, Nmap/NSE and OWASP ZAP active scanning.

---

## 1. Recommended environment

These instructions assume a Debian/Ubuntu-style Linux host.

### Minimum practical resources

For **ONUS**:

- Docker + Docker Compose v2
- ~8 GB free disk
- ~6 GB free RAM
- First build can take roughly 10–15 minutes

For **KP WebScanner**:

- Docker or Podman
- `--network host`
- `NET_ADMIN` and `NET_RAW` capabilities
- More CPU/RAM makes the full scan noticeably faster

ONUS bundles its scanning tools and services in Docker. KP WebScanner is also self-contained, so you do not need to install Nmap, Nuclei, ZAP, sqlmap, etc. separately.

---

# 2. Install Docker

Check whether Docker already exists:

```bash
docker --version
docker compose version
```

If both work, skip to section 3.

## Debian/Ubuntu

```bash
sudo apt update
sudo apt install -y ca-certificates curl git
```

Install Docker using Docker's official convenience script:

```bash
curl -fsSL https://get.docker.com | sudo sh
```

Start Docker:

```bash
sudo systemctl enable --now docker
```

Verify:

```bash
docker --version
docker compose version
sudo docker run --rm hello-world
```

### Optional: run Docker without sudo

```bash
sudo usermod -aG docker "$USER"
```

Then log out and log back in.

Verify:

```bash
docker run --rm hello-world
```

If this works, you can use `docker` without `sudo`.

---

# 3. Create a VAPT workspace

Keep both projects and their results organized:

```bash
mkdir -p ~/VAPT
cd ~/VAPT
```

Suggested structure:

```text
~/VAPT/
├── ONUS/
├── webscanner/
└── results/
    ├── onus/
    └── webscanner/
```

Create the results directories:

```bash
mkdir -p ~/VAPT/results/onus
mkdir -p ~/VAPT/results/webscanner
```

---

# 4. Tool 1 — ONUS

Repository:

```text
https://github.com/maverickaayush/ONUS
```

ONUS provides:

- Web dashboard
- 8 parallel scanning modules
- Nmap/recon
- OWASP ZAP
- Nikto
- Katana
- Nuclei
- FFUF
- SSL/TLS checks
- HTTP security headers
- OWASP checks
- Technology/WAF fingerprinting
- Deterministic CVSS v3.1 scoring
- Confidence verification
- PDF report
- Optional local AI analysis through Ollama

---

## 4.1 Clone ONUS

```bash
cd ~/VAPT
git clone https://github.com/maverickaayush/ONUS.git ONUS
cd ONUS
```

Check the repository:

```bash
git status
```

---

## 4.2 Configure ONUS

Copy the environment file:

```bash
cp .env.example .env
```

Copy the Subfinder provider configuration:

```bash
cp backend/subfinder-config/provider-config.yaml.example \
   backend/subfinder-config/provider-config.yaml
```

The provider configuration can remain empty.

You do **not** need API keys for the basic ONUS workflow.

Optional free-tier keys can later improve subdomain enumeration:

- GitHub token
- ProjectDiscovery Chaos API key

---

## 4.3 Start ONUS

From the ONUS directory:

```bash
docker compose up -d
```

Check containers:

```bash
docker compose ps
```

Watch startup logs:

```bash
docker compose logs -f
```

You can also specifically watch the backend and worker:

```bash
docker compose logs -f backend worker
```

Wait until the ZAP service reports healthy.

---

## 4.4 Open the ONUS dashboard

Open:

```text
http://localhost:3000
```

ONUS is self-hosted and normally does not require account creation or sign-in.

The basic workflow is:

```text
Open dashboard
      ↓
Enter authorized target domain
      ↓
Confirm authorization checkbox
      ↓
Start scan
      ↓
Wait for scan modules
      ↓
Review findings
      ↓
Download PDF
```

Example target format:

```text
https://your-authorized-domain.com
```

Use your own domain or an intentionally vulnerable lab target for testing.

---

# 5. ONUS final result

ONUS provides two main results:

### A. Live dashboard

Open:

```text
http://localhost:3000
```

You can see:

- Scan progress
- Findings
- Severity
- CVSS score
- OWASP category
- Verification/confidence
- Remediation information

### B. PDF report

The ONUS dashboard provides the report download once the scan is complete.

The report is generated through WeasyPrint and uses the same scored findings shown by the dashboard.

The report is intended to contain:

- Executive-level summary
- Finding details
- Severity
- CVSS v3.1
- OWASP mapping
- Evidence
- Remediation
- Technical information

---

# 6. Optional — ONUS AI descriptions

AI is **optional**.

ONUS works without Ollama. Without Ollama, findings still receive deterministic severity/CVSS scoring and rule-based descriptions.

If you want local AI-generated explanations/remediation, install Ollama on the host.

Install:

```bash
curl -fsSL https://ollama.com/install.sh | sh
```

Pull the model:

```bash
ollama pull qwen2.5:7b
```

Verify Ollama:

```bash
curl http://localhost:11434/api/tags
```

Because ONUS runs inside Docker, make Ollama reachable from the Docker network.

Create a systemd override:

```bash
sudo systemctl edit ollama
```

Add:

```ini
[Service]
Environment="OLLAMA_HOST=0.0.0.0:11434"
```

Then:

```bash
sudo systemctl daemon-reload
sudo systemctl restart ollama
```

Verify again:

```bash
curl http://localhost:11434/api/tags
```

Restart ONUS backend/worker:

```bash
cd ~/VAPT/ONUS
docker compose restart backend worker
```

Now ONUS can use the local Qwen model for plain-English finding descriptions and remediation guidance.

> If you do not need AI-generated text, skip this entire section.

---

# 7. ONUS — useful commands

### Start

```bash
cd ~/VAPT/ONUS
docker compose up -d
```

### Stop

```bash
cd ~/VAPT/ONUS
docker compose down
```

### Restart

```bash
cd ~/VAPT/ONUS
docker compose restart
```

### View status

```bash
docker compose ps
```

### View logs

```bash
docker compose logs -f backend worker
```

### Backend API documentation

```text
http://localhost:8000/docs
```

---

# 8. Tool 2 — KP WebScanner

Repository:

```text
https://github.com/kpirnie/webscanner
```

KP WebScanner is a much more aggressive all-in-one scanner.

Its pipeline includes:

1. Fingerprinting
2. Passive recon
3. Subdomain enumeration
4. DNS enumeration
5. Port scanning
6. Nmap service fingerprinting
7. SSL/TLS analysis
8. HTTP security headers
9. Nikto
10. CMS scanning
11. Endpoint discovery
12. Parameter discovery
13. XSS scanning
14. SQL injection testing
15. Dependency scanning
16. Nuclei
17. Bot-blocker validation
18. OWASP ZAP active scanning

It can automatically generate a consolidated `report.html`.

---

# 9. KP WebScanner — easiest method

You do **not** need to clone the repository if you only want to run the released Docker image.

Pull the image:

```bash
docker pull ghcr.io/kpirnie/webscanner:latest
```

Verify:

```bash
docker images | grep webscanner
```

---

# 10. Run a quick KP WebScanner scan

Replace the target with an authorized website:

```bash
docker run --rm \
  --network host \
  --cap-add NET_ADMIN \
  --cap-add NET_RAW \
  ghcr.io/kpirnie/webscanner:latest \
  https://your-authorized-domain.com
```

This prints a condensed summary to the terminal.

However, for actual VAPT work, use the output directory method below.

---

# 11. Run a full KP WebScanner scan + save results

Create a result directory:

```bash
mkdir -p ~/VAPT/results/webscanner
```

Run:

```bash
cd ~/VAPT

docker run --rm \
  --network host \
  --cap-add NET_ADMIN \
  --cap-add NET_RAW \
  -v "$PWD/results/webscanner:/output" \
  ghcr.io/kpirnie/webscanner:latest \
  https://your-authorized-domain.com \
  -o results
```

The scan will execute the pipeline and write the results to the mounted host directory.

---

# 12. Find the final KP WebScanner report

After the scan:

```bash
find ~/VAPT/results/webscanner -maxdepth 2 -type f -name "report.html" -print
```

You should get something similar to:

```text
~/VAPT/results/webscanner/your-authorized-domain.com_YYYYMMDD_HHMMSS/report.html
```

Open the report with your browser:

```bash
xdg-open "$(find ~/VAPT/results/webscanner \
  -maxdepth 2 \
  -type f \
  -name report.html \
  | sort \
  | tail -n 1)"
```

The main consolidated report is:

```text
report.html
```

---

# 13. KP WebScanner output structure

A completed scan creates a timestamped directory similar to:

```text
results/
└── your-authorized-domain.com_YYYYMMDD_HHMMSS/
    ├── scan.log
    ├── report.html
    ├── whatweb.txt
    ├── httpx.txt
    ├── subdomains.txt
    ├── subdomains_live.txt
    ├── dns.txt
    ├── ports.txt
    ├── nmap.txt
    ├── nmap.xml
    ├── testssl.txt
    ├── testssl.json
    ├── observatory.json
    ├── nikto.txt
    ├── nikto.json
    ├── endpoints.txt
    ├── gobuster.txt
    ├── ffuf.json
    ├── arjun.json
    ├── dalfox.txt
    ├── sqlmap/
    ├── nuclei.txt
    ├── nuclei.json
    ├── botblocker_test.txt
    └── zap_report.html
```

The important files are:

```text
report.html
zap_report.html
nmap.txt
nuclei.txt
sqlmap/
dalfox.txt
testssl.txt
nikto.txt
scan.log
```

`report.html` is the consolidated report.

---

# 14. KP WebScanner — fast scan

For a faster first-pass assessment:

```bash
cd ~/VAPT

docker run --rm \
  --network host \
  --cap-add NET_ADMIN \
  --cap-add NET_RAW \
  -v "$PWD/results/webscanner:/output" \
  ghcr.io/kpirnie/webscanner:latest \
  https://your-authorized-domain.com \
  -o results \
  --skip-zap \
  --skip-brute \
  --skip-arjun \
  --severity high,critical
```

This skips some of the slower/aggressive steps.

---

# 15. KP WebScanner — recon-only scan

For a lower-impact initial assessment:

```bash
cd ~/VAPT

docker run --rm \
  --network host \
  --cap-add NET_ADMIN \
  --cap-add NET_RAW \
  -v "$PWD/results/webscanner:/output" \
  ghcr.io/kpirnie/webscanner:latest \
  https://your-authorized-domain.com \
  -o results \
  --skip-nikto \
  --skip-xss \
  --skip-sqlmap \
  --skip-nuclei \
  --skip-zap
```

This is useful before moving to active vulnerability testing.

---

# 16. Optional API keys for KP WebScanner

The basic scan works without API keys.

Optional services:

### WPScan

A free API token can improve WordPress vulnerability data.

Run:

```bash
docker run --rm \
  --network host \
  --cap-add NET_ADMIN \
  --cap-add NET_RAW \
  -e WPSCAN_API_TOKEN="YOUR_TOKEN" \
  -v "$PWD/results/webscanner:/output" \
  ghcr.io/kpirnie/webscanner:latest \
  https://your-authorized-domain.com \
  -o results
```

### Shodan

Optional:

```bash
-e SHODAN_API_KEY="YOUR_KEY"
```

### Censys

Optional:

```bash
-e CENSYS_APP_ID="YOUR_APP_ID" \
-e CENSYS_TOKEN="YOUR_TOKEN"
```

Do not put API keys directly into scripts that you plan to commit to Git.

---

# 17. Build KP WebScanner locally instead

If you want to modify the scanner:

```bash
cd ~/VAPT

git clone https://github.com/kpirnie/webscanner.git webscanner

cd webscanner

docker build -t webscanner .
```

First build may take approximately 10–20 minutes because the image compiles/installs many security tools.

Then run:

```bash
cd ~/VAPT

docker run --rm \
  --network host \
  --cap-add NET_ADMIN \
  --cap-add NET_RAW \
  -v "$PWD/results/webscanner:/output" \
  webscanner \
  https://your-authorized-domain.com \
  -o results
```

---

# 18. Which one should I run first?

For a new target, I recommend:

```text
               Authorized Target
                      │
                      ▼
             ┌─────────────────┐
             │     ONUS         │
             │ Dashboard + PDF  │
             └────────┬────────┘
                      │
                      ▼
             ┌─────────────────┐
             │ KP WebScanner    │
             │ Deep assessment  │
             └────────┬────────┘
                      │
                      ▼
              Compare findings
                      │
                      ▼
             Manual verification
```

### ONUS

Best for:

- Clean dashboard
- CVSS
- OWASP mapping
- Finding verification
- Executive-style PDF
- Easier review

### KP WebScanner

Best for:

- Broad automated coverage
- Raw evidence
- Recon
- Endpoint discovery
- XSS
- SQLi
- Nuclei
- ZAP
- Nmap
- SSL/TLS
- CMS checks

---

# 19. Recommended workflow

For a website you are authorized to test:

## Phase 1 — ONUS

```bash
cd ~/VAPT/ONUS
docker compose up -d
```

Open:

```text
http://localhost:3000
```

Run the target.

Download the PDF report.

---

## Phase 2 — KP WebScanner

```bash
cd ~/VAPT

docker run --rm \
  --network host \
  --cap-add NET_ADMIN \
  --cap-add NET_RAW \
  -v "$PWD/results/webscanner:/output" \
  ghcr.io/kpirnie/webscanner:latest \
  https://your-authorized-domain.com \
  -o results
```

Open:

```text
~/VAPT/results/webscanner/<timestamp>/report.html
```

---

# 20. Compare the results

Do not blindly treat every automated finding as a confirmed vulnerability.

Create a simple verification table:

| Finding | ONUS | WebScanner | Manually verified | Final status |
|---|---:|---:|---:|---|
| Missing CSP | Yes | Yes | Yes | Confirmed |
| TLS issue | Yes | Yes | Yes | Confirmed |
| Reflected XSS | No | Yes | Pending | Needs verification |
| SQLi | No | Yes | Pending | Needs verification |
| Exposed service | Yes | Yes | Yes | Confirmed |
| Nuclei finding | Yes | Yes | Pending | Needs verification |

This is especially important for SQLi, XSS, CVEs and scanner-generated high/critical findings.

---

# 21. Important: active scanning can affect production

KP WebScanner performs active operations including:

- SQL injection testing
- XSS testing
- Directory/path fuzzing
- Parameter discovery
- Nmap/NSE
- Nuclei checks
- OWASP ZAP active scanning

Therefore:

### Do not start with the full scan against an unknown production application.

Prefer:

```text
Recon
  ↓
Basic/passive checks
  ↓
Review scope
  ↓
Active scan
  ↓
Manual verification
```

For a production target, agree on:

- Allowed domains/subdomains
- Allowed IP ranges
- Testing window
- Rate limits
- Authentication accounts, if applicable
- Out-of-scope endpoints
- Whether SQLi/XSS/active testing is permitted
- Whether DoS/stress testing is prohibited

---

# 22. Safe lab targets

If you want to test the installation before touching a real website, use intentionally vulnerable applications such as:

- OWASP Juice Shop
- DVWA
- WebGoat
- bWAPP
- Mutillidae
- NodeGoat

ONUS includes an optional Docker Compose profile for intentionally vulnerable practice targets.

Start the ONUS practice targets:

```bash
cd ~/VAPT/ONUS
docker compose --profile targets up -d
```

Then inspect the published ports:

```text
Juice Shop  :3001
DVWA        :8081
bWAPP       :8083
Mutillidae  :8084
NodeGoat    :8085
WebGoat      :8082
```

Only expose these lab services to a trusted/local network.

---

# 23. Troubleshooting

## ONUS containers are not healthy

Check:

```bash
cd ~/VAPT/ONUS
docker compose ps
```

Then:

```bash
docker compose logs -f
```

For scanner/backend problems:

```bash
docker compose logs -f backend worker
```

---

## ONUS cannot start because of resources

Check:

```bash
free -h
df -h
docker system df
```

ONUS can consume several GB of RAM/disk because it runs multiple services and security tools.

---

## KP WebScanner cannot perform Nmap/Naabu

Make sure you used:

```bash
--network host
--cap-add NET_ADMIN
--cap-add NET_RAW
```

Example:

```bash
docker run --rm \
  --network host \
  --cap-add NET_ADMIN \
  --cap-add NET_RAW \
  ...
```

---

## KP WebScanner report is missing

Check the mounted output directory:

```bash
find ~/VAPT/results/webscanner -maxdepth 2 -type f | sort
```

Check:

```bash
find ~/VAPT/results/webscanner \
  -maxdepth 2 \
  -type f \
  -name "scan.log" \
  -print
```

Read the log:

```bash
less "$(find ~/VAPT/results/webscanner \
  -maxdepth 2 \
  -type f \
  -name scan.log \
  | sort \
  | tail -n 1)"
```

---

# 24. One-command KP WebScanner helper

You can create a small helper script:

```bash
nano ~/VAPT/run-webscanner.sh
```

Paste:

```bash
#!/usr/bin/env bash

set -euo pipefail

TARGET="${1:-}"

if [[ -z "$TARGET" ]]; then
    echo "Usage: $0 https://authorized-target.example"
    exit 1
fi

BASE="$HOME/VAPT"
OUTPUT="$BASE/results/webscanner"

mkdir -p "$OUTPUT"

echo "[+] Target: $TARGET"
echo "[+] Output: $OUTPUT"
echo "[+] Starting KP WebScanner..."

docker run --rm \
    --network host \
    --cap-add NET_ADMIN \
    --cap-add NET_RAW \
    -v "$OUTPUT:/output" \
    ghcr.io/kpirnie/webscanner:latest \
    "$TARGET" \
    -o results

echo
echo "[+] Scan finished."
echo "[+] Reports:"
find "$OUTPUT" -maxdepth 2 -type f \
    \( -name "report.html" -o -name "zap_report.html" \) \
    -print
```

Make executable:

```bash
chmod +x ~/VAPT/run-webscanner.sh
```

Run:

```bash
~/VAPT/run-webscanner.sh https://your-authorized-domain.com
```

---

# 25. Final directory layout

After using both tools, your workspace can look like:

```text
~/VAPT/
│
├── ONUS/
│   ├── backend/
│   ├── frontend/
│   ├── docker-compose.yml
│   └── ...
│
├── webscanner/
│   ├── Dockerfile
│   ├── scan.sh
│   └── ...
│
├── run-webscanner.sh
│
└── results/
    │
    ├── onus/
    │
    └── webscanner/
        └── target.example.com_YYYYMMDD_HHMMSS/
            ├── report.html
            ├── zap_report.html
            ├── scan.log
            ├── nmap.txt
            ├── nuclei.txt
            ├── nikto.txt
            ├── dalfox.txt
            ├── testssl.txt
            ├── endpoints.txt
            └── ...
```

---

# 26. Quick command cheat sheet

## ONUS

```bash
cd ~/VAPT
git clone https://github.com/maverickaayush/ONUS.git ONUS

cd ONUS

cp .env.example .env

cp backend/subfinder-config/provider-config.yaml.example \
   backend/subfinder-config/provider-config.yaml

docker compose up -d

docker compose ps
```

Open:

```text
http://localhost:3000
```

Stop:

```bash
docker compose down
```

---

## KP WebScanner

```bash
docker pull ghcr.io/kpirnie/webscanner:latest
```

Full scan:

```bash
cd ~/VAPT

docker run --rm \
  --network host \
  --cap-add NET_ADMIN \
  --cap-add NET_RAW \
  -v "$PWD/results/webscanner:/output" \
  ghcr.io/kpirnie/webscanner:latest \
  https://your-authorized-domain.com \
  -o results
```

Find report:

```bash
find ~/VAPT/results/webscanner \
  -maxdepth 2 \
  -type f \
  -name "report.html" \
  -print
```

Open latest report:

```bash
xdg-open "$(find ~/VAPT/results/webscanner \
  -maxdepth 2 \
  -type f \
  -name report.html \
  | sort \
  | tail -n 1)"
```

---

# 27. Expected final deliverables

After running both tools, you should have:

### ONUS

```text
PDF VAPT Report
+
Web Dashboard
```

### KP WebScanner

```text
report.html
+
zap_report.html
+
Raw scanner evidence
```

### Combined assessment

Use the ONUS PDF as the cleaner management/client-facing report and KP WebScanner's raw outputs as technical evidence for deeper verification.

---

## Official repositories

ONUS:
https://github.com/maverickaayush/ONUS

KP WebScanner:
https://github.com/kpirnie/webscanner

Both repositories should be checked before each deployment for changed commands, dependencies, Docker image tags, and scanner behavior.

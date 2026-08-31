# 🛡️ SSH Honeypot Comparative Performance & Evaluation Benchmark

An enterprise-grade, containerized benchmarking suite to evaluate, stress-test, and compare the telemetry, resource utilization, and threat intelligence fidelity of **Cowrie**, **Kippo**, and **OpenCanary**.

This repository delivers an end-to-end evaluation harness designed for security researchers, DevSecOps teams, and SOC analysts. By coupling medium and low-interaction honeypots with an automated Kali Linux testing container on an isolated bridge network, this framework enables reproducible benchmarks across hardware resource overhead (CPU/RAM), connection response latency, credential attack resilience, and interactive session logging fidelity.

---

## 🚀 Key Highlights

* **Unified Docker Orchestration:** Deploy Cowrie, Kippo, OpenCanary, and Kali Linux simultaneously with zero host port collisions and direct inter-container DNS discovery.
* **Comprehensive Attack Playbooks:** Pre-packaged Nmap Scripting Engine (NSE) brute-force workflows, high-throughput connection stress loops, and interactive command execution simulations.
* **Deterministic Performance Profiling:** Precision 30-second sliding-window scripts to measure baseline idle vs. attack-induced CPU and memory footprints.
* **Forensic Log Analysis:** Structured evaluation of SIEM-ready JSON output, connection attempt tracking, and interactive shell session reconstruction.

---

## 🏗️ Architecture & Network Topology

All containers are attached to a dedicated Docker user-defined bridge network (`honeynet`). This allows the Kali attack container to target each honeypot using internal service names over standard port `2222`, while exposing designated ports to the host for external inspection.

```text
+-----------------------------------------------------------------------------------+
| Host System (Docker Engine)                                                       |
|                                                                                   |
|  [ Isolated Bridge Network: honeynet ]                                            |
|                                                                                   |
|   +-----------------------+         +-------------------------------------------+ |
|   |  kali_tester          |         |  cowrie (Medium/High Interaction)         | |
|   |  (Nmap / SSH Suite)   | =======> |  - Internal: 2222 | Host Port: 2222       | |
|   +-----------------------+         |  - Logs: JSON & Malware TTY Dumps         | |
|               ||                    +-------------------------------------------+ |
|               ||                                                                  |
|               ||                    +-------------------------------------------+ |
|               ||                    |  kippo (Medium Interaction / Legacy)      | |
|               +===================> |  - Internal: 2222 | Host Port: 2223       | |
|               ||                    |  - Logs: Container Stdout / TTY Logs      | |
|               ||                    +-------------------------------------------+ |
|               ||                                                                  |
|               ||                    +-------------------------------------------+ |
|               ||                    |  opencanary_latest (Low Interaction)      | |
|               +===================> |  - Internal: 2222 | Host Port: 2224       | |
|                                     |  - Logs: Fast Alert Daemon (4002 Event)   | |
|                                     +-------------------------------------------+ |
+-----------------------------------------------------------------------------------+
```

---

## 📊 Honeypot Feature & Architecture Comparison

| Feature / Capability | Cowrie | Kippo | OpenCanary |
| :--- | :--- | :--- | :--- |
| **Interaction Level** | Medium to High | Medium | Low to Medium (Tripwire) |
| **Target Use Case** | In-depth malware analysis, attacker shell recording | Legacy SSH interaction research | Rapid intrusion detection and alert tripwire |
| **Modern SSH Cipher Support** | Full (OpenSSH, modern KEX & MACs) | Legacy Only (requires `diffie-hellman-group1-sha1`) | Standard OpenSSH Handshake Emulation |
| **TTY Shell Simulation** | Advanced fake Debian filesystem & command parser | Simulated filesystem & command execution | Authentication rejection only (No TTY) |
| **Payload / Artifact Retention** | Automated file capture via `wget` / `curl` / `sftp` | Basic simulated download capture | None (Telemetry only) |
| **Structured Logging** | JSON logs (`cowrie.json`), Syslog, Splunk, ELK | Plain text container stdout | JSON log events (`logtype: 4002`) |

---

## 📁 Repository Layout

```text
.
├── docker-compose.yml             # Unified multi-service orchestration (Honeypots + Kali)
├── Dockerfile.kali                # Hardened Kali Linux test container build recipe
├── data/
│   └── .opencanary.conf           # OpenCanary service configuration and alert toggles
└── kali_data/
    ├── users.txt                  # Targeted username dictionary
    └── small_passlist.txt         # Common brute-force password dictionary
```

---

## 📈 Empirical Benchmark & Performance Results

| Benchmark Metric | Cowrie | Kippo | OpenCanary |
| :--- | :--- | :--- | :--- |
| **Idle CPU Consumption** | 0.00% | 0.00% | 0.05% |
| **Stress Attack CPU Load** | 37.26% | 35.87% | 25.19% |
| **Idle Memory Footprint** | 52.48 MB | 40.96 MB | 52.22 MB |
| **Stress Attack Memory Load** | 53.27 MB | 41.33 MB | 52.57 MB |
| **Port Response Latency** | ~0.002s – 0.007s | ~0.004s – 0.011s | ~0.013s |
| **Log Growth (per 500 attempts)** | ~1.48 MB (JSON) | ~1.18 MB (Docker log) | ~0.58 MB (File log) |
| **Interactive Commands Recorded** | 100% captured (9/9 commands) | 100% captured (9/9 commands) | N/A (No shell emulation) |
| **Container Restarts Under Stress** | 0 (Stable) | 0 (Stable) | 0 (Stable) |

---

## 🧪 Reproducible Testing & Benchmark Workflows

### Step 1: Build and Launch the Testbed

```bash
# Build and launch all services in detached mode
docker compose up -d --build

# Verify container health status
docker compose ps
```

### Step 2: Enter the Kali Linux Attack Environment

```bash
docker exec -it kali_tester bash
```

### Step 3: Execute Attack Simulations (Inside Kali)

**Targeted Nmap Brute-Force Authentication Attack:**

```bash
# Test Cowrie
nmap -p 2222 --script ssh-brute --script-args userdb=users.txt,passdb=small_passlist.txt cowrie

# Test Kippo
nmap -p 2222 --script ssh-brute --script-args userdb=users.txt,passdb=small_passlist.txt kippo

# Test OpenCanary
nmap -p 2222 --script ssh-brute --script-args userdb=users.txt,passdb=small_passlist.txt opencanary_latest
```

**Continuous Connection Flooding Stress Test (60 Seconds):**

```bash
timeout 60 bash -lc 'while :; do ssh -o BatchMode=yes -o ConnectTimeout=1 -p 2222 cowrie exit >/dev/null 2>&1; done'
```

**Interactive Shell Session & Post-Exploitation Simulation:**

```bash
# Interactive login into Cowrie
ssh -p 2222 root@cowrie

# Interactive login into Kippo (utilizing legacy cipher negotiation)
ssh -p 2222 root@kippo -oKexAlgorithms=+diffie-hellman-group1-sha1 -oHostKeyAlgorithms=+ssh-rsa
```

### Step 4: Real-Time Host Metric Profiling (On Host Machine)

**Calculate 30-Second Average CPU Usage:**

```bash
CID="cowrie"
for i in $(seq 1 30); do docker stats --no-stream --format "{{.CPUPerc}}" "$CID" | tr -d '%'; sleep 1; done | awk '{sum+=$1} END {print "Avg CPU% over 30s:", sum/NR}'
```

**Calculate 30-Second Average Memory Usage (in MB):**

```bash
CID="cowrie"
for i in $(seq 1 30); do docker stats --no-stream --format "{{.MemUsage}}" "$CID" | awk -F' / ' '{print $1}' | awk '/GiB/ {gsub("GiB",""); print $1*1024*1024} /MiB/ {gsub("MiB",""); print $1*1.048576} /KiB/ {gsub("KiB",""); print $1/1024*1.048576} /B$/ {gsub("B",""); print $1/1024/1024*1.048576}'; sleep 1; done | awk '{sum+=$1} END {printf "Avg Memory used over 30s: %.2f MB
", sum/NR}'
```

**Measure TCP Port Connection Latency:**

```bash
PORT=2222
(time bash -lc "printf '' | nc -w 3 localhost $PORT >/dev/null") 2>&1 | tail -n 1
```

**Verify Forensic Command Logging:**

```bash
# Verify Cowrie captured commands
grep -a "CMD" ~/honeypots/cowrie/log/cowrie.json

# Verify Kippo captured commands
docker logs kippo | grep "CMD"
```

---

## 💡 Strategic Recommendations & Verdict

* **Select Cowrie when:** You require high-fidelity adversary intelligence, automated malware binary capture, realistic interactive Linux shell sandboxing, and direct SIEM ingestion via structured JSON.
* **Select OpenCanary when:** You require a lightweight, zero-maintenance network canary to detect internal lateral movement or reconnaissance with minimal resource utilization and sub-millisecond setup time.
* **Avoid Kippo for modern production honeynets:** Kippo has been superseded by Cowrie and requires legacy SSH cipher negotiation (`SHA-1` / `ssh-rsa`), which causes modern SSH scanners to drop connections prematurely.

---

## 🔒 Security & Isolation Notice

When deploying honeypots on internet-facing infrastructure:

* Ensure honeypot containers run with non-root privileges where applicable.
* Isolate container network bridges from production VPCs and critical internal subnets to mitigate container breakout or pivot risks.
* Rotate storage volumes and establish log retention policies to prevent disk exhaustion during large-scale automated brute-force campaigns.
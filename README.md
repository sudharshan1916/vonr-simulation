# VoNR Demo using Open5GS
### End-to-End Voice over New Radio Simulation
**Sudharshan Mothukuru (RP1141), NeWS Lab, IIT Hyderabad**

> **What this is:** A complete, working VoNR (Voice over 5G New Radio) simulation on a single Ubuntu machine. Two software phones make a real voice call over a fully simulated 5G Standalone network with IMS — no hardware, no SIM cards, no spectrum license.

---

## Quick Start (TL;DR)

```bash
./start_vonr.sh          # start everything (~3 min)
./vonr_call.sh           # make a VoNR call
./vonr_full_kpi_logs.sh  # measure call quality KPIs
./verify_vonr_complete.sh # verify 55 checks pass
./stop_vonr.sh           # stop before shutdown
```

---

## Table of Contents

- [Verified Results](#verified-results)
- [Architecture](#architecture)
- [Deviations from Original Proposal](#deviations-from-original-proposal)
- [Prerequisites](#prerequisites)
- [Step 1 — Install Dependencies](#step-1--install-dependencies)
- [Step 2 — Clone and Configure](#step-2--clone-and-configure)
- [Step 2.5 — Provision Subscribers](#step-25--provision-subscribers-required-not-automated-by-any-script)
- [Step 3 — Start the Stack](#step-3--start-the-stack)
- [Step 4 — Make a VoNR Call](#step-4--make-a-vonr-call)
- [Step 5 — Full KPI Measurement](#step-5--full-kpi-measurement)
- [Step 6 — Verify Everything](#step-6--verify-everything)
- [Step 7 — Stop Before Shutdown](#step-7--stop-before-shutdown)
- [Critical Bugs Fixed](#critical-bugs-fixed)
- [Troubleshooting](#troubleshooting)
- [Scripts Reference](#scripts-reference)
- [Subscriber Configuration](#subscriber-configuration)
- [References](#references)

---

## Verified Results

All results from live capture on May 5, 2026 — `vonr.pcap` (20-second call, Opus codec, ZMQ simulation).

### Call Quality (RTP)

| Metric | Stream 1 (UE→RTPEngine) | Stream 2 (RTPEngine→UE) | 3GPP Limit |
|--------|------------------------|------------------------|------------|
| Packets | 978 | 978 | — |
| Packet Loss | **0.0%** | **0.0%** | < 1% |
| Mean Delta | **19.988ms** | **19.988ms** | ~20ms (50pps) |
| Mean Jitter | **9.912ms** | **9.910ms** | < 50ms |
| Max Jitter | **10.604ms** | **10.601ms** | < 50ms |
| Codec | **Opus** | **Opus** | — |

### Call Timing

| Metric | Measured |
|--------|----------|
| Call Setup Time (INVITE → 200 OK) | **0.061 seconds** |
| Call Duration | **19.877 seconds** |
| IMS Registration Latency | **~318ms** |

### Verification

```
Total checks : 55
Passed       : 55
Failed       : 0
Warnings     : 0
VoNR stack is fully operational!
```

### SIP Call Flow (from pcap)

```
0.44s    →  200 OK  (initial registration)
20.00s   →  REGISTER
20.19s   ←  401 Unauthorized (challenge)
20.42s   →  REGISTER (with credentials)
          ←  200 OK (registered)
24.87s   →  INVITE  (UE1 calls UE2)
24.94s   ←  180 Ringing
24.94s   ←  200 OK  (UE2 answers)
25.16s   →  ACK
25.21s      RTP audio flows (Opus, 50 pps)
44.75s   →  BYE
44.76s   ←  200 OK  (call ended cleanly)
```

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Single Ubuntu Host                           │
│                                                                     │
│  ┌─────────────┐   ZMQ    ┌─────────────┐   N2/N3  ┌───────────┐    │
│  │  srsRAN UE  │◄────────►│ srsRAN gNB  │◄────────►│  Open5GS  │    │
│  │ linphonec   │          │ (ZMQ RF sim)│          │  5G Core  │    │
│  │ UE1: :5070  │          │ 172.22.0.37 │          │ AMF / SMF │    │
│  │ UE2: :5071  │          └─────────────┘          │ UPF / NRF │    │
│  │ 172.22.0.34 │                                   └─────┬─────┘    │
│  └──────┬──────┘                                         │          │
│         │ SIP over IMS APN                       ogstun2 │          │
│         │ 192.168.101.2                    192.168.101.1 │          │
│         ▼                                                │          │
│  ┌─────────────────────────────────┐                    │           │
│  │  Kamailio IMS  (172.22.0.x)     │◄───────────────────┘           │
│  │  P-CSCF :5060  →  I-CSCF :4060 │                                 │
│  │  I-CSCF        →  S-CSCF :6060 │                                 │ 
│  └──────┬──────────────┬───────────┘                                │
│         │ Diameter Cx  │ RTP relay                                  │
│         ▼              ▼                                            │
│  ┌──────────┐   ┌─────────────┐                                     │
│  │  pyHSS   │   │  RTPEngine  │  ← relays Opus audio packets        │
│  │  MySQL   │   │ 172.22.0.16 │                                     │
│  └──────────┘   └─────────────┘                                     │
│                                                                     │
│  Docker Network: 172.22.0.0/24    26 containers total               │
└─────────────────────────────────────────────────────────────────────┘
```

**Three-phase call flow:**

1. **5G Registration** — srsUE attaches to gNB (ZMQ) → AMF authenticates via 5G-AKA → SMF creates IMS PDU session → UE gets IP `192.168.101.2` on `ogstun2`
2. **IMS Registration** — linphonec sends `SIP REGISTER` → P-CSCF → I-CSCF queries pyHSS via Diameter UAR → S-CSCF challenges with 401 → UE responds with credentials → `200 OK`
3. **VoNR Call** — `SIP INVITE` → `180 Ringing` → `200 OK` → `ACK` → RTP audio (Opus, 50 pps, 20ms intervals) relayed via RTPEngine → `SIP BYE` → `200 OK`

---

## Deviations from Original Proposal

> Two tools were changed from the original proposal. Both are upgrades, not failures.

| Original Plan | What Was Used | Why |
|---|---|---|
| **UERANSIM** | **srsRAN (ZMQ)** | srsRAN simulates full L1/L2/L3 stack (HARQ, MAC, RLC, PDCP) — more realistic protocol behavior |
| **RTPProxy** | **RTPEngine** | RTPEngine is actively maintained and already integrated in docker_open5gs |
| **Wireshark** | **tshark + tcpdump** | Same libpcap engine, works headless in Docker; tshark gives scriptable KPI extraction |
| **IBCF** (`sa-vonr-ibcf-deploy.yaml`) | **No IBCF** (`sa-vonr-deploy.yaml`) | `ibcf`'s build fetches an ETSI codec spec that etsi.org now blocks (403); it's not on the VoNR call path anyway ([Bug 10](#bug-10--ibcf-build-fails-on-external-etsi-download-403-forbidden)) |

---

## Prerequisites

- **OS:** Ubuntu 22.04 LTS (tested), Ubuntu 20.04 works
- **Hardware:** 8-core CPU, 16GB RAM, 50GB disk
- **Network:** Internet for initial setup only

---

## Step 1 — Install Dependencies

```bash
# Docker
sudo apt update && sudo apt upgrade -y
sudo apt install -y docker.io docker-compose-plugin
sudo usermod -aG docker $USER
newgrp docker

# Verify Docker Compose V2 (must be v2.x)
docker compose version

# Network tools
sudo apt install -y tshark tcpdump net-tools linphone-cli

# Allow tcpdump without password (needed for RTP capture scripts)
echo "$USER ALL=(ALL) NOPASSWD: /usr/bin/tcpdump" | sudo tee /etc/sudoers.d/tcpdump
sudo chmod 440 /etc/sudoers.d/tcpdump

# Enable IP forwarding permanently
sudo sysctl -w net.ipv4.ip_forward=1
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
```

---

## Step 2 — Clone and Configure

```bash
# Clone the base stack
git clone https://github.com/herlesupreeth/docker_open5gs.git
cd docker_open5gs

# CRITICAL FIX — disable N5 QoS before first build
# Without this, SIP REGISTER returns 412 error
sed -i '/WITH_N5/s/^/#DISABLED /' pcscf/pcscf_init.sh

# CRITICAL FIX — srsUE reads apn from this file at runtime (see Bug 3)
sed -i 's/apn = internet/apn = ims/' srslte/ue_5g_zmq.conf

# CRITICAL FIX — set DOCKER_HOST_IP / *_ADVERTISE_IP in .env to THIS machine's
# real LAN IP (`ip -4 addr show`), not the placeholder in the file.

# Clone THIS repo (vonr-simulation) if you haven't already — its scripts/
# folder is what you actually need. (An earlier version of this README
# pointed at a separate "VoNR_Private" repo; that repo does not have a
# scripts/ folder and should not be used — ignore any reference to it.)
git clone https://github.com/sudharshan1916/vonr-simulation.git ~/vonr-simulation
cp ~/vonr-simulation/scripts/*.sh ~/
chmod +x ~/start_vonr.sh ~/vonr_call.sh ~/stop_vonr.sh
chmod +x ~/verify_vonr_complete.sh ~/vonr_full_kpi_logs.sh
```

> **Use `sa-vonr-deploy.yaml`, not `sa-vonr-ibcf-deploy.yaml`.** The `ibcf` image's
> build downloads an ETSI EVS codec spec (`wget www.etsi.org/...zip`) that now
> returns `403 Forbidden` from etsi.org — the build fails and nothing comes up.
> `ibcf` (voicemail/interconnect border function) is not part of the VoNR call
> path in the [Architecture](#architecture) diagram above, so skip it — after
> copying the scripts below, patch the compose file reference:
> ```bash
> sed -i 's/sa-vonr-ibcf-deploy.yaml/sa-vonr-deploy.yaml/' ~/start_vonr.sh
> ```
> If you *do* want `ibcf`, add `NET_ADMIN`/routing yourself and be ready to
> patch its Dockerfile to skip the EVS codec `wget` step.

### Get the pre-built images (don't let `docker compose up` try to pull them)

The images referenced in the compose files (`docker_open5gs`, `docker_kamailio`,
`docker_pyhss`, `docker_mysql`, `docker_srslte`, `docker_srsran`, ...) are not on
Docker Hub — they must be pulled from GHCR and locally re-tagged, or `up -d`
will fail with `pull access denied`:

```bash
for img in docker_open5gs docker_grafana docker_metrics docker_pyhss \
           docker_kamailio docker_mysql docker_srslte docker_srsran; do
  docker pull ghcr.io/herlesupreeth/${img}:master
  docker tag ghcr.io/herlesupreeth/${img}:master ${img}
done
```

`dns` and `rtpengine` have no pre-built image — they build from source on
first `docker compose up -d` (takes a few extra minutes, no action needed).

---

## Step 2.5 — Provision Subscribers (REQUIRED, not automated by any script)

Neither `start_vonr.sh` nor the WebUI login work out of the box for this. You
must provision **both** the 5G core subscriber and the two IMS/pyHSS
subscribers *before* starting the UE, or registration will silently fail.
See [Subscriber Configuration](#subscriber-configuration) below for the exact,
verified commands (the WebUI login is blocked by a CSRF check, and the SQL in
older versions of this README used column names that don't exist in the
current pyHSS schema — use the corrected SQL there, not `id`).

---

## Step 3 — Start the Stack

```bash
~/start_vonr.sh
```

The script does this automatically:

| Step | Action |
|------|--------|
| 1 | Stop conflicting systemd services + disable ufw |
| 2 | Remove any leftover containers |
| 3 | Start 22 core containers via `sa-vonr-deploy.yaml` (no `ibcf`, see Step 2) |
| 4 | Wait 30s for initialization |
| 5 | Fix pyHSS `operation_log` schema (ALTER TABLE) |
| 6 | Start srsRAN gNB → wait 15s |
| 7 | Start srsRAN UE → wait 20s |
| 8 | Add routing rules on P/I/S-CSCF to reach UE subnet |
| 9 | Add NAT/MASQUERADE on UPF for IMS traffic |
| 10 | Install linphonec + tcpdump inside UE container |
| 11 | Add IMS DNS entries to UE `/etc/hosts` |
| 12 | Create linphonec configs for UE1 (port 5070) and UE2 (port 5071) |

> Step 8 needs `cap_add: NET_ADMIN` on `icscf` and `scscf` in the compose file
> to actually succeed (only `pcscf` has it by default) — see
> [Bug 12](#bug-12--icscfscscf-route-add-fails-silently-no-net_admin). Without
> subscribers provisioned (Step 2.5) and that capability added, this step will
> still report "ready" but the UE/calls will not actually work — the script
> doesn't verify its own success.

**Expected output at end:**
```
=== VoNR stack ready ===
PDU Session Establishment successful. IP: 192.168.101.2
Run: ~/vonr_call.sh
```

>  If you see `IP: 192.168.100.x` — the UE got the internet APN. See [Troubleshooting](#troubleshooting).
>
>  If the UE hangs forever at `Attaching UE...` with no further log output,
>  the gNB and UE ZMQ link went stale (usually after restarting only one of
>  them). Recreate **both together**:
>  ```bash
>  cd ~/docker_open5gs
>  docker compose -f srsue_5g_zmq.yaml down
>  docker compose -f srsgnb_zmq.yaml down
>  docker compose -f srsgnb_zmq.yaml up -d && sleep 15
>  docker compose -f srsue_5g_zmq.yaml up -d
>  ```

---

## Step 4 — Make a VoNR Call

```bash
~/vonr_call.sh
```

**Expected output:**
```
=== UE1 ===
Call 1 to sip:9076543211@ims.mnc001.mcc001.3gppnetwork.org ringing.
Call 1 with sip:9076543211@ims.mnc001.mcc001.3gppnetwork.org connected.
Media streams established with sip:9076543211@ims.mnc001.mcc001.3gppnetwork.org for call 1 (audio).
Call 1 with sip:9076543211@ims.mnc001.mcc001.3gppnetwork.org ended (No error).

=== UE2 ===
-------auto answering to call-------
Call 1 with sip:9076543210@ims.mnc001.mcc001.3gppnetwork.org connected.
Media streams established with sip:9076543210@ims.mnc001.mcc001.3gppnetwork.org for call 1 (audio).
Call 1 with sip:9076543210@ims.mnc001.mcc001.3gppnetwork.org ended (No error).
```

If you see `ringing → connected → Media streams established → ended (No error)` on **both UEs** — VoNR is working.

---

## Step 5 — Full KPI Measurement

```bash
~/vonr_full_kpi_logs.sh
```

This script captures a complete VoNR call, copies the pcap to `~/vonr.pcap`, and prints:

- **SIP call flow** — every REGISTER, INVITE, 180, 200, ACK, BYE with timestamps
- **Call setup time** — time from INVITE to 200 OK (our result: **0.061 seconds**)
- **Call duration** — time from INVITE to BYE (our result: **19.877 seconds**)
- **RTP KPIs** — packets, loss, mean delta, jitter per stream

**Expected output (abbreviated):**
```
SIP CALL FLOW
==============================
24.87s   INVITE
24.94s   180 Ringing
24.94s   200 OK
25.16s   ACK
44.75s   BYE
44.76s   200 OK

Call Setup Time: 0.061 seconds
Call Duration:   19.877 seconds

RTP KPI SUMMARY
==============================
Packets: 978     Lost: 0 (0.0%)
Mean Jitter: 9.912ms    Max Jitter: 10.604ms
Mean Delta: 19.988ms    Codec: opus
```

---

## Step 6 — Verify Everything

```bash
~/verify_vonr_complete.sh
```

Runs automated checks across all layers. **Expected: 55/55 PASS**.

| Section | Checks |
|---------|--------|
| 1. Containers | All 21 required NFs are Up |
| 2. gNB | Started, AMF connected, ZMQ active |
| 3. UE 5G | Random Access, RRC Connected, IMS IP 192.168.101.2 |
| 4. 5G Core | AMF, SMF IMS DNN, UPF ogstun2 tunnel, NAT rule |
| 5. IMS Routing | Routes to UE subnet on P/I/S-CSCF |
| 6. Diameter | pyHSS peers connected |
| 7. Subscriber DB | AUC, subscriber, IMS subscriber, APNs |
| 8. SIP Registration | Both UEs registered, no 412 errors |
| 9. Live Call Test | Ringing → connected → media → clean BYE |
| 11. QoS | ogstun QFI=9 and ogstun2 QFI=1 active |

---

## Step 7 — Stop Before Shutdown

```bash
~/stop_vonr.sh
```

Always run this before shutting down. Cleanly stops all containers.

---

## Critical Bugs Fixed

> These bugs exist in the upstream `docker_open5gs` repo. All fixes are applied automatically by `start_vonr.sh`. Documented here so you understand what was changed and why.

---

### Bug 1 — pyHSS IMSI/MSISDN Mismatch Most Critical

**Symptom:** SIP REGISTER never gets a response. I-CSCF Diameter UAR returns empty `server_name` AVP.

**Root cause:** pyHSS passes the public SIP identity (MSISDN = `9076543210`) as the key to query `WHERE imsi = ?`. But the database stores the real IMSI (`001011234567895`). The SQL returns zero rows, silently.

**Fix — set `imsi = MSISDN` in all three IMS tables:**
```sql
UPDATE auc SET imsi='9076543210' WHERE id=1;
UPDATE subscriber SET imsi='9076543210' WHERE id=1;
UPDATE ims_subscriber SET imsi='9076543210' WHERE id=1;
```

**How to debug:** Check pyHSS SQLAlchemy logs:
```bash
docker logs pyhss 2>&1 | grep "WHERE imsi" | tail -3
```

---

### Bug 2 — N5 QoS Authorization Failure (412 Error)

**Symptom:** `412 Precondition Failed — N5 QoS authorization failed` on every SIP REGISTER.

**Root cause:** `pcscf/pcscf_init.sh` enables `WITH_N5` flag when `DEPLOY_MODE=5G`. PCF N5 interface is not configured for IMS in the default stack.

**Fix — apply before building containers:**
```bash
sed -i '/WITH_N5/s/^/#DISABLED /' pcscf/pcscf_init.sh
```

**Verify fix:**
```bash
docker logs pcscf 2>&1 | grep "412" | wc -l   # must be 0
```

---

### Bug 3 — srsUE Reads Wrong Config File

**Symptom:** Setting `apn = ims` in the mounted config has no effect. UE always gets `192.168.100.x` (internet APN) instead of `192.168.101.x` (IMS APN).

**Root cause:** The container reads `/etc/srsran/ue.conf` at runtime — NOT the mounted file at `/mnt/srslte/ue_5g_zmq.conf`.

**Fix — edit both:**
```bash
sed -i 's/apn = internet/apn = ims/' srslte/ue_5g_zmq.conf
docker exec srsue_5g_zmq sed -i 's/apn = internet/apn = ims/' /etc/srsran/ue.conf
```

---

### Bug 4 — pyHSS operation_log Schema Error

**Symptom:** pyHSS crashes with `IntegrityError: NOT NULL constraint failed: operation_log.item_id` when processing SAR messages.

**Fix — run after every fresh container start:**
```bash
docker exec mysql mysql -u root -pchangeme ims_hss_db \
  -e "ALTER TABLE operation_log MODIFY item_id INTEGER NULL;"
```
Already included in `start_vonr.sh`.

---

### Bug 5 — Diameter Peer Reconnection Timing

**Symptom:** After restarting icscf/scscf, Diameter UAR returns `Connection refused` on port 3875.

**Root cause:** pyHSS Diameter server needs ~15 seconds to start listening after container start.

**Fix — always start in this order:**
```bash
docker compose -f sa-vonr-ibcf-deploy.yaml up -d   # starts pyHSS
sleep 30                                             # wait for Diameter to listen
docker compose -f srsgnb_zmq.yaml up -d
docker compose -f srsue_5g_zmq.yaml up -d
```
Already handled in `start_vonr.sh`.

---

### Bug 6 — baresip Cannot Read stdin in Docker

**Symptom:** baresip starts but ignores all typed commands. Calls cannot be made.

**Root cause:** Docker security context blocks `epoll` on file descriptor 0 (stdin).

**Fix:** Use **linphonec** instead. It reads stdin correctly inside Docker containers.

---

### Bug 7 — linphonec Overwrites Config with Wrong Defaults

**Symptom:** `verify_server_certs=1` appears in config after each run, silently blocking SIP registration.

**Root cause:** linphonec writes its state database to `$HOME/.local/share/linphone/` and overwrites your config on startup if HOME is shared.

**Fix — give each UE its own HOME directory:**
```bash
docker exec srsue_5g_zmq mkdir -p /root/ue1/.local/share/linphone
docker exec srsue_5g_zmq mkdir -p /root/ue2/.local/share/linphone

# Run UE1
docker exec srsue_5g_zmq bash -c 'export HOME=/root/ue1; linphonec -c /root/ue1/linphonerc'

# Run UE2
docker exec srsue_5g_zmq bash -c 'export HOME=/root/ue2; linphonec -c /root/ue2/linphonerc'
```

---

### Bug 8 — DNS Resolution Fails for IMS Domain

**Symptom:** `belle-sip-error: DNS resolution failed for scscf.ims.mnc001.mcc001.3gppnetwork.org`

**Fix:**
```bash
docker exec srsue_5g_zmq bash -c '
echo "172.22.0.21 ims.mnc001.mcc001.3gppnetwork.org" >> /etc/hosts
echo "172.22.0.20 scscf.ims.mnc001.mcc001.3gppnetwork.org" >> /etc/hosts
echo "172.22.0.19 icscf.ims.mnc001.mcc001.3gppnetwork.org" >> /etc/hosts'
```
Already included in `start_vonr.sh`.

---

### Bug 9 — tcpdump Inside Container Captures 0 Packets

**Symptom:** `tcpdump -i eth0` inside srsue_5g_zmq container captures nothing during an active call.

**Root cause:** Docker iptables forwarding — inter-container traffic does not pass through the container's `eth0` in a way tcpdump can intercept.

**Fix:** Capture on the Docker network inside the container using `-i any`:
```bash
docker exec -u root srsue_5g_zmq bash -c \
  "tcpdump -i any -w /tmp/vonr.pcap port 5060 or udp"
```
This is exactly what `vonr_full_kpi_logs.sh` does — it captures on `-i any` instead of `-i eth0`.

---

### Bug 10 — `ibcf` Build Fails on External ETSI Download (403 Forbidden)

**Symptom:** `docker compose -f sa-vonr-ibcf-deploy.yaml up -d` fails while building `ibcf`, with:
```
ERROR: process "/bin/sh -c wget www.etsi.org/deliver/.../ts_126443v160100p0.zip" did not complete successfully: exit code: 8
403 Forbidden
```

**Root cause:** `ibcf`'s Dockerfile fetches an ETSI EVS codec spec at build time; etsi.org now blocks this request (likely bot/UA filtering). This is an external dependency, not a bug in this repo, but it currently makes `sa-vonr-ibcf-deploy.yaml` unusable as-is.

**Fix:** Use `sa-vonr-deploy.yaml` instead (see [Step 2](#step-2--clone-and-configure)) — `ibcf` is not required for the VoNR call path shown in the [Architecture](#architecture) diagram.

---

### Bug 11 — pyHSS Crashes Generating SAA When `ifc_path` Is NULL

**Symptom:** SIP REGISTER gets stuck retrying 401 challenges forever; S-CSCF logs show `Transaction timeout - did not get SAA`; pyHSS logs show:
```
[ERROR] [diameter.py] [generateDiameterResponse] [SAR] Error generating response:
AttributeError: 'NoneType' object has no attribute 'split'
```

**Root cause:** pyHSS renders the initial-filter-criteria XML via Jinja2 using `ims_subscriber.ifc_path`. If that column is NULL (as it is after inserting subscribers with the minimal columns shown in older versions of this README), Jinja2's template loader crashes trying to `.split("/")` on `None`, and the Server-Assignment-Answer is never sent — so S-CSCF can never complete registration.

**Fix:** Always set `ifc_path` and `sh_template_path` when provisioning a subscriber:
```sql
UPDATE ims_subscriber SET ifc_path='default_ifc.xml', sh_template_path='default_sh_user_data.xml';
```
Both files ship inside the `pyhss` container at `/pyhss/default_ifc.xml` and `/pyhss/default_sh_user_data.xml`. See the corrected SQL in [Subscriber Configuration](#subscriber-configuration).

---

### Bug 12 — I-CSCF/S-CSCF Route-Add Fails Silently (No `NET_ADMIN`)

**Symptom:** `start_vonr.sh`'s Step 8 (`docker exec icscf ip route add ...`, `docker exec scscf ip route add ...`) fails with `Operation not permitted`, silenced by the script's `2>/dev/null`. `verify_vonr_complete.sh` reports `[FAIL] I-CSCF — no route to UE subnet` / `S-CSCF — no route to UE subnet`.

**Root cause:** Only `pcscf` has `cap_add: NET_ADMIN` / `privileged: true` in `docker_open5gs`'s compose files by default. In practice **P-CSCF is the only element that needs the direct route** (it is the sole UE-facing proxy — I-CSCF/S-CSCF only talk to pyHSS via Diameter and to P-CSCF), so this doesn't block calls. To get a clean 55/55 on the verification script anyway, add the capability and recreate:

```bash
# In sa-vonr-deploy.yaml, under both `icscf:` and `scscf:` service blocks, add:
#   cap_add:
#     - NET_ADMIN

cd ~/docker_open5gs
docker compose -f sa-vonr-deploy.yaml up -d icscf scscf   # recreates with the new capability
docker exec icscf ip route add 192.168.101.0/24 via 172.22.0.8
docker exec scscf ip route add 192.168.101.0/24 via 172.22.0.8
```

---

### Bug 13 — UE Hangs Forever at "Attaching UE..." After a Lone Restart

**Symptom:** After `docker restart srsue_5g_zmq` (or recreating only the UE container), the UE log stops at `Attaching UE...` indefinitely — no Random Access attempt, no error, `/mnt/srslte/ue.log` stays empty. `docker restart srsgnb_zmq` alone shows the same symptom in reverse.

**Root cause:** The ZMQ RF link between gNB and UE is a pair of persistent TCP sockets (`tx_port`/`rx_port` in `ue_5g_zmq.conf` / `gnb_zmq.conf`). Restarting only one side of the pair leaves the ZMQ transport in a stale state that srsRAN doesn't recover from on its own — no PRACH is ever transmitted.

**Fix:** Always recreate the gNB and UE together, gNB first:
```bash
cd ~/docker_open5gs
docker compose -f srsue_5g_zmq.yaml down
docker compose -f srsgnb_zmq.yaml down
docker compose -f srsgnb_zmq.yaml up -d
sleep 15
docker compose -f srsue_5g_zmq.yaml up -d
```

---

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| `IP: 192.168.100.x` not `101.x` | Wrong APN in ue.conf | `docker exec srsue_5g_zmq sed -i 's/apn = internet/apn = ims/' /etc/srsran/ue.conf` then restart |
| SIP REGISTER never reaches P-CSCF | Missing DNS entries | Add ims.mnc001... to `/etc/hosts` inside container |
| 412 on SIP REGISTER | N5 QoS enabled | `sed -i '/WITH_N5/s/^/#/' pcscf/pcscf_init.sh` and restart pcscf |
| Call errors without ringing | UE2 not registered yet | Wait 20s after starting UE2 before calling |
| Diameter UAR returns empty | pyHSS IMSI/MSISDN bug | Set `imsi = MSISDN` in all 3 DB tables |
| pyHSS crashes on SAR | operation_log schema | `ALTER TABLE operation_log MODIFY item_id INTEGER NULL` |
| Containers conflict on start | Old containers still running | `docker stop $(docker ps -q) && docker rm $(docker ps -aq)` |
| linphonec ignores config | Wrong HOME directory | Use `export HOME=/root/ue1` before running linphonec |
| tcpdump captures 0 packets | Wrong interface | Use `-i any` not `-i eth0` inside container |
| Call setup > 16 seconds | ICE/STUN negotiation | Expected in linphonec — disable ICE in linphonerc for faster setup |
| `ibcf` build fails, 403 from etsi.org | External EVS codec download blocked | Use `sa-vonr-deploy.yaml` (skip `ibcf`) — see [Bug 10](#bug-10--ibcf-build-fails-on-external-etsi-download-403-forbidden) |
| `docker compose up` says `pull access denied` | Images aren't on Docker Hub | Pull from GHCR + `docker tag` locally — see [Step 2](#step-2--clone-and-configure) |
| REGISTER loops on 401 forever, never 200 OK | pyHSS `ifc_path` NULL, SAA never sent | `UPDATE ims_subscriber SET ifc_path=..., sh_template_path=...` — see [Bug 11](#bug-11--pyhss-crashes-generating-saa-when-ifc_path-is-null) |
| PDU session rejected: "DNN not supported in slice" | `ims` APN slice missing `sst` | Set `slice.$.sst` to match the default slice (usually `1`) — see [Subscriber Configuration](#subscriber-configuration) |
| UE stuck at `Attaching UE...` forever, no logs | gNB/UE ZMQ link went stale from a lone restart | Recreate gNB + UE together — see [Bug 13](#bug-13--ue-hangs-forever-at-attaching-ue-after-a-lone-restart) |
| `verify_vonr_complete.sh`: I-CSCF/S-CSCF no route | Only P-CSCF has `NET_ADMIN` by default | Harmless for calls; add `NET_ADMIN` to fix the check — see [Bug 12](#bug-12--i-cscfs-cscf-route-add-fails-silently-no-net_admin) |
| WebUI login returns `403 CSRF token missing` | `curl`/API login without a CSRF token | Provision subscribers via `open5gs-dbctl` instead (see [Subscriber Configuration](#subscriber-configuration)), or use a real browser for the WebUI |

---

## Scripts Reference

| Script | Purpose | When to Use |
|--------|---------|-------------|
| `start_vonr.sh` | Start full stack (26 containers + all config) | After every reboot |
| `vonr_call.sh` | Make a VoNR call between UE1 and UE2 | Quick call test |
| `vonr_full_kpi_logs.sh` | Full KPI measurement — SIP flow, setup time, RTP metrics | For measurements |
| `verify_vonr_complete.sh` | 55-point automated verification | Confirm everything works |
| `stop_vonr.sh` | Stop all containers cleanly | Before shutdown |

---

## Subscriber Configuration

> **This entire section is a required manual step** — `start_vonr.sh` does
> not provision any subscribers. Do this once, before the first
> `~/start_vonr.sh` run (or before starting the UE, if the core is already up).

### Open5GS 5G Core — via `open5gs-dbctl`, not the WebUI

The WebUI at `http://localhost:9999` (admin / 1423) looks like the obvious way
to do this, but its `/api/login` endpoint enforces CSRF checks that a plain
`curl` script can't satisfy — you'd need a real browser. It's simpler and
scriptable to use `open5gs-dbctl`, which ships inside the `webui` container
and talks to the same Mongo database:

```bash
# Base subscriber with the "internet" APN
docker exec webui /open5gs/misc/db/open5gs-dbctl --db_uri=mongodb://mongo/open5gs \
  add 001011234567895 8baf473f2f8fd09487cccbd7097c6862 8E27B6AF0E692E750F32667A3B14605D

# Add the "ims" APN as a second network slice
docker exec webui /open5gs/misc/db/open5gs-dbctl --db_uri=mongodb://mongo/open5gs \
  update_apn 001011234567895 ims 1
```

> **Critical, undocumented gap:** `update_apn` creates the second slice
> **without an `sst` (slice type) field**. The UE's PDU Session Establishment
> Request always carries the default S-NSSAI, and if the `ims` slice's `sst`
> doesn't match, SMF rejects it with `Ue requested DNN "ims" Not Supported OR
> Not Subscribed in the Slice` — the call never gets an IP. Fix it directly
> in Mongo, matching the `sst` your first (`internet`) slice already has
> (check with `showpretty` below — it's `1` unless you changed it):
> ```bash
> docker exec mongo mongosh --quiet open5gs --eval '
> db.subscribers.updateOne(
>   {imsi: "001011234567895", "slice.session.name": "ims"},
>   {$set: {"slice.$.sst": 1}}
> )'
> ```

Verify both slices look right (`sst` present on both, `ims` APN present):
```bash
docker exec webui /open5gs/misc/db/open5gs-dbctl --db_uri=mongodb://mongo/open5gs showpretty
```

| Field | Value |
|-------|-------|
| IMSI | `001011234567895` |
| Key (Ki) | `8baf473f2f8fd09487cccbd7097c6862` |
| OPC | `8E27B6AF0E692E750F32667A3B14605D` |
| AMF | `8000` |
| APN 1 | `internet` — slice `sst=1` (default) |
| APN 2 | `ims` — same slice `sst=1`, added via `update_apn` + the Mongo fix above |

### IMS pyHSS — via MySQL direct insert

> **Important #1:** The `imsi` column must be set to the **MSISDN value** (not the real IMSI) due to the pyHSS bug described in Bug 1.
>
> **Important #2:** The column names below (`auc_id`, `subscriber_id`,
> `ims_subscriber_id`, `apn_ambr_dl/ul`, `amf`) match the pyHSS schema as
> shipped in the `ghcr.io/herlesupreeth/docker_pyhss:master` image. An
> earlier version of this README used a plain `id` column and omitted
> `apn_ambr_dl/ul` and `amf` — those don't exist in this schema and the
> inserts fail with `Unknown column 'id' in 'field list'`. Check your own
> image with `DESCRIBE auc; DESCRIBE subscriber; DESCRIBE ims_subscriber;`
> if these don't match.
>
> **Important #3:** `ifc_path` and `sh_template_path` on `ims_subscriber`
> are **not optional** — see [Bug 11](#bug-11--pyhss-crashes-generating-saa-when-ifc_path-is-null).
> Without them, REGISTER loops on 401 forever and never reaches 200 OK.

```sql
-- Connect: docker exec -it mysql mysql -u root -pchangeme ims_hss_db
-- (use `docker exec -i` — not just `-it` — if piping this via a heredoc/script,
--  or the SQL never reaches the container's stdin and nothing happens)

-- APN entries (apn_ambr_dl/apn_ambr_ul are NOT NULL in this schema)
INSERT INTO apn (apn_id, apn, apn_ambr_dl, apn_ambr_ul) VALUES (1, 'internet', 1000000000, 1000000000);
INSERT INTO apn (apn_id, apn, apn_ambr_dl, apn_ambr_ul) VALUES (3, 'ims', 1000000000, 1000000000);

-- Authentication credentials (imsi = MSISDN intentionally, see Bug 1)
INSERT INTO auc (auc_id, imsi, ki, opc, amf, sqn)
VALUES (1, '9076543210', '8baf473f2f8fd09487cccbd7097c6862',
        '8E27B6AF0E692E750F32667A3B14605D', '8000', 0);

-- Subscriber (imsi = MSISDN intentionally, apn_list = '1,3')
INSERT INTO subscriber (subscriber_id, imsi, msisdn, enabled, auc_id, default_apn, apn_list)
VALUES (1, '9076543210', '9076543210', 1, 1, 1, '1,3');

-- IMS subscriber with S-CSCF address + IFC/Sh templates (see Bug 11)
INSERT INTO ims_subscriber (ims_subscriber_id, imsi, msisdn, scscf, ifc_path, sh_template_path)
VALUES (1, '9076543210', '9076543210',
        'sip:scscf.ims.mnc001.mcc001.3gppnetwork.org:6060',
        'default_ifc.xml', 'default_sh_user_data.xml');

-- Second subscriber (UE2)
INSERT INTO auc (auc_id, imsi, ki, opc, amf, sqn)
VALUES (2, '9076543211', '8baf473f2f8fd09487cccbd7097c6862',
        '8E27B6AF0E692E750F32667A3B14605D', '8000', 0);
INSERT INTO subscriber (subscriber_id, imsi, msisdn, enabled, auc_id, default_apn, apn_list)
VALUES (2, '9076543211', '9076543211', 1, 2, 1, '1,3');
INSERT INTO ims_subscriber (ims_subscriber_id, imsi, msisdn, scscf, ifc_path, sh_template_path)
VALUES (2, '9076543211', '9076543211',
        'sip:scscf.ims.mnc001.mcc001.3gppnetwork.org:6060',
        'default_ifc.xml', 'default_sh_user_data.xml');
```

`default_ifc.xml` and `default_sh_user_data.xml` ship inside the `pyhss`
container at `/pyhss/`. After inserting, restart pyHSS so the Diameter
service picks up a clean state and reapply the operation_log fix (Bug 4):

```bash
docker restart pyhss
sleep 15   # Diameter needs ~15s to start listening again, see Bug 5
docker exec mysql mysql -u root -pchangeme ims_hss_db \
  -e "ALTER TABLE operation_log MODIFY item_id INTEGER NULL;"
```

---

## References

| Resource | Link |
|----------|------|
| Base Docker stack | https://github.com/herlesupreeth/docker_open5gs |
| Open5GS docs | https://open5gs.org/open5gs/docs/ |
| srsRAN Project (gNB) | https://docs.srsran.com/projects/project |
| srsRAN 4G (srsUE) | https://docs.srsran.com/projects/4g |
| Kamailio IMS | https://www.kamailio.org/wiki/ |
| pyHSS | https://github.com/nickvsnetworking/pyhss |
| RTPEngine | https://github.com/sipwise/rtpengine |
| 3GPP TS 23.228 | IMS Stage 2 architecture |
| 3GPP TS 23.501 | 5G System architecture |
| 3GPP TS 26.114 | Voice quality requirements (jitter < 50ms, loss < 1%) |
| ITU-T G.107 | E-model for MOS score computation |


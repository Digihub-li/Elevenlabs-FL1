# ElevenLabs + FL1 SIP bridge

Small Asterisk registrar that registers an FL1 CommPlus trunk and forwards the call to an ElevenLabs SIP agent.

No customer credentials are in this repository. Copy `.env.example` to `.env` on the server.

## What you need

- A machine with its own public IPv4 (a small DigitalOcean droplet in Frankfurt or Amsterdam is enough, 1 GB).
- The FL1 trunk username, password, and SIP domain from CommPlus.
- The main number imported on an ElevenLabs agent as a SIP trunk (`sip.rtc.elevenlabs.io`).
- CommPlus IP filter opened for this machine's public IP.

## Droplet

Ubuntu 24.04. Install Docker, then:

```bash
apt-get update && apt-get install -y docker.io docker-compose-v2
git clone <this-repo> /opt/el-fl1-bridge
cd /opt/el-fl1-bridge
cp .env.example .env
# edit PUBLIC_IP, BIND_IP, FL1_SIP_USER, FL1_SIP_PASSWORD, FL1_SIP_DOMAIN, DID, CLI_NAME
python3 bridge/setup.py
cd bridge && docker compose up -d
docker exec el-fl1-bridge asterisk -rx "pjsip show registrations"
```

`Registered` means FL1 accepted the trunk. Inbound calls to `DID` are then dialed to ElevenLabs.

Cloud firewall, inbound:

- TCP 22 from your admin IP only
- TCP 5062 from the FL1 SBC (`FL1_SBC_IP`, default `80.66.238.49`)
- UDP 10000-10100 for RTP

Do not commit `.env` or anything under `bridge/asterisk/*.conf`. `setup.py` writes those from `.env`.

## Current deployment

The production bridge runs as Docker container `fo-voice-bridge` (image `andrius/asterisk:20`, host network) on the FO voice droplet. Its config lives in `/opt/fo-voice/asterisk/` (`pjsip.conf`, `extensions.conf`, `rtp.conf`, `modules.conf`), bind-mounted read-only into the container. The droplet is reached by SSH via the `spark-553e` jump host.

That config was generated once and has since been patched in place; `setup.py` mirrors it but is not re-run there. When editing live files, write in place (`cat new > file`, not `sed -i` or `mv`) so the single-file bind mounts keep the same inode, then reload without restarting:

```bash
docker exec fo-voice-bridge asterisk -rx "module reload res_pjsip.so"
docker exec fo-voice-bridge asterisk -rx "dialplan reload"
docker exec fo-voice-bridge asterisk -rx "module reload res_rtp_asterisk.so"
```

Hardening in place (2026-10-30): peers identified by source IP only (`endpoint_identifier_order=ip`, `identify_by=ip`), `strictrtp=yes` with `probation=4`, and an outbound allowlist in `[from-elevenlabs]`: LI `+423`/`423`/`0423`/`00423` and CH `+41`/`41`/`0041` and Swiss national `0[1-8]…`/`091…` (10 digits, rewritten to `41…`), with CH 090x and LI 90x premium numbers and everything else rejected (`Hangup(21)` plus a log line).

## Layout

```text
bridge/setup.py            renders Asterisk config from .env
bridge/docker-compose.yml  Asterisk 20, host network
bridge/asterisk/           generated, gitignored
docs/ARCHITECTURE.md       call flow
```

Outbound PSTN (the bridge calling a mobile through FL1) depends on the trunk being allowed to terminate calls. Registration alone is the inbound path.

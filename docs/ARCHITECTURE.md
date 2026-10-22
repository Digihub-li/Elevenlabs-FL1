# ElevenLabs behind an FL1 SIP trunk

FL1 delivers a number only to a PBX that stays registered. ElevenLabs does not register on the customer trunk, and CommPlus cannot forward the number to a `sip:` URI. This bridge is that PBX.

```text
Caller
  -> FL1 SBC (TCP 5083)
  -> this bridge (Asterisk, REGISTER + media anchor)
  -> ElevenLabs (sip.rtc.elevenlabs.io TCP 5060)
```

1. The bridge sends SIP REGISTER to the trunk with the CommPlus username and password.
2. FL1 sends the inbound INVITE to the registered contact.
3. The bridge dials ElevenLabs and stays in the media path. RTP does not flow directly between the SBC and ElevenLabs.
4. Codecs offered: G.711 A-law, G.711 u-law, G.722.

Cloudflare Workers cannot carry this call. They have no UDP RTP socket and cannot hold the trunk registration.

## Firewall on the bridge host

| Direction | Port | Who |
|-----------|------|-----|
| Inbound TCP | 22 | your admin address only |
| Inbound TCP | 5062 | FL1 SBC only |
| Inbound UDP | 10000-10100 | RTP from FL1 and ElevenLabs |
| Outbound TCP | 5083 | FL1 SBC |
| Outbound TCP | 5060 | ElevenLabs |

In CommPlus, exempt the bridge public IP from the trunk IP filter, or ask FL1 to whitelist it.

## What this repo does not include

No trunk password, no real username, no phone number, no ElevenLabs API key, and no agent prompt. Those stay in `.env` on the server.

# PCR — Private Chat Room

**A lightweight, serverless chat room built with WebRTC Direct Mesh.**

PCR is a browser-based, single-file chat application for small private groups. Participants connect directly to one another, without a central chat server or message relay. Room setup is handled through a simple invite–reply–confirmation exchange.

<p align="center">
  <strong>Peer-to-peer · IPv6-only · No backend required</strong>
</p>

## Features

- **Private rooms** — Host and Peer roles, room passwords, topics, participant profiles, and moderation tools.
- **Real-time chat** — Room messages, direct messages, replies, reactions, typing indicators, and message editing.
- **Voice** — Recorded voice messages and one-to-one live voice calls.
- **File sharing** — Direct peer-to-peer transfers with progress tracking, cancellation, and SHA-256 integrity checks.
- **Connection setup** — Manual signaling codes with copy/paste and optional QR support.
- **Privacy-focused design** — No central chat server; direct WebRTC connections between participants.
- **Local history** — Optional encrypted chat history stored in the browser.
- **Customizable interface** — English and Persian labels, themes, accent colors, and Jalali/Gregorian date display.
- **Built-in diagnostics** — Tools for checking IPv6 availability, connection status, permissions, and room setup.

## How It Works

PCR uses a **full-mesh peer-to-peer architecture**. The Host creates the room and coordinates participant setup. Once connected, participants exchange chat and other supported data directly over WebRTC DataChannels.

```text
                  HOST
          Room setup and coordination
                    |
         Invite → Reply → Confirmation
                    |
          +---------+---------+
          |                   |
        Peer A  <----------> Peer B
          |      Direct      |
          |      WebRTC      |
          +---------+---------+
                    |
                  Peer C
```

The Host coordinates the room but does not act as a relay for normal peer-to-peer chat traffic. Full mesh is designed for small groups; each participant establishes a direct connection with the other participants.

## Networking

PCR currently uses **IPv6-only direct connections**.

- No STUN or TURN servers are configured.
- No external signaling or chat server is required.
- Public IPv6 addresses can be used for connections across networks.
- Unique Local Addresses (ULA, commonly beginning with `fd`) can be used when devices are reachable on the same local network.
- IPv4-only connectivity is not supported.

Connection success depends on IPv6 availability and network rules. Firewalls, VPNs, router settings, and Wi-Fi client isolation may prevent direct connections. PCR does not provide a relay fallback.

Use `/nettest` on both devices to check the available IPv6 candidates. If the browser only exposes an mDNS `.local` address, PCR may need microphone permission to discover a usable address. This permission check stops the acquired audio tracks immediately; it does not start a recording.

## Getting Started

### Requirements

- A modern browser with WebRTC and Web Crypto support.
- A usable IPv6 path between participants.
- Microphone permission for voice features and, on some browsers, network-address discovery.
- Camera permission only when using QR scanning.
- HTTPS or `localhost` for browser features that require a secure context.

### Run locally

Clone the repository:

```bash
git clone https://github.com/taha8478/PCR.git
cd PCR
```

Start a local web server:

```bash
python -m http.server 8000
```

Open `http://localhost:8000/PCR.html` in your browser.

PCR is a static HTML application. No package installation, build process, or backend setup is required.

### Create or join a room

1. Open `PCR.html` in the browser.
2. On the first device, choose **Create room as Host**.
3. Set a nickname and any optional room details.
4. Share the generated invite code with the participant through a separate channel.
5. On the second device, choose **Join as Peer** and enter or scan the invite code.
6. The Peer generates a reply code. Send it back to the Host.
7. The Host verifies the reply and returns a confirmation code.
8. The Peer applies the confirmation code to complete the connection.

Repeat the setup process to add more participants. QR support is optional; codes can also be exchanged by copy and paste.

## Commands

PCR includes an in-app command system. Enter `/help` in the chat to see the commands available in the current version.

Common commands include:

| Command | Description |
|---|---|
| `/help` | Show available commands |
| `/nick <name>` | Change your nickname |
| `/bio [text]` | Set or view your bio |
| `/users` | List room participants |
| `/msg <user> <message>` | Send a direct message |
| `/topic [text]` | View or update the room topic |
| `/call <user>` | Start a one-to-one voice call |
| `/answer` · `/decline` · `/hangup` | Manage an incoming or active call |
| `/status` | Show room and connection status |
| `/nettest` | Check IPv6 connectivity information |
| `/security` | Display security information |
| `/history on` · `/history off` · `/history clear` | Manage local chat history |
| `/search <term>` | Search available chat messages |
| `/export` | Export available chat history |

Host moderation commands and additional interface, room, and diagnostic commands are also available through `/help`.

## Security and Privacy

PCR combines WebRTC transport encryption with application-level encryption for text messaging.

- **Transport:** WebRTC DataChannels use DTLS encryption.
- **Room messages:** Room-wide text messages use ECDH, HKDF, and AES-256-GCM. The Host holds the room key and can read room-wide messages.
- **Direct messages:** Peer-to-peer direct messages use a key derived by the two participants. The Host is not in the data path for messages exchanged between two Peers.
- **Files and voice:** File and voice-message payloads are sent directly over WebRTC and use its transport encryption. They do not use the same application-level encryption layer as text messages.
- **Identity:** PCR displays a Host fingerprint. When identity verification matters, compare it with the Host through a separate trusted channel.
- **Local history:** When enabled, chat history is encrypted and stored locally in the browser.

PCR is experimental and has not undergone an independent security audit. Direct peer-to-peer connections may expose network information, including IPv6 addresses, to other participants. Use the application with people you trust and avoid sharing room setup codes publicly.

## Troubleshooting

**A participant cannot connect**
- Run `/nettest` on both devices and confirm that each has a usable IPv6 address.
- Check that the invite, reply, and confirmation codes were copied completely.
- Review `/bootstrap`, `/network`, and `/status` for connection details.
- Check firewall, VPN, router, and Wi-Fi client-isolation settings.

**The connection works on the same network but not across networks**

Local ULA addresses are generally limited to the local network. Connections across networks require usable public IPv6 connectivity on both sides.

**Voice features are unavailable**

Check browser compatibility, microphone permission, and whether the page is served from HTTPS or `localhost`.

**QR scanning is unavailable**

QR scanning depends on browser support and camera permission. Use copy/paste if scanning is not supported.

**A file transfer fails**

Check the file-size limit shown by the application, confirm that the peer connection is still active, and try a smaller file.

## Technical Overview

| Item | Implementation |
|---|---|
| Application | Single-file HTML (`PCR.html`) |
| Network architecture | WebRTC Direct Mesh |
| IP support | IPv6 only |
| STUN / TURN | Not used |
| Signaling | Manual invite, reply, and confirmation |
| Application protocol | Version 4 |
| Signaling format | Version 5 |
| Mesh signaling protocol | Version 1 |
| Maximum message length | 2,000 characters |
| File-size limits | 32 MB (up to 4 participants), 16 MB (5–9), 8 MB (10+) |
| Concurrent file transfers | Up to 4 |
| Local chat history | Up to 500 records |

Protocol and signaling formats may change as the project evolves. Use compatible versions of PCR when connecting participants.

## Project Structure

```text
PCR/
├── PCR.html
├── README.md
└── LICENSE.txt
```

The application is contained in `PCR.html`; the interface, networking, room setup, messaging, and supporting features are included in that file.

## License

PCR is released under the [MIT License](LICENSE.txt).

## Author

**taha8478** · [GitHub](https://github.com/taha8478)

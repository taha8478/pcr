# PCR — Private Chat Room

**A browser-based, serverless chat room built around WebRTC Direct Mesh.**

PCR is a single-file HTML application for creating small private chat rooms without running a dedicated chat server. Participants establish WebRTC connections using manually exchanged signaling codes, then communicate directly whenever their networks permit it.

> PCR is an experimental project. It is not a professionally audited secure messenger, and direct connectivity is not guaranteed on every network.

## Features

- Browser-based, single-file deployment (`PCR.html`)
- Host and Peer roles with manual Host Code → Peer Offer → Host Answer signaling
- QR-code display and scanning for signaling codes
- Direct-mesh room chat and peer-to-peer direct messages
- Voice messages (browser microphone permission required)
- File sharing with progress feedback and SHA-256 integrity verification
- Replies, emoji reactions, typing indicators, message search, and message export
- Nicknames, bios, away state, user lists, and connection-state indicators
- Room passwords and room topic
- IRC-inspired commands and moderation controls
- Persistent Host identity and Host fingerprint verification
- WebRTC DTLS transport encryption and application-level encryption
- Configurable ICE servers, including custom STUN and TURN entries
- Intranet mode for attempting connections on a local network without the default public STUN servers
- `/status`, `/bootstrap`, `/security`, and `/network` diagnostics
- Responsive desktop and mobile interface
- No build system or required backend

## Architecture

PCR separates room coordination (the **control plane**) from normal application traffic (the **data plane**).

The Host coordinates room membership, identity, roles, topic, moderation, and initial signaling. Chat messages, direct messages, files, voice messages, typing indicators, and reactions are designed to travel directly between participants over WebRTC DataChannels rather than through a central chat server.

```text
                  HOST
          Room coordination
                  |
       Manual signaling exchange
                  |
        +---------+---------+
        |                   |
      Peer A  <----------> Peer B
        |       WebRTC      |
        +---------+---------+
                  |
                Peer C
```

A full mesh requires a direct connection between each pair of participants. This is best suited to small rooms, not large public communities.

## Getting Started

### Requirements

- A modern browser with WebRTC support
- A microphone and permission to use it for voice messages
- A way to exchange Host Codes, Peer Offers, and Host Answers privately
- Network conditions that allow the peers to establish direct WebRTC connections

### Run locally

Clone the repository:

```bash
git clone https://github.com/taha8478/PCR.git
cd PCR
```

If the HTML file is named `PCR.html`, start a local static server from the directory:

```bash
python -m http.server 8000
```

Open `http://localhost:8000/PCR.html` in your browser. You can also open or host the HTML file directly, although browser permissions and WebRTC behavior can vary by context. For microphone access and reliable browser behavior, use `localhost` or HTTPS.

### Create a room

1. Open PCR on the device that will host the room.
2. Enter a nickname and optional bio.
3. Create a room as **Host**.
4. Share the generated **Host Code** with a participant.
5. The participant joins as **Peer**, enters or scans the Host Code, and generates a **Peer Offer**.
6. Send that Peer Offer to the Host.
7. The Host processes the offer and returns a **Host Answer**.
8. The Peer enters or scans the Host Answer and applies it to complete the connection.
9. Repeat the process for additional participants.

PCR uses manual signaling, so users need an external way to exchange these codes. Treat codes as connection setup data and verify the Host identity/fingerprint when prompted. Avoid posting room codes publicly.

## Networking: STUN, TURN, and Intranet Mode

PCR uses WebRTC ICE to negotiate a path between participants.

### Default behavior

Outside Intranet Mode, PCR includes Google's public STUN servers by default:

```text
stun:stun.l.google.com:19302
stun:stun1.l.google.com:19302
stun:stun2.l.google.com:19302
stun:stun3.l.google.com:19302
stun:stun4.l.google.com:19302
```

STUN helps a browser discover network candidates; it does not relay chat messages or files.

### Custom ICE servers

PCR supports custom STUN/TURN entries through its network configuration. Entries are comma-separated. For TURN, provide the URL, username, and credential in this format:

```text
stun:stun.example.org:3478
turn:turn.example.org:3478|USERNAME|PASSWORD
turns:turn.example.org:5349|USERNAME|PASSWORD
```

Use credentials from a TURN service you trust. Do not share a README or screenshot containing live TURN credentials. Custom server support does not mean a server is bundled with PCR.

### Intranet Mode

Intranet Mode omits the default public STUN list and shortens the ICE candidate-gathering timeout. It is intended for local-network testing and may help when public internet services are unavailable. It does not guarantee connectivity: browser privacy behavior, Wi-Fi client isolation, firewalls, VPNs, and network configuration can still prevent peers from connecting.

### Important connectivity limitation

PCR attempts direct peer-to-peer connections. If no usable ICE candidate pair can be established, the connection fails. A configured TURN server can provide a relay path; without a working TURN service, some NATs, firewalls, mobile networks, or restricted environments will prevent a direct connection.

## Security Model

PCR uses multiple layers of protection, but its trust model has important limits.

### Transport encryption

WebRTC DataChannels use DTLS for transport encryption.

### Room-wide messages

Room-wide messages use application-level cryptographic protection. The Host creates and holds the room-wide key, so the Host can read room-wide messages as a member of the room. PCR should therefore not be treated as a group messenger where the Host is unable to read room messages.

### Direct messages

Direct messages are intended to use a key derived for the two participating peers and to travel over their direct mesh connection.

### Host identity and signaling

PCR maintains a Host identity and displays a fingerprint. Host Answers are checked against the Host identity accepted during Host Code verification. Check the fingerprint through a trusted channel when identity verification matters.

### File integrity and input validation

File transfers include SHA-256 integrity verification. The application also validates incoming message and transfer data. These measures reduce specific risks, but they do not replace an independent security audit.

### Security caveats

- PCR has not been represented as professionally audited security software.
- The Host is trusted for room coordination and room-wide messages.
- WebRTC may expose network metadata, including candidate information, to connection participants.
- A compromised device or browser can expose messages before encryption or after decryption.
- Manual signaling requires care: do not accept codes from untrusted sources.
- Encryption does not make users anonymous.

Use PCR for experimentation and small private rooms, not as a substitute for a professionally audited messenger when high-assurance security is required.

## File and Voice Sharing

- Maximum file size: **4 MB per file**
- File data is split into chunks and sent through WebRTC DataChannels
- The sender applies backpressure handling to avoid endlessly filling a recipient's send buffer
- Up to four concurrent transfers are allowed by the current implementation
- The receiver verifies the completed file's SHA-256 hash
- Voice recording depends on browser support and microphone permission

Transfer success depends on the peer connection remaining usable. Large or numerous transfers can consume memory and bandwidth on participant devices.

## Commands

Enter commands in the chat input. Use `/help` in the running application for the in-app command list and syntax. Commands currently recognized by the client include:

| Command | Purpose |
|---|---|
| `/help` | Show available commands |
| `/nick` | Change nickname |
| `/bio` | Set or update bio |
| `/users`, `/names`, `/who` | Inspect room participants |
| `/whois` | View participant details |
| `/msg`, `/query` | Send a direct message |
| `/me` | Send an action message |
| `/away` | Set away status |
| `/ignore`, `/unignore` | Ignore or restore messages from a user |
| `/mute`, `/unmute` | Mute or unmute a user locally |
| `/topic` | View or manage the room topic, subject to permissions |
| `/invite`, `/join`, `/part`, `/leave`, `/room` | Room-related actions |
| `/kick`, `/ban`, `/unban` | Moderation actions, subject to permissions |
| `/op`, `/deop`, `/voice`, `/devoice` | Manage room roles, subject to permissions |
| `/status` | Show connection and room status |
| `/bootstrap` | Inspect Host/Peer setup diagnostics |
| `/security` | Show security-model information |
| `/network` | Inspect network configuration/status |
| `/version`, `/time`, `/motd`, `/ping` | Information and diagnostic commands |
| `/search` | Search chat history available in the current client |
| `/export` | Export available chat history |
| `/clear` | Clear the visible chat |
| `/reload`, `/cycle`, `/quit` | Client/room lifecycle actions |

Exact behavior can depend on the user's role and the room state. `/help` in the current build is the source of truth for command usage.

## Troubleshooting

### The Peer does not connect after applying the Host Answer

1. Confirm that the Host processed the correct Peer Offer and returned the matching Host Answer.
2. Make sure the Peer applied the entire Host Answer without truncation or modification.
3. Inspect `/bootstrap` on both sides for the stage where setup stopped.
4. Inspect `/status` and `/network` for ICE, peer-connection, and DataChannel states.
5. Test without a VPN, then test with both devices on different networks.
6. If direct ICE connectivity fails, try a trusted, correctly configured TURN server.

### It works on the same Wi-Fi but not across networks

Different networks can have stricter NATs and firewalls. STUN alone cannot guarantee a direct path. A TURN server may be needed as a relay fallback.

### Intranet Mode does not connect

Check that both devices are on a network that allows device-to-device traffic. Some guest Wi-Fi networks and hotspots isolate clients. Ensure firewalls permit browser WebRTC traffic. Intranet Mode does not bypass network isolation.

### Voice recording does not work

Check browser support, microphone permission, and whether the page is running in a context where microphone access is allowed.

### File transfer fails

Confirm the file is within the 4 MB limit, the peer remains connected, and the browser has enough available memory. Check the transfer status and connection diagnostics.

## Testing Checklist

Test with at least two browser instances; then test on separate physical devices and networks.

- [ ] Create a room and generate a Host Code
- [ ] Join as a Peer and generate a Peer Offer
- [ ] Process the Peer Offer and apply the Host Answer
- [ ] Verify Host identity/fingerprint handling
- [ ] Exchange public messages
- [ ] Send direct messages
- [ ] Test replies, reactions, typing indicators, and search
- [ ] Record and send a voice message
- [ ] Send a file and verify integrity
- [ ] Test multiple participants and simultaneous transfers
- [ ] Test disconnect/reconnect behavior
- [ ] Review `/status`, `/bootstrap`, `/security`, and `/network`
- [ ] Test normal mode, Intranet Mode, and a trusted custom TURN configuration where available

Testing all participants on one computer or the same LAN does not reproduce the NAT and firewall conditions found on real-world networks.

## Project Structure

```text
PCR/
├── PCR.html
└── README.md
```

The core client is contained in `PCR.html`; it includes the interface, signaling flow, WebRTC networking, cryptographic logic, file-transfer implementation, and diagnostics. There is no build step.

## Technical Reference

| Setting | Current value |
|---|---|
| Application protocol version | `3` |
| Network mode | `DIRECT_MESH` |
| Maximum message length | `2,000` characters |
| Maximum file size | `4 MB` |
| Maximum concurrent file transfers | `4` |
| Default public STUN | Google STUN servers |
| TURN | Optional custom configuration; no TURN service is bundled |

Protocol versions describe the current implementation and may change between releases. Participants should use compatible versions of PCR.

## Limitations

- **Direct connectivity is not guaranteed.** Network conditions may prevent a direct path.
- **Small-room architecture.** Full mesh connection count grows approximately as `N × (N - 1) / 2`.
- **Manual signaling.** Users must exchange setup codes out of band.
- **Host trust.** The Host controls room coordination and can read room-wide messages.
- **No built-in offline delivery.** A disconnected participant cannot receive messages while offline.
- **No bundled TURN service.** Users must configure their own service if relay fallback is needed.
- **Experimental software.** The implementation has not been presented as independently audited security software.

## Privacy

Serverless does not mean anonymous. Participants and network infrastructure may learn connection metadata, and WebRTC candidate gathering can reveal network information. PCR does not send normal chat traffic through a central chat server, but peer-to-peer communication creates direct network relationships between participants.

Be thoughtful about who you invite, how you exchange signaling codes, and whether your network and device are suitable for your use case.

## License

The project README currently identifies the license as **MIT**. Ensure a matching `LICENSE` file is included in the repository before distributing the project under that license.

## Author

**taha8478**

- GitHub: <https://github.com/taha8478>

## AI-Assisted Development

PCR was developed with assistance from AI tools for code generation, debugging, architectural discussion, and iterative improvements. The project direction, implementation review, testing, and final decisions are guided by the author.

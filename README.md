# PCR

**Private Chat Room — a serverless, peer-to-peer chat application built with WebRTC Direct Mesh.**

PCR is a self-contained, IRC-style private chat client that runs entirely in the browser. It uses WebRTC DataChannels for direct peer-to-peer communication, manual signaling for bootstrapping, public STUN for NAT discovery, and application-level cryptography for protected room and peer-to-peer messaging.

The project intentionally does **not** use a chat server, WebSocket backend, database, TURN relay, Cloudflare Worker, or other permanent application infrastructure.

## Highlights

- Direct WebRTC mesh networking
- Serverless client architecture
- IRC-style private room experience
- Public room chat
- Peer-to-peer direct messages
- Peer-to-peer file transfer
- Voice messages
- Typing indicators
- Message replies
- Emoji reactions
- User presence, status/bio, away state, operator and voice roles
- Room password support
- Signed Host Codes and Host Answers
- Persistent host identity
- Host identity fingerprints
- ECDH-based key agreement
- AES-256-GCM application-level encryption
- DTLS-encrypted WebRTC transport
- QR-code display and scanning for signaling codes
- Built-in `/status`, `/bootstrap`, and `/security` diagnostics
- File integrity verification using SHA-256
- File-transfer backpressure handling
- Bounded reconnect and replay protection
- Responsive desktop and mobile UI
- Single-file deployment

## How PCR Works

PCR separates room coordination from application data.

The Host is authoritative for the **control plane**:

- signaling
- room membership
- participant registry
- identity
- roles
- moderation
- topic
- join/leave state

Normal application traffic belongs to the **data plane**:

- chat
- direct messages
- files
- voice
- typing indicators
- reactions
- acknowledgements

These data-plane messages are sent directly between participants over the WebRTC mesh rather than being relayed through the Host. The implementation explicitly uses `DIRECT_MESH` as its network mode. fileciteturn1file2L202-L208

Conceptually:

```text
                    ┌─────────────┐
                    │    HOST     │
                    │ Control     │
                    │ Plane       │
                    └──────┬──────┘
                           │
                    Bootstrap / Signaling
                           │
              ┌────────────┴────────────┐
              │                         │
         ┌────▼────┐               ┌────▼────┐
         │  Peer A │◄─────────────►│  Peer B │
         └────┬────┘    WebRTC     └────┬────┘
              │       Direct Mesh       │
              └──────────┬──────────────┘
                         │
                    ┌────▼────┐
                    │  Peer C │
                    └─────────┘
```

The Host still has a direct bootstrap/control connection with each participant, but it is not used as a normal data-plane hop.

## Signaling

PCR does not require a signaling server.

Instead, the initial WebRTC setup is performed by exchanging compact signaling codes manually.

### Host

1. Enter a nickname and optionally a bio.
2. Create a room as Host.
3. Share the generated **Host Code**.
4. Receive a **Peer Offer** from each participant.
5. Verify and add each peer.
6. Send the generated **Host Answer** back to that participant.

### Peer

1. Enter a nickname.
2. Select **Join as Peer**.
3. Paste or scan the Host Code.
4. Verify the Host Code.
5. Generate a **Peer Offer**.
6. Send the Peer Offer to the Host.
7. Receive the Host Answer.
8. Paste or scan the Host Answer.
9. Apply it and establish the WebRTC connection.

QR display and scanning are included for Host Codes, Peer Offers, and Host Answers.

The Host Answer is signature-checked against the Host identity pinned during the Host Code verification step, helping detect a substituted answer during manual signaling. fileciteturn1file7L514-L548

## Networking

PCR uses WebRTC `RTCPeerConnection` and `RTCDataChannel`.

Google's public STUN servers are configured by default:

```text
stun:stun.l.google.com:19302
stun:stun1.l.google.com:19302
stun:stun2.l.google.com:19302
stun:stun3.l.google.com:19302
stun:stun4.l.google.com:19302
```

A custom STUN server can also be supplied by the user. The application sends the configured STUN servers together as ICE servers and lets ICE determine which candidate pair works. fileciteturn1file2L145-L187

### STUN vs TURN

PCR deliberately uses **STUN without TURN**.

STUN helps peers discover reachable network candidates. It does not carry normal chat messages or file contents.

There is no TURN relay fallback in the current design. Therefore, a direct connection is not guaranteed on every network. Restrictive NATs, firewalls, corporate networks, or mobile carrier networks can prevent two participants from establishing a direct path.

This is an intentional architectural trade-off rather than an implementation oversight.

## Security Model

PCR uses several layers of protection.

### WebRTC transport encryption

WebRTC DataChannels are protected by DTLS as part of the WebRTC transport.

### Room-wide application encryption

Room-wide chat and `/me` messages use a room key generated by the Host.

The application derives cryptographic material using:

- ECDH P-256
- HKDF-SHA-256
- AES-256-GCM

The implementation defines a dedicated E2EE domain and derives AES-GCM keys from ECDH shared secrets through HKDF. fileciteturn1file6L440-L496

Because the Host generates and holds the room-wide key, the Host is able to read room-wide messages as a legitimate member of the room. This is important: PCR should **not** be described as a system where the Host is cryptographically blind to all room messages.

### Peer-to-peer DMs

Direct messages between two peers use a pairwise ECDH-derived key.

The Host is not the normal data-path hop for peer-to-peer DMs. The implementation describes these messages as encrypted for the two participating peers and sent over their direct mesh channel. fileciteturn1file0L49-L56

### Host identity

The Host has a persistent ECDSA identity and a room-specific identity fingerprint.

Peers pin the Host public identity while processing the Host Code and use it to verify subsequent Host Answers.

This provides protection against silently replacing the Host Answer during the manual signaling process.

### Replay protection

Incoming message IDs are handled through bounded replay caches. This prevents repeated packets from being processed indefinitely while keeping the cache size bounded.

### Input validation

Data received over WebRTC is treated as untrusted input.

PCR validates packet shapes, message IDs, nicknames, file metadata, chunks, MIME types, filenames, transfer IDs, and cryptographic hashes before using them.

## File Transfer

PCR supports direct file transfer over WebRTC DataChannels.

The current client limits individual files to **4 MB**. File data is split into chunks and sent over the appropriate direct channel.

The implementation uses:

- 4 MB maximum file size
- approximately 12 KB binary chunks
- up to 4 concurrent transfers
- per-channel backpressure
- transfer timeouts
- sender identity binding
- SHA-256 integrity verification

The file-transfer implementation explicitly checks the sender identity associated with a transfer and rejects chunks or completion messages from a different participant. fileciteturn1file1L76-L102

After all chunks arrive, PCR reconstructs the file and verifies its SHA-256 hash before rendering or offering the file to the user. A mismatch causes the file to be rejected. fileciteturn1file3L241-L271

### Per-recipient delivery

File transfers use independent delivery lanes for recipients.

A slow connection therefore does not have to block every other recipient. Each channel respects its own DataChannel backpressure state. fileciteturn1file5L362-L423

## Voice Messages

PCR also supports recording and sending voice messages through the same peer-to-peer data architecture.

Voice transfers are handled as application data rather than requiring a dedicated voice server.

## Chat Features

The interface provides an IRC-inspired experience while using modern browser UI.

Supported features include:

- public room messages
- direct messages
- replies
- emoji reactions
- typing indicators
- voice messages
- file attachments
- user bios
- away status
- operator/voice roles
- message search
- mute controls
- connection state indicators

Participants also have visible mesh connection state, distinguishing direct connectivity from merely being present in the room.

## Diagnostics

PCR includes diagnostics designed specifically for WebRTC troubleshooting.

### `/status`

Shows current room, participant, connection, mesh, and reconnect information.

It can show:

- participant IDs
- mesh connection state
- DataChannel state
- RTCPeerConnection state
- last activity
- reconnect attempts
- queued mesh signals

It also explicitly reports that no TURN relay is being used. fileciteturn1file0L10-L20

### `/bootstrap`

Displays the Host/Peer bootstrap diagnostic trail.

This helps identify whether a failure happened during:

- Host Code processing
- offer generation
- offer acceptance
- signature verification
- ICE connectivity
- DataChannel establishment
- room initialization
- roster/key delivery

The diagnostic trail is intentionally restricted to state and diagnostic information rather than secrets. fileciteturn1file0L25-L45

### `/security`

Displays the application's security/trust model so that users can inspect how the current build actually handles:

- transport encryption
- room encryption
- peer-to-peer DM encryption
- Host visibility
- network architecture

The goal is to make security behavior explicit rather than relying on vague "private" labels. fileciteturn1file0L49-L56

## Reliability

WebRTC connections can disappear because of network changes, NAT behavior, device sleep, browser state, or firewall changes.

PCR therefore tracks mesh connection lifecycle states:

```text
new
  ↓
signaling
  ↓
connecting
  ↓
connected
  ↓
disconnected / failed
  ↓
reconnect / close
```

Reconnect behavior is bounded rather than allowing unlimited attempts.

Mesh signaling is also queued when the bootstrap connection is not ready yet, preventing early mesh negotiation messages from simply being discarded. fileciteturn1file2L202-L208

## Room Passwords

Hosts can optionally protect a room with a password.

The password is not placed directly into the Host Code. Participants who join a protected room must provide the corresponding password during the join process.

## Identity

PCR maintains several forms of identity:

- nickname
- participant ID
- Host identity
- Host fingerprint
- ECDH session keys
- room cryptographic key

The Host identity is particularly important for manual signaling because peers use it to verify that subsequent Host Answers correspond to the Host identity they originally accepted.

## UI

PCR intentionally uses a terminal-inspired visual style:

- monospace typography
- dark interface
- compact status information
- IRC-like room presentation
- terminal-style command input
- desktop window controls
- responsive mobile layout

The application can be minimized or maximized, and the layout adapts to smaller screens.

## Deployment

PCR is designed as a standalone HTML application.

There is no build step and no required backend.

Clone the repository:

```bash
git clone https://github.com/taha8478/PCR.git
cd PCR
```

You can serve it locally with Python:

```bash
python -m http.server 8000
```

Then open:

```text
http://localhost:8000
```

You can also deploy the HTML file to a static hosting service.

Because the application does not require a permanent application backend, static hosting is sufficient for distributing the client.

## Project Structure

The core application is intentionally self-contained:

```text
PCR/
├── PCR.html
└── README.md
```

The current application contains the UI, networking layer, cryptographic logic, signaling flow, file-transfer implementation, diagnostics, and client state in the HTML document.

## Limitations

PCR is deliberately decentralized, but that architecture introduces trade-offs.

### Direct connectivity is not guaranteed

Without TURN, some peers simply cannot establish a direct WebRTC path.

### Mesh scalability

A full mesh requires connections between participants.

For `N` participants, a complete mesh can require approximately:

```text
N × (N - 1) / 2
```

peer-to-peer connections.

This makes PCR more appropriate for small private rooms than very large communities.

### Manual signaling

The current signaling process requires exchanging codes between users.

This removes the need for a signaling backend but makes the initial connection less convenient.

### Host trust

The Host controls the room's control plane and possesses the room-wide encryption key.

Therefore, PCR's trust model is not equivalent to a fully decentralized, zero-trust group messenger.

### No offline messaging

A disconnected participant cannot receive a message through a connection that does not exist.

Persistent offline delivery would require additional storage and synchronization infrastructure.

### File size

The current maximum file size is 4 MB.

This is intentional because file data is carried directly over WebRTC DataChannels and the application is designed to keep memory and transfer state bounded. fileciteturn1file4L190-L200

## Privacy Considerations

"Serverless" does not mean "anonymous."

WebRTC can involve ICE candidates containing network information, and network infrastructure can still observe connection metadata.

PCR does not route normal application data through a central chat server, but users should still consider:

- IP/network exposure
- browser privacy
- local network visibility
- signaling-code handling
- Host trust
- endpoint security
- device compromise

The absence of TURN also means PCR does not hide peer-to-peer network relationships behind a relay.

## Threat Model

PCR is primarily designed to reduce dependence on centralized application infrastructure.

It is **not** presented as a replacement for professionally audited secure messaging software.

The current design provides meaningful protections against several classes of accidental or malicious behavior, including:

- substituted Host Answers
- malformed network packets
- replayed message IDs
- unauthorized file chunks
- corrupted file transfers
- unsafe MIME rendering
- unbounded transfer state

However, the Host remains trusted for room-wide chat and control-plane operations.

If you need protection against a malicious Host, PCR's current room-wide encryption model is not sufficient.

## Technical Details

Current protocol-related constants include:

```text
Application protocol version: 3
Network mode: DIRECT_MESH
Network protocol version: 1
Maximum message length: 2000 characters
Maximum file size: 4 MB
Maximum concurrent transfers: 4
File-transfer timeout: 60 seconds
Replay cache: 500 IDs
```

The implementation also uses bounded DataChannel buffering thresholds to prevent a fast sender from indefinitely filling a slow recipient's browser-side queue. fileciteturn1file4L190-L200

## Testing

For basic testing, use multiple browser instances or devices.

Recommended test matrix:

```text
Browser A ─ Host
Browser B ─ Peer
Browser C ─ Peer
```

Test:

- room creation
- Host Code generation
- Peer Offer generation
- Host Answer generation
- signature verification
- WebRTC connection
- room initialization
- public chat
- peer-to-peer DM
- replies
- reactions
- typing indicators
- voice messages
- file transfers
- multiple simultaneous transfers
- reconnect behavior
- `/status`
- `/bootstrap`
- `/security`

For realistic network testing, test devices on different networks. Testing every participant on the same machine or LAN can hide NAT, firewall, latency, and bandwidth problems.

## Philosophy

PCR follows a simple principle:

> **A private chat room does not necessarily need a chat server.**

Instead of building another centralized messaging backend, PCR uses browser-native WebRTC capabilities to create a direct communication mesh.

The resulting architecture is intentionally small:

```text
                PCR
                 │
        ┌────────┴────────┐
        │                 │
   Control Plane      Data Plane
        │                 │
   Manual Signaling    WebRTC Mesh
        │                 │
        ▼                 ▼
      Host          Peer ↔ Peer
                        │
              ┌─────────┼─────────┐
              ▼         ▼         ▼
             Chat      Files     Voice
             DMs       Typing    Reactions
```

The Host coordinates the room, while participants communicate directly whenever the network permits it.

## License

```text
MIT License
```

## Author

**taha8478**

GitHub: https://github.com/taha8478

PCR is an experimental serverless peer-to-peer communication project focused on direct WebRTC networking, minimal infrastructure, and an inspectable client-side architecture.

## AI-Assisted Development

PCR was developed with the assistance of AI tools. AI was used throughout the development process for code generation, debugging, architectural discussions, security reviews, and iterative improvements.

The final architecture, implementation decisions, testing, and project direction were reviewed and guided by the project author.

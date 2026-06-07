# Networking Protocol Surface

MECE mapping of all relevant protocols by OSI layer with per-ship relationships. Companion reference to DREAM-STATE-ooda.md.

---

## Historical Design Lessons

**1. Trust was never built in.** Every layer was designed trusting — Ethernet assumed all machines on the segment were friendly, IP assumed all packets were honest, SMTP assumed all senders were who they claimed, HTTP was plaintext. Each got exploited, each had security bolted on afterward (ARP spoofing → 802.1X, IP spoofing → IPsec, spam → DKIM/SPF/DMARC, eavesdropping → TLS). The most recent protocols (WireGuard, QUIC, Signal, MLS) were designed encrypted-by-default from day one. **We take: assumption of breach is correct; encrypt-by-default at every layer.**

**2. Decentralization → centralization → re-decentralization.** ARPANET was decentralized by design (survive nuclear attack). The web centralized around DNS registrars, CAs, Google, AWS, Cloudflare. Now blockchain, IPFS, Fediverse, Nostr push back toward decentralization. **We take: federated architecture — each ship autonomous, optional federation via the primitive whiteboard. Neither fully centralized nor fully decentralized.**

**3. Protocol ossification.** TCP/IP won not because it was best but because it shipped and worked. IPv6 took 25+ years to reach ~40% adoption. HTTP/1.1 lasted 18 years before HTTP/2. **We take: design for protocol evolution, not protocol perfection. MCP is our session protocol today; it won't be forever. The harness should be protocol-agnostic at the communication layer.**

**4. The session collapse.** OSI layers 5 and 6 never got clean protocol boundaries in practice — TCP/IP collapsed them into the application layer. But the *functions* (session management, data encoding) still exist; they just live inside application protocols. **We take: don't over-engineer layer boundaries. Our architecture already does this — MCP handles session, encoding, and application concerns in one protocol.**

**5. Identity is the hardest problem.** Kerberos (1988) → RADIUS (1991) → LDAP (1993) → SAML (2002) → OAuth 2.0 (2012) → OIDC (2014) → WebAuthn (2018) → Passkeys (2022). 34 years, still no universal solution. Each trades convenience vs. security vs. decentralization. **We take: identity is a spectrum, not a binary. The trust model already accounts for this — earned autonomy, not assumed identity.**

**6. The overlay pattern.** When the base layer can't be changed, build on top: Tor over TCP/IP, IPFS over HTTP, Tailscale over WireGuard over UDP, service mesh over Kubernetes. **We take: our architecture IS an overlay — the harness overlays Claude Code which overlays the OS. Each layer independently useful (composability principle).**

**7. Convergence toward "encrypted everywhere."** Separate encryption at each layer (IPsec, TLS, WPA, Signal) is converging toward encryption as invisible infrastructure. QUIC encrypts transport. WireGuard encrypts the tunnel. MLS encrypts group messaging. The endpoint — not individual protocols — becomes the trust boundary. **We take: the five encryption layers already in the deferred log are the right model. Encryption shouldn't be a per-protocol decision; it should be a per-layer guarantee.**

---

## Layer 1 — Physical

| Protocol/Tech | Year | What it does | Historical significance | What we take |
|---|---|---|---|---|
| **Telegraph** | 1844 | Electrical signals over wire | First long-distance instant communication; proved information could move faster than matter | The original "message passing" — every protocol since is a refinement |
| **Telephone** | 1876 | Voice over copper | Circuit-switched: dedicated path held open for the duration. Worked for voice, terrible for data | Circuit-switching is our "dedicated agent" model; packet-switching is our "shadow clone" model |
| **Coaxial cable** | 1880s | Shielded signal transmission | Higher bandwidth than twisted pair; still the backbone of cable internet | Shielding = isolation. Physical isolation predates all software isolation |
| **Fiber optics** | 1970s | Light pulses through glass | Orders of magnitude more bandwidth; physically harder to tap than copper (light doesn't leak electromagnetically) | Physical security matters — fiber tapping requires physical access and leaves evidence |
| **Wi-Fi / 802.11** | 1997 | Radio-based local networking | Traded physical security for convenience — radio signals are inherently broadcastable | Wireless = assume interception. Every wireless channel needs encryption (WPA3) |
| **5G/cellular** | 2019 | High-bandwidth mobile radio | Low latency enables real-time agent communication from any location | Mobile pilot → mobile Observe. Voice surface over cellular |
| **Starlink/LEO satellite** | 2020 | Low-orbit satellite internet | Global coverage, lower latency than geostationary. SpaceX/xAI compute-in-space thesis | Connects to compute infrastructure trajectory (deferred). Tier 4 is space-based |
| **QKD** | Experimental | Quantum key distribution | Physics-guaranteed eavesdropping detection — measuring a quantum state changes it | Far future. Post-quantum cryptography is the nearer-term concern |

**Ship relationships:** All ships use whatever physical medium is available (transparent). No ship directly controls physical layer. Dreamtime's always-on requirement means redundant physical connectivity matters more.

---

## Layer 2 — Data Link

| Protocol | Year | What it does | Historical significance | What we take |
|---|---|---|---|---|
| **Ethernet** | 1973 | Frames on local network | Won the LAN war (vs. Token Ring, FDDI). CSMA/CD collision handling — multiple talkers, one wire | Collision detection is a coordination primitive. Our merge queue is the same problem |
| **ARP** | 1982 | Maps IP → MAC address | Completely trusting — any device can claim any IP. ARP spoofing is trivial | Trust-by-default fails. ARP poisoning is the canonical example |
| **Wi-Fi/WPA3** | 1997/2018 | Wireless LAN security | WEP → WPA → WPA2 → WPA3. Each generation broken, replaced. WPA3 uses SAE (simultaneous authentication of equals) — no more PSK handshake vulnerabilities | Security is iterative, not one-shot. Design for upgradability |
| **MACsec / 802.1AE** | 2006 | Encrypted Ethernet frames | Encrypts everything at the link layer — even ARP. "IPsec but for your LAN" | Relevant for harness instances on the same physical network. Local != trusted |
| **LoRa/LoRaWAN** | 2015 | Long-range low-power IoT radio | Kilometers of range at tiny bandwidth. For sensors, not data | IoT/Robotics loop (deferred). Sensor data ingestion |
| **Bluetooth/BLE** | 1998 | Short-range device pairing | Personal area network. Constantly discovered new attack surfaces | Relevant for hardware key auth (FIDO2) and IoT peripherals |

**Ship relationships:** Mostly transparent. MACsec relevant for servicegrid (multi-machine local network between three engineers). LoRa relevant for pilgrimage if physical game installations.

---

## Layer 3 — Network

| Protocol | Year | What it does | Historical significance | What we take |
|---|---|---|---|---|
| **IPv4** | 1981 | Packet addressing and routing | The addressing scheme of the internet. 4.3 billion addresses seemed infinite; ran out | Design for scale you can't imagine yet. We will need more address space than we think |
| **IPv6** | 1998 | Extended addressing (3.4x10^38 addresses) | 28 years old, still not universal. The canonical example of protocol ossification | Don't count on protocol transitions happening quickly. Design the harness to work on both |
| **ICMP** | 1981 | Network diagnostics (ping, traceroute) | The internet's heartbeat check. Also weaponized (ping of death, ICMP floods) | Diagnostic capability = attack surface. Our heartbeat protocol has the same dual nature |
| **IPsec** | 1995 | Encrypted IP tunnels | First attempt at network-layer encryption. Complex, heavyweight, but universal | The "right idea, wrong ergonomics" pattern. WireGuard replaced it by being simpler |
| **BGP** | 1989 | Inter-network routing | The glue between ISPs. Trusting — route hijacking is still a live threat. One wrong announcement can redirect internet traffic through a hostile network | Understanding BGP matters defensively — route hijacking can redirect agent API traffic |
| **NAT** | 1994 | Address translation (private → public IP) | A hack that became infrastructure. Broke the end-to-end principle but saved IPv4 | NAT traversal is a real problem for inter-harness communication. Tailscale solves this |

**Ship relationships:** All ships use IP transparently. IPsec/NAT traversal relevant for inter-ship mesh. BGP understanding defensive for all ships.

---

## Layer 4 — Transport

| Protocol | Year | What it does | Historical significance | What we take |
|---|---|---|---|---|
| **TCP** | 1974 | Reliable ordered delivery | The workhorse. Three-way handshake, congestion control, retransmission. Optimized for reliability over latency | Reliability > speed for most harness operations (API calls, artifact sync, MCP) |
| **UDP** | 1980 | Unreliable fast delivery | "Send and pray." No handshake, no ordering, no retransmission. Optimized for latency over reliability | Voice surface, DNS queries, real-time game state (pilgrimage) |
| **QUIC** | 2012/2021 | Encrypted UDP with TCP reliability | Google built it, IETF standardized it. 0-RTT connection establishment. Multiplexed streams without head-of-line blocking. TLS 1.3 built in — can't do unencrypted QUIC | The "encrypt by default" model. HTTP/3 runs on QUIC. Navi already uses it implicitly |
| **SCTP** | 2000 | Multi-stream reliable delivery | Better than TCP for many use cases but never adopted widely. Multi-homing (survive interface failover) | Good ideas don't win by being good — they win by shipping. TCP's installed base was insurmountable |
| **DTLS** | 2006 | TLS for UDP | Encrypted real-time streams. Used by WebRTC, VoIP | Voice surface encryption (SRTP uses DTLS for key exchange) |

**Ship relationships:** All ships use TCP and UDP. QUIC increasingly universal. DTLS relevant for any ship with voice surface. SCTP a cautionary tale, not a protocol to adopt.

---

## Layer 5 — Session (collapsed into application in practice)

| Protocol | Year | What it does | Historical significance | What we take |
|---|---|---|---|---|
| **NetBIOS** | 1983 | Session naming and management | Microsoft networking backbone for decades. Broadcast-based, terrible security | Legacy protocols persist far longer than expected. Plan for coexistence |
| **RPC** | 1984 | Remote procedure calls | The idea that a network call should look like a local function call. Spawned CORBA, DCOM, Java RMI, gRPC | The abstraction that won. MCP tool calls ARE RPC |
| **SOCKS** | 1992 | Generic proxy protocol | Routes any TCP/UDP through a proxy. Tor uses SOCKS. Simple, composable | Composable proxying. Relevant if Navi routes traffic through Tor |
| **WebSocket** | 2011 | Persistent bidirectional channel over HTTP | Solved the "server can't push to client" problem. HTTP was request/response only | MCP uses this. Slack socket mode uses this. The persistent connection model |
| **gRPC** | 2015 | Modern RPC with Protocol Buffers | Google's replacement for REST when performance matters. Streaming, strong typing, code generation | Inter-harness communication candidate. More efficient than REST for high-frequency coordination |
| **MCP** | 2024 | Model Context Protocol | Anthropic's protocol for tool/resource access. JSON-RPC over stdio or SSE. The harness's native session protocol | Our Layer 5. The session protocol that connects Navi to tools, resources, and other agents |
| **A2A** | 2025 | Agent-to-Agent protocol (Google) | Agent discovery and communication standard. Complementary to MCP (which is agent-to-tool) | Agent-to-agent communication. The inter-harness protocol candidate alongside MCP |

**Ship relationships:** All ships use MCP and WebSocket. gRPC relevant for high-throughput inter-harness communication. A2A relevant for agentic mesh (deferred). SOCKS relevant for Tor routing (dreamtime, superorganism research).

---

## Layer 6 — Presentation (collapsed into application in practice)

| Protocol/Format | Year | What it does | Historical significance | What we take |
|---|---|---|---|---|
| **ASN.1** | 1984 | Abstract data encoding | The original "how do you serialize structured data?" Still used in X.509 certificates | Formal encoding standards outlive everything. X.509 is 36 years old |
| **XML** | 1996 | Human-readable structured data | Ruled enterprise for a decade. SOAP, XSLT, XPath. Verbose but powerful schema validation | Over-engineering kills adoption. JSON won by being simpler |
| **JSON** | 2001 | Lightweight structured data | Replaced XML everywhere. No schema by default. JavaScript-native | Our primary data format. MCP uses JSON-RPC. Simple wins |
| **Protocol Buffers** | 2008 | Binary serialization with schema | Google's answer to "JSON is too slow." Schema-first, code-generated, compact | High-throughput inter-harness communication. Pairs with gRPC |
| **MessagePack** | 2008 | Binary JSON | JSON semantics, binary encoding. Faster parsing, smaller payloads | Alternative to Protobuf for cases where schema-first is overkill |
| **CBOR** | 2013 | Concise Binary Object Representation | IETF standard binary encoding. Used in WebAuthn, COSE | Relevant for IoT and constrained environments |
| **TLS handshake** | 1999/2018 | Negotiates encryption parameters | TLS 1.3 (2018) dramatically simplified — 1-RTT, removed insecure options. The "simplify by deleting" approach | TLS 1.3 is the encrypt-by-default model. Navi uses it on every HTTPS call |
| **Compression (gzip/brotli/zstd)** | 1992/2015/2016 | Data compression | Each generation: better ratio, faster. Zstandard is the current leader | Context window is the harness's bandwidth constraint. Compression thinking applies to context engineering |

**Ship relationships:** All ships use JSON and TLS. Protocol Buffers / gRPC relevant for high-throughput inter-ship communication. Compression thinking universal.

---

## Layer 7 — Application

### Web / API

| Protocol | Year | What it does | Historical significance | What we take |
|---|---|---|---|---|
| **HTTP/1.0** | 1996 | Request-response web | One request per connection. Simple, wasteful | The original agent-API interaction model |
| **HTTP/1.1** | 1997 | Persistent connections, pipelining | Keep-alive, chunked transfer. Lasted 18 years as the standard | Persistence matters — keep connections alive |
| **HTTP/2** | 2015 | Multiplexed streams, server push | Multiple requests on one connection. Binary framing. Header compression | Multiplexing = parallel agent requests on one connection |
| **HTTP/3** | 2022 | HTTP over QUIC | Encrypted by default, no head-of-line blocking, faster connection setup | The future of all HTTP. Navi uses it where available |
| **REST** | 2000 | Resource-oriented API design | Stateless, cacheable, uniform interface. Won the API war vs. SOAP | Most APIs Navi talks to are REST |
| **GraphQL** | 2015 | Query-shaped API responses | Client specifies what it wants. Reduces over-fetching. Single endpoint | Relevant for complex data queries (knowledge graph API?) |
| **WebHooks** | ~2007 | Server-to-server event notification | "Don't call us, we'll call you." Reversed the polling model | Gate resolution (GitHub webhooks for PR merge, CI pass) |
| **SSE** | 2006 | Server-sent events | One-way streaming from server. Simpler than WebSocket for push-only | MCP SSE transport. Execution log streaming |

### Remote Access

| Protocol | Year | What it does | Historical significance | What we take |
|---|---|---|---|---|
| **Telnet** | 1969 | Remote terminal, plaintext | The original remote access. Completely unencrypted | The "before security" baseline. Everything Telnet does, SSH does encrypted |
| **SSH** | 1995 | Encrypted remote terminal + tunneling | Replaced Telnet, rlogin, rsh. Key-based auth. Port forwarding. File transfer (SCP/SFTP) | Navi's primary remote access. Git over SSH. Remote command execution |
| **Mosh** | 2012 | Mobile-friendly SSH | Handles intermittent connectivity, roaming. UDP-based | Relevant for mobile pilot scenario — SSH that survives network transitions |

### DNS

| Protocol | Year | What it does | Historical significance | What we take |
|---|---|---|---|---|
| **DNS** | 1983 | Name → address resolution | Hierarchical, distributed, cached. The internet's phone book. Originally plaintext | Navi uses it constantly. Understanding DNS is understanding internet topology |
| **DNSSEC** | 2005 | Authenticated DNS responses | Proves the response really came from the authoritative server. Prevents DNS spoofing | Defense against DNS poisoning attacks on agent API calls |
| **DoH** | 2018 | DNS over HTTPS | Encrypts DNS queries. Your ISP can't see what you're looking up | Privacy for agent research activity. Prevents DNS-level surveillance |
| **DoT** | 2016 | DNS over TLS | Same goal as DoH, different mechanism (dedicated port vs. blending with HTTPS) | Alternative to DoH. Either works for encrypted DNS |

### Email

| Protocol | Year | What it does | Historical significance | What we take |
|---|---|---|---|---|
| **SMTP** | 1982 | Send email | The store-and-forward model. Originally completely trusting — anyone could send as anyone | Notification surface for harness. Also: SMTP's trust model is the canonical cautionary tale |
| **IMAP** | 1986 | Read email (server-side) | Emails stay on server, accessible from multiple clients | If the harness reads email (inbox processing), IMAP is the protocol |
| **POP3** | 1984 | Read email (download) | Download and delete. Simple but single-device | Mostly superseded by IMAP |
| **DKIM** | 2004 | Cryptographic email signing | Domain signs outgoing email; recipient verifies. Proves email really came from that domain | If Navi sends email, DKIM proves it came from the right domain |
| **SPF** | 2006 | Authorized sender IPs | DNS record listing which IPs can send email for a domain | Complement to DKIM. Together they solve email authentication |
| **DMARC** | 2012 | Policy enforcement for DKIM/SPF | Tells receiving servers what to do when DKIM/SPF fails (reject, quarantine, report) | The policy layer on top of authentication. Matches our gate model |
| **STARTTLS** | 1999 | Upgrade plaintext to encrypted | Opportunistic encryption — try TLS, fall back to plaintext if not supported | The "upgrade" pattern — start simple, negotiate up |
| **JMAP** | 2019 | Modern email API | JSON-based replacement for IMAP. Designed for modern clients | If building email integration, JMAP > IMAP |

### File Transfer

| Protocol | Year | What it does | Historical significance | What we take |
|---|---|---|---|---|
| **FTP** | 1971 | File transfer | The oldest application protocol still in use. Plaintext credentials, dual-channel (command + data) | Legacy. SFTP replaced it |
| **SFTP** | 1997 | Encrypted file transfer over SSH | Solved FTP's security problems by running over SSH | Artifact transfer between harness instances |
| **rsync** | 1996 | Differential file sync | Only transfers changed blocks. The delta-sync model | Artifact synchronization. Only sync what changed |
| **rclone** | 2014 | Multi-cloud file sync | rsync for cloud storage (S3, GCS, Azure, etc.) | If artifacts live in cloud storage |

---

## Cross-Layer: Authentication & Identity

| Protocol | Year | What it does | Historical significance | What we take |
|---|---|---|---|---|
| **Passwords** | Ancient | Something you know | The original authentication. Terrible — reuse, phishing, brute force. Still 80%+ of auth | The baseline that everything else tries to replace |
| **Kerberos** | 1988 | Ticket-based auth | Prove identity once, get tokens for services. Still backbone of Active Directory. Mutual auth | The "authenticate once, authorize many" model. Navi's OAuth tokens work similarly |
| **RADIUS** | 1991 | Centralized network auth | "Can this user/device connect to the network?" Dial-up, Wi-Fi, VPN auth | Relevant if servicegrid controls network access for customers |
| **LDAP** | 1993 | Directory services | Centralized identity store. AD runs on LDAP. Hierarchical (OU structure) | Enterprise identity. Servicegrid may need to integrate |
| **X.509 / PKI** | 1988 | Certificate infrastructure | The CA trust chain behind HTTPS. CAs sign certs proving "this server is who it says" | Navi needs to understand and manage certificates for deployed services |
| **SAML** | 2002 | Enterprise SSO (XML) | Enterprise single sign-on. "I authenticated with my company; let me into your app" | Servicegrid customer integration |
| **OAuth 2.0** | 2012 | Delegated authorization | "Let this app act on my behalf." The token-based model that powers all modern API auth | Navi's primary auth mechanism for APIs. GitHub, Vercel, Slack all use OAuth |
| **OIDC** | 2014 | Identity layer on OAuth | "Who is this user?" on top of "what can they do?" JSON-based (vs. SAML's XML) | Modern identity. Servicegrid user auth |
| **JWT** | 2015 | Signed tokens | Self-contained, verifiable claims. No server-side session needed | Inter-harness auth tokens. Agent identity claims |
| **FIDO2/WebAuthn** | 2018 | Hardware-backed auth | Public key auth with hardware keys/biometrics. Phishing-resistant by design | Pilot authentication for high-trust operations (deploy approval, gate decisions) |
| **Passkeys** | 2022 | Platform-native WebAuthn | WebAuthn made seamless — synced across devices via platform (iCloud, Google) | The end state for human auth. Pilot uses passkeys |
| **mTLS** | (TLS feature) | Mutual certificate auth | Both client AND server prove identity. Not just "is this server real?" but "is this client authorized?" | **Inter-harness authentication.** How superorganism proves identity to servicegrid and vice versa |
| **Certificate Transparency** | 2013 | Public certificate log | All issued certs are public so you can detect misissued ones. Google's response to CA compromise | Defense — detect if someone gets a cert for your domain |
| **ACME / Let's Encrypt** | 2015 | Automated cert issuance | Free, automated HTTPS certificates. Single biggest reason HTTPS became universal | Navi auto-provisions certs for deployed services |

---

## Cross-Layer: Encryption

| Protocol | Year | What it does | Historical significance | What we take |
|---|---|---|---|---|
| **DES** | 1977 | Block cipher (56-bit) | First federal encryption standard. Broken by brute force in 1998 | Encryption has a shelf life. Design for algorithm agility |
| **PGP/GPG** | 1991 | Email + file encryption | Web of trust (vs. CA hierarchy). Famously hard to use | Artifact signing. Git commit signing. The UX lesson: security that's hard to use doesn't get used |
| **AES** | 2001 | Block cipher (128/256-bit) | Replaced DES. Still unbroken. The workhorse of symmetric encryption | AES-256 is our at-rest encryption standard (via FileVault/LUKS) |
| **RSA** | 1977 | Public key encryption | The original asymmetric algorithm. Made secure key exchange possible | Being replaced by elliptic curve. Still in wide use |
| **Elliptic Curve (ECDH, ECDSA)** | 2004 | Compact public key crypto | Same security as RSA at much smaller key sizes. X25519 for key exchange, Ed25519 for signing | Our modern crypto primitive. WireGuard, Signal, age all use it |
| **NaCl/libsodium** | 2008/2013 | Misuse-resistant crypto library | "You can't mess this up" — safe defaults, no knobs to turn wrong. djb designed it | Design philosophy: make the safe path the easy path. Applies to harness design broadly |
| **ChaCha20-Poly1305** | 2008 | Stream cipher + MAC | Djb's alternative to AES. Faster in software (no hardware acceleration needed). Used by WireGuard, TLS 1.3 | The "software-first" cipher. Relevant for non-hardware-accelerated environments |
| **Signal Protocol / Double Ratchet** | 2013 | Forward-secret E2E messaging | Every message gets a new key. Compromising one key doesn't reveal past OR future messages | Forward secrecy model. If inter-harness messaging is compromised, limit blast radius |
| **age/rage** | 2019 | Modern file encryption | Replacement for GPG for simple cases. One tool, one job, no complexity | Artifact encryption. Simple, correct, auditable |
| **MLS** | 2023 | Group messaging encryption at scale | IETF standard. Scales to large groups (Signal Protocol doesn't scale well beyond ~100) | Multi-harness group communication. When 4+ ships need encrypted group channel |
| **Post-quantum (Kyber, Dilithium)** | 2022 | Quantum-resistant crypto | NIST standardized. Protect against "harvest now, decrypt later" quantum attacks | Future-proofing. TLS 1.3 already supports hybrid post-quantum key exchange |

---

## Cross-Layer: Disk & File Encryption

| Protocol | Year | What it does | Ship relationship |
|---|---|---|---|
| **FileVault** | 2003 | macOS full-disk encryption | All ships (macOS) — transparent |
| **LUKS/dm-crypt** | 2004 | Linux full-disk encryption | All ships (Linux servers) — transparent |
| **BitLocker** | 2006 | Windows full-disk encryption | Not relevant (no Windows ships) |
| **age** | 2019 | Individual file encryption | All ships — artifact encryption before sync/federation |
| **GPG** | 1991 | File + email encryption, signing | All ships — git commit signing, artifact provenance |

---

## Overlay Networks

### Anonymity & Censorship Resistance

| Network | Year | What it does | Historical significance | What we take |
|---|---|---|---|---|
| **Freenet** | 2000 | Distributed encrypted data store | First "you can't know what you're hosting." Content addressed by key, not location | Content-addressed storage. The ancestor of IPFS's approach |
| **Tor** | 2002 | Onion routing (3-hop anonymity) | No single node knows both who you are AND what you're accessing. Circuit-based | **Anonymous research capability.** When Navi needs to research without attribution. Dreamtime's autonomous research surface |
| **I2P** | 2003 | Garlic routing (network-internal services) | Optimized for services WITHIN the network, not accessing clearnet. Packet-based, not circuit-based | Potentially relevant for hidden harness services. More niche than Tor |

### VPN & Mesh

| Network | Year | What it does | Historical significance | What we take |
|---|---|---|---|---|
| **OpenVPN** | 2001 | TLS-based VPN | The workhorse VPN for 20 years. ~100,000 lines of code. Complex config | Being replaced by WireGuard. Still widely deployed |
| **WireGuard** | 2018 | Minimal VPN (~4,000 LOC) | Single round-trip handshake. Cryptographically opinionated (no negotiation). ~100x simpler than OpenVPN | **The inter-harness tunnel protocol.** Simple, fast, auditable. The "lean by default" approach applied to VPN |
| **Tailscale** | 2019 | Mesh VPN over WireGuard | Every device connects to every other device. NAT traversal solved. Coordination plane + WireGuard data plane | **The inter-ship mesh.** Superorganism, servicegrid, pilgrimage, dreamtime all on one Tailscale network. Each ship sees every other as if on the same LAN. The "four ships as one network" implementation |
| **Nebula** | 2019 | Mesh VPN (Slack/Defined) | Open-source Tailscale alternative. No coordination server required (fully decentralized) | Alternative to Tailscale if full decentralization is preferred |

### Content Distribution & P2P

| Network | Year | What it does | Historical significance | What we take |
|---|---|---|---|---|
| **CDN** | ~2000 | Edge caching | Cache content near users. Cloudflare, Akamai, Fastly. Most of the web sits behind one | Relevant for servicegrid's public-facing services. Vercel includes CDN |
| **BitTorrent** | 2001 | Decentralized file distribution | Download from many peers simultaneously. Piece-based. DHT for peer discovery | Artifact distribution at scale. If artifacts are large (models, datasets) |
| **IPFS** | 2015 | Content-addressed storage | Files identified by hash (CID), fetched from any node. Decentralized web layer. Persistent via Filecoin pinning | **Artifact storage and sharing.** Hash-based IDs (already encoded in sovereign data hub). Artifacts identified by content, not location. Natural federation primitive |
| **Hypercore/Dat** | 2013 | Append-only distributed log | Merkle tree of changes. Sync between peers. Each peer has full history | Could model run archives as append-only distributed logs |

### Decentralized / Blockchain

| Network | Year | What it does | Historical significance | What we take |
|---|---|---|---|---|
| **Bitcoin** | 2009 | Decentralized value transfer | Proof of work, trustless consensus. Solved the double-spend problem without a central authority | The "trustless" model. Agent economics (deferred). x402 payment protocol |
| **Ethereum** | 2015 | Programmable blockchain (smart contracts) | Turing-complete contracts. DeFi, NFTs, DAOs. Proved you can encode arbitrary logic on-chain | Smart contracts for agent-to-agent agreements. Tokenized agent ownership (Virtuals Protocol, deferred) |
| **Lightning Network** | 2018 | Bitcoin payment channels | Off-chain transactions, on-chain settlement. Micro-transactions feasible | Agent micro-payments — pay per API call, per artifact, per compute unit |
| **ActivityPub** | 2018 | Federated social protocol | Mastodon, Pixelfed, PeerTube. Your server talks to other servers. You own your data | **Federated harness communication model.** Each ship is a server; they federate via a standard protocol |
| **Nostr** | 2020 | Minimalist decentralized social | Cryptographic identity (public/private keys), relay-based message distribution. No accounts, no servers | Simpler than ActivityPub. Agent identity IS a key pair. Natural for agent communication |
| **AT Protocol / Bluesky** | 2023 | Portable social protocol | Account portability — move your identity between servers. Data repos per user | Account portability maps to harness instance portability — move your navigator between machines |

### Service Mesh (Modern Infrastructure)

| Network | Year | What it does | Historical significance | What we take |
|---|---|---|---|---|
| **Linkerd** | 2016 | Service mesh (lightweight) | Sidecar proxy for every service. Automatic mTLS, observability, load balancing | Lightweight inter-service encryption. Relevant at scale |
| **Istio** | 2017 | Service mesh (full-featured) | Control plane + data plane. Traffic management, security, observability | The "orchestrator for services" — analogous to what we're building for agents |
| **Consul Connect** | 2018 | Service mesh + discovery | Service discovery + mesh networking. HashiCorp ecosystem | Service discovery maps to capability advertisement (agentic mesh, deferred) |

### Agent-Specific (Emerging)

| Protocol | Year | What it does | What we take |
|---|---|---|---|
| **MCP** | 2024 | Agent-to-tool protocol | Our native protocol. Tool discovery, resource access, prompt templates |
| **A2A** | 2025 | Agent-to-agent protocol (Google) | Agent discovery and communication. Complementary to MCP |
| **Agent Skills** | 2025 | Skill portability standard (agentskills.io) | Already adopted for skill format |
| **x402** | 2025 | Agent payment protocol (Coinbase) | Machine-to-machine payments. Already in agent economics (deferred) |

---

## Per-Ship Protocol Map

### Universal Baseline (ALL ships)

| Layer | Protocols |
|---|---|
| Physical–Network | IP (transparent), TCP, UDP, QUIC |
| Session | MCP, WebSocket |
| Presentation | JSON, TLS 1.3 |
| Application | HTTP/HTTPS, SSH, DNS (DoH), SSE |
| Auth | OAuth 2.0, JWT, mTLS (inter-harness), ACME/Let's Encrypt |
| Encryption | AES-256 at rest (FileVault/LUKS), TLS 1.3 in transit, age (artifact encryption), GPG (commit signing) |
| Mesh | Tailscale (inter-ship), WireGuard (underlying) |
| Agent | MCP, Agent Skills |

### Superorganism (one pilot + internet, navigator: Navi)

All universal baseline, plus:

| Protocol | Why |
|---|---|
| SRTP/DTLS | Voice surface encryption (dream-protocol sessions carry privileged context) |
| Tor/SOCKS | Anonymous research capability — research-protocol without attribution |
| IPFS | Content-addressed artifact sharing with other ships |
| DoH | Encrypted DNS for privacy during research |
| Mosh | Mobile pilot — SSH that survives network transitions |
| x402 | Agent economics — Navi transacting autonomously |
| Nostr | Lightweight decentralized identity + communication |
| gRPC/Protobuf | High-throughput communication with servicegrid |

### Servicegrid (three humans + customers + customer agents)

All universal baseline, plus:

| Protocol | Why |
|---|---|
| OIDC | Customer identity management |
| SAML | Enterprise customer SSO integration |
| Kerberos/LDAP | Enterprise directory integration (if customers require it) |
| SMTP + DKIM/SPF/DMARC | Customer email communication with provenance |
| JMAP | Modern email API for inbox processing |
| MLS | Encrypted group messaging across three engineers + customer agents |
| CDN (Cloudflare) | Public-facing service performance |
| GraphQL | Complex customer-facing data queries |
| mTLS service mesh | Inter-service authentication at scale |
| WebHooks | Customer event notifications, gate resolution |
| FIDO2/WebAuthn | High-security customer operations (financial, compliance) |
| Stripe SPT + x402 | Dual payment rails — traditional (customers) + crypto (agents) |
| ActivityPub | Federated communication with customer agent ecosystems |

### Pilgrimage (game design)

All universal baseline, plus:

| Protocol | Why |
|---|---|
| UDP + custom game protocol | Real-time game state synchronization |
| WebSocket | Persistent game connections |
| WebRTC / SRTP | Voice chat between players |
| CDN | Game asset distribution |
| IPFS / BitTorrent | Decentralized game distribution |
| OAuth 2.0 / OIDC | Player identity |
| Ethereum / smart contracts | In-game economies, ownership (if applicable) |
| LoRa | Physical game installations (IoT sensors) |

### Dreamtime (fully autonomous)

All universal baseline, plus **everything superorganism has**, plus:

| Protocol | Why |
|---|---|
| Tor (always-on) | Autonomous research without attribution — no human to decide when privacy matters |
| I2P | Hidden services within the autonomous network |
| Full Tailscale mesh | Active communication with all other ships |
| MLS | Group encrypted communication with all ships simultaneously |
| Nostr / ActivityPub | Autonomous social presence — dreamtime publishes discoveries |
| Blockchain (Ethereum/Lightning) | Autonomous value exchange — dreamtime transacts without human approval |
| IPFS (pinned) | Persistent artifact sharing — dreamtime's artifacts must survive without human curation |
| Post-quantum crypto | Future-proofing — dreamtime runs longest without human intervention, most exposure to harvest-now-decrypt-later |
| Certificate Transparency monitoring | Autonomous security posture — detect certificate anomalies without human review |

---

## Available to Add as Needed (not ship-specific)

These protocols are in the menu but not assigned to any ship by default — they activate when a specific need arises:

| Protocol | Activates when |
|---|---|
| IPsec | Direct tunnel needed (non-Tailscale environment) |
| Nebula | Tailscale alternative needed (fully decentralized mesh) |
| SCTP | Multi-homing needed (survive interface failover) |
| rsync / rclone | Large artifact sync between ships |
| Matrix/Olm | Decentralized messaging alternative to MLS |
| Hypercore | Append-only distributed log for run archives |
| OpenVPN | Legacy VPN integration required |
| Consul Connect | Service discovery at multi-service scale |
| AT Protocol | Portable identity across harness instances |
| PGP/S/MIME | Encrypted email (if email becomes a primary surface) |

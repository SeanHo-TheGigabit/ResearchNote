# Linux Kernel Networking Glossary

A comprehensive glossary of networking terms explained at multiple levels with real-world scenarios.

## How to Use This Glossary

Each term is explained at multiple levels:
- 🌟 **Simple**: For beginners (ages 6-12)
- 📚 **Student**: For learners (ages 13-18)
- 🎓 **Technical**: For practitioners (university level)
- 🔬 **Advanced**: For experts (master's/PhD level)
- 🎬 **Scenario**: Real-world use cases and examples

---

## A

### ACK (Acknowledgment)

**🌟 Simple**: When you receive a letter, you send a postcard back saying "I got it!" That's an ACK.

**📚 Student**: A message sent by the receiver to tell the sender "I successfully received your data." Like a delivery confirmation.

**🎓 Technical**: In TCP, an acknowledgment packet confirming receipt of data. Contains an acknowledgment number indicating the next expected sequence number.

**🔬 Advanced**: TCP ACK packets can be cumulative (acknowledging all data up to a point), selective (SACK for specific ranges), or delayed (piggybacking on return data to reduce packet count).

**🎬 Scenario**:
- **Web browsing**: Your browser sends ACK for each chunk of webpage data received
- **File download**: Every few KB downloaded, ACK confirms receipt, allowing sender to continue
- **Video streaming**: Continuous ACKs ensure smooth playback without data loss
- **Debugging**: Missing ACKs indicate network problems or packet loss

---

### ARP (Address Resolution Protocol)

**🌟 Simple**: Like asking "Who lives at house number 5?" and someone answers "That's Alice!" ARP finds who owns an address.

**📚 Student**: A protocol that finds the hardware (MAC) address for a given IP address on your local network. Like looking up someone's phone number from their name.

**🎓 Technical**: Network layer protocol that maps IP addresses to MAC addresses. Broadcasts "Who has IP x.x.x.x?" and the device with that IP responds with its MAC address. Responses are cached in the ARP table.

**🔬 Advanced**: ARP operates at layer 2/3 boundary. Uses broadcast frames (FF:FF:FF:FF:FF:FF). Vulnerable to ARP spoofing/poisoning attacks. Gratuitous ARP used for duplicate detection and cache updates. Proxy ARP enables routing between subnets.

**🎬 Scenario**:
- **First packet**: Computer doesn't know neighbor's MAC, sends ARP request, caches result
- **Network troubleshooting**: `arp -a` shows MAC-to-IP mappings, helps identify device conflicts
- **Security attack**: Attacker sends fake ARP replies to intercept traffic (man-in-the-middle)
- **IP address conflict**: Gratuitous ARP detects if another device has same IP

---

## B

### Backlog

**🌟 Simple**: A waiting line of packages that haven't been opened yet.

**📚 Student**: A queue where incoming packets wait when they arrive faster than they can be processed.

**🎓 Technical**: In sockets, the backlog is the queue of pending connections (for listen sockets) or packets waiting to be processed. Configured via `listen(sockfd, backlog)` system call.

**🔬 Advanced**: Multiple backlog types: listen backlog (SYN queue + accept queue), per-socket receive backlog (when socket is locked), per-CPU input_pkt_queue. Backlog exhaustion leads to packet drops (SYN cookies mitigate SYN flood attacks).

**🎬 Scenario**:
- **Web server**: Backlog=128 means 128 pending connections can wait while server handles current clients
- **DDoS attack**: Attacker floods SYN packets to fill backlog, preventing legitimate connections
- **High load**: Process locked doing computation, packets accumulate in backlog until unlocked
- **Tuning**: Increasing backlog helps handle traffic bursts without dropping connections

---

### BBR (Bottleneck Bandwidth and RTT)

**🌟 Simple**: A smart system that figures out how fast to send data so nothing gets stuck in traffic.

**📚 Student**: A modern way of controlling how fast data is sent, based on measuring how much the network can handle and how long it takes for responses.

**🎓 Technical**: TCP congestion control algorithm developed by Google that estimates bottleneck bandwidth and round-trip time to optimize throughput and latency. Unlike loss-based algorithms (Reno, Cubic), BBR is model-based.

**🔬 Advanced**: BBR operates in probe/cruise cycles, using a state machine (STARTUP, DRAIN, PROBE_BW, PROBE_RTT). Measures delivery rate and RTT to build network model. Achieves higher throughput and lower latency than Cubic, especially on lossy networks. Mainlined in Linux 4.9.

**🎬 Scenario**:
- **WiFi networks**: BBR handles random packet loss better than traditional algorithms
- **Long-distance connections**: Optimizes throughput for high latency links (e.g., satellite)
- **YouTube/Google services**: BBR improves video streaming quality and reduces buffering
- **Datacenter**: Reduces queue buildup and latency in high-speed networks

---

### Berkeley Sockets / BSD Sockets

**🌟 Simple**: A standard way programs ask the computer to send and receive messages over the network.

**📚 Student**: The common interface (API) that programs use to do networking. Like a universal remote that works with all TVs.

**🎓 Technical**: Standard API for network programming, providing functions like `socket()`, `bind()`, `listen()`, `connect()`, `send()`, `recv()`. Originated in BSD Unix, now standard across UNIX/Linux/Windows (Winsock).

**🔬 Advanced**: Socket layer abstracts protocol families (AF_INET, AF_INET6, AF_UNIX) and types (SOCK_STREAM, SOCK_DGRAM). Implemented via file descriptors in Unix. Kernel maps operations through proto_ops and proto structures to protocol-specific implementations.

**🎬 Scenario**:
- **Web server**: Creates socket, binds to port 80, listens for connections
- **SSH client**: Creates socket, connects to server port 22
- **Python script**: `socket.socket()` uses BSD socket API under the hood
- **Cross-platform**: Same code works on Linux, macOS, Windows (with minor differences)

---

## C

### Checksum

**🌟 Simple**: A special number calculated from your message to check if it got damaged during delivery.

**📚 Student**: A mathematical value calculated from data. The receiver recalculates it to verify data wasn't corrupted in transit. Like a seal on an envelope.

**🎓 Technical**: Error-detection code computed over packet data. IP uses 16-bit one's complement checksum of header. TCP/UDP checksum includes pseudo-header with addresses. Receiver validates and drops packets with bad checksums.

**🔬 Advanced**: IP checksum covers header only (efficiency). TCP/UDP includes data (reliability). Modern NICs often do checksum offload (hardware calculation). IPv6 removed IP-level checksum, relying on link-layer and transport-layer checks. Weak against intentional corruption but catches transmission errors.

**🎬 Scenario**:
- **Corrupted packet**: Bit flipped during transmission, checksum mismatch, packet dropped
- **Hardware offload**: NIC calculates checksum, reducing CPU load
- **Debugging**: `ethtool -k eth0` shows checksum offload status
- **Performance tuning**: Disable checksum for trusted internal networks (risk vs. speed)

---

### Congestion Control

**🌟 Simple**: Like slowing down when traffic is heavy, so everyone can move smoothly.

**📚 Student**: Methods to prevent sending data too fast and overwhelming the network. Adjust sending speed based on network conditions.

**🎓 Technical**: TCP mechanisms to detect and respond to network congestion. Includes slow start, congestion avoidance, fast retransmit, fast recovery. Uses congestion window (cwnd) to limit sending rate.

**🔬 Advanced**: Evolution: Tahoe → Reno → NewReno → Cubic (Linux default) → BBR. Loss-based algorithms (Reno, Cubic) interpret loss as congestion. Delay-based (Vegas) use RTT. Model-based (BBR) estimate bandwidth. ECN (Explicit Congestion Notification) allows routers to signal congestion without dropping packets.

**🎬 Scenario**:
- **File download**: Start slow, increase speed until loss detected, back off, repeat
- **Datacenter**: DCTCP uses ECN for early congestion signals, reduces latency
- **Satellite link**: High latency causes Reno to be inefficient, BBR performs better
- **Network research**: Tuning congestion control parameters for specific workloads

---

### Connection Tracking (conntrack)

**🌟 Simple**: Remembering all the conversations happening so you know which reply belongs to which question.

**📚 Student**: A system that keeps track of all active network connections, remembering which packets belong to which conversation.

**🎓 Technical**: Netfilter component that tracks TCP/UDP/ICMP sessions, maintaining state tables. Used by NAT and stateful firewalls to associate reply packets with original connections.

**🔬 Advanced**: Maintains connection tuples (proto, src_ip, src_port, dst_ip, dst_port) and state (NEW, ESTABLISHED, RELATED, INVALID). Hash table with configurable size. Handles NAT translation, FTP/SIP helpers for complex protocols. Can exhaust under high connection rates (DoS). Tunable via `/proc/sys/net/netfilter/`.

**🎬 Scenario**:
- **NAT router**: Tracks outbound connections to rewrite reply packets correctly
- **Firewall**: Allow ESTABLISHED,RELATED packets, block NEW from outside
- **DDoS mitigation**: Connection tracking table fills up, legitimate connections fail
- **Container networking**: Each namespace can have separate conntrack table

---

## D

### DMA (Direct Memory Access)

**🌟 Simple**: A helper that can move things directly without asking the main worker (CPU) every time.

**📚 Student**: Hardware that can transfer data directly to/from memory without involving the CPU. Makes data transfer much faster.

**🎓 Technical**: Mechanism allowing network cards to read/write memory directly without CPU intervention. NIC uses DMA to store received packets in RAM and fetch packets to transmit. Reduces CPU overhead and improves throughput.

**🔬 Advanced**: Uses ring buffers with descriptors pointing to memory regions. DMA mappings (dma_map_single, dma_map_page) handle IOMMU and cache coherency. Scatter-gather DMA allows non-contiguous buffers. Modern: IOMMU provides device isolation and virtualization support.

**🎬 Scenario**:
- **High-speed NIC**: 10Gbps card DMAs packets directly, CPU just processes headers
- **Virtualization**: IOMMU (VT-d, AMD-Vi) allows safe DMA to guest VMs
- **Zero-copy**: DMA from file cache to NIC without copying to intermediate buffers
- **Driver bug**: Incorrect DMA mapping can corrupt memory or cause crashes

---

### DSCP (Differentiated Services Code Point)

**🌟 Simple**: A priority label on packets saying "I'm important!" or "I can wait."

**📚 Student**: A 6-bit field in IP headers that marks packets with priority levels. Routers can use this to prioritize important traffic.

**🎓 Technical**: Replaces old ToS (Type of Service) field. Defines up to 64 traffic classes. Common values: EF (Expedited Forwarding) for low-latency, AF (Assured Forwarding) classes with different drop priorities, BE (Best Effort) default.

**🔬 Advanced**: Part of DiffServ QoS architecture. Per-hop behavior (PHB) determined by DSCP. Configured via tc (traffic control) in Linux. Remarking can occur at trust boundaries. Often combined with queue disciplines (pfifo_fast uses TOS/DSCP for priority queues).

**🎬 Scenario**:
- **VoIP call**: Packets marked EF for low latency, prioritized over email
- **Video conference**: AF41 for video, AF31 for audio, ensuring quality
- **Enterprise network**: Different DSCP for different departments or applications
- **Troubleshooting**: Packet captures show DSCP values, verify QoS markings

---

## E

### eBPF (Extended Berkeley Packet Filter)

**🌟 Simple**: A way to teach the kernel new tricks without changing the whole system.

**📚 Student**: A technology that lets you run small, safe programs inside the kernel to customize how it processes packets and other events.

**🎓 Technical**: In-kernel virtual machine allowing custom programs to run in kernel space safely. Used for packet filtering, tracing, performance monitoring, security. Verifier ensures safety (no infinite loops, bounds checking). JIT compilation for performance.

**🔬 Advanced**: eBPF programs attach to hooks (XDP, tc, tracepoints, kprobes, socket filters). Maps for state sharing between programs and user space. Helper functions for kernel interaction. CO-RE (Compile Once, Run Everywhere) for portability. Used by Cilium, Falco, bpftrace.

**🎬 Scenario**:
- **DDoS mitigation**: XDP eBPF program drops malicious packets at wire speed
- **Observability**: bpftrace script tracks TCP retransmissions without overhead
- **Load balancing**: Katran (Facebook) uses eBPF for L4 load balancing
- **Security**: Cilium provides network policy enforcement using eBPF

---

### ECN (Explicit Congestion Notification)

**🌟 Simple**: Instead of dropping packages when crowded, the post office puts a "busy" sticker on them.

**📚 Student**: A way for routers to tell senders "I'm getting congested" without dropping packets. More efficient than packet loss.

**🎓 Technical**: IP and TCP header bits allow routers to mark packets instead of dropping them when queues fill. Sender receives marked packets and reduces transmission rate. Requires support from both endpoints and intermediate routers.

**🔬 Advanced**: Uses 2 bits in IP header (ECT, CE) and 2 in TCP (ECE, CWR). AQM algorithms like RED can mark packets. L4S (Low Latency, Low Loss, Scalable throughput) uses ECN for microsecond-level latency control. Adoption challenges due to middlebox incompatibility.

**🎬 Scenario**:
- **Datacenter**: DCTCP uses ECN extensively for low latency
- **Video streaming**: ECN prevents bufferbloat, reducing playback delays
- **Path MTU**: ECN-capable routers can signal congestion earlier than drop
- **Debugging**: tcpdump shows ECN bits, verifying proper marking

---

### Ethernet

**🌟 Simple**: The most common way computers talk to each other using cables (or WiFi uses similar ideas).

**📚 Student**: A networking technology that defines how devices on a local network communicate. Uses MAC addresses to identify devices.

**🎓 Technical**: Link-layer protocol (IEEE 802.3) for local area networks. Frames contain source/dest MAC addresses, EtherType (protocol), data, and CRC. CSMA/CD for wired (obsolete with switches), CSMA/CA for wireless. Supports 10Mbps to 400Gbps speeds.

**🔬 Advanced**: Frame structure: Preamble(7) + SFD(1) + Dest MAC(6) + Src MAC(6) + 802.1Q VLAN(4, optional) + EtherType(2) + Payload(46-1500) + FCS(4). Jumbo frames up to 9000 bytes. Modern: Energy Efficient Ethernet (EEE), Time-Sensitive Networking (TSN).

**🎬 Scenario**:
- **Home network**: Switch forwards Ethernet frames based on MAC address table
- **Packet capture**: Wireshark shows Ethernet layer, source/dest MACs
- **VLAN tagging**: 802.1Q adds VLAN ID for network segmentation
- **Troubleshooting**: MAC address conflicts cause intermittent connectivity

---

## F

### Flow Control

**🌟 Simple**: Telling the sender "slow down, I can't handle more!" or "send more, I'm ready!"

**📚 Student**: Mechanisms to prevent a fast sender from overwhelming a slow receiver with too much data at once.

**🎓 Technical**: TCP uses receive window (rwnd) to advertise available buffer space. Sender limits transmission to min(cwnd, rwnd). Ethernet has PAUSE frames (802.3x) for link-level flow control. Prevents receiver buffer overflow.

**🔬 Advanced**: TCP window scaling (RFC 1323) allows windows > 64KB. Zero window probes when receiver advertises rwnd=0. Priority Flow Control (PFC, 802.1Qbb) for specific traffic classes. Interaction with congestion control: flow control is end-to-end receiver limit, congestion control is network capacity limit.

**🎬 Scenario**:
- **Slow application**: App doesn't read data fast enough, TCP advertises small window
- **Zero window**: Receiver full, stops sender, probes periodically to resume
- **RDMA networks**: PFC prevents packet loss in lossless Ethernet
- **Performance issue**: Window too small limits throughput on high-latency links

---

### Fragmentation

**🌟 Simple**: Breaking a big package into smaller boxes so it can fit through doors.

**📚 Student**: Splitting large packets into smaller pieces to fit through networks with smaller maximum packet sizes (MTU).

**🎓 Technical**: IP layer process when packet exceeds link MTU. IPv4 allows router fragmentation (DF bit can prevent). IPv6 requires source fragmentation only. Fragments reassembled at destination. Inefficient: increases overhead, any lost fragment loses entire packet.

**🔬 Advanced**: Fragment offset field (in 8-byte units) for reassembly. Identification field groups fragments. MF (More Fragments) flag. PMTUD (Path MTU Discovery) uses ICMP "fragmentation needed" with DF=1 to find path MTU. Fragmentation attacks: overlapping fragments, resource exhaustion.

**🎬 Scenario**:
- **VPN tunnel**: Inner packet + tunnel headers exceed MTU, fragmentation occurs
- **Large ping**: `ping -s 5000` creates fragmented packets
- **Performance**: Fragmentation causes throughput degradation, enable PMTUD
- **Firewall**: May block fragments for security, breaking legitimate traffic

---

## G

### GSO/TSO (Generic/TCP Segmentation Offload)

**🌟 Simple**: Sending one big package to the mail room, letting them split it into smaller pieces.

**📚 Student**: A technique where the kernel sends large packets and the network card (or software) splits them into smaller pieces. More efficient than splitting in the application.

**🎓 Technical**: Allows TCP to send super-packets (up to 64KB) through the stack. Segmentation happens late (GSO in software, TSO in hardware). Reduces per-packet overhead: fewer context switches, fewer trips through network stack.

**🔬 Advanced**: skb_shinfo(skb)->gso_size stores MSS. gso_type indicates protocol (TCP, UDP, etc.). Hardware TSO: NIC segments. Software GSO: dev_queue_xmit calls skb_gso_segment. Virtual: virtio_net supports GSO for VM performance. Interaction with checksum offload.

**🎬 Scenario**:
- **File transfer**: App sends 64KB, kernel processes once, NIC segments to 1500-byte packets
- **Virtualization**: Guest sends large packets, host/hypervisor segments
- **Performance**: 10x improvement in throughput for bulk transfers
- **Debugging**: `ethtool -k eth0 | grep gso` shows GSO/TSO status

---

### GRO (Generic Receive Offload)

**🌟 Simple**: Collecting multiple small packages and combining them into one big box to carry inside.

**📚 Student**: The opposite of GSO - combines multiple received packets into one large packet before processing. Makes receiving more efficient.

**🎓 Technical**: Aggregates packets in driver/NAPI layer before passing to stack. Combines packets from same flow (same TCP connection). Reduces number of trips through network stack. Software implementation of hardware LRO (Large Receive Offload).

**🔬 Advanced**: napi_gro_receive() checks if packet can merge with existing GRO packet. Time limit (napi_gro_flush_timer) ensures latency bounds. Careful protocol handling (TCP seq numbers, IP IDs). GRO + forwarding requires special handling. Enabled by default in modern kernels.

**🎬 Scenario**:
- **High throughput**: Server receiving 10Gbps sees 40% CPU reduction with GRO
- **Latency-sensitive**: Disable GRO for ultra-low latency trading applications
- **Virtual machine**: vhost-net uses GRO for VM receive performance
- **Debugging**: `ethtool -K eth0 gro on/off` to toggle

---

## H

### Handshake (TCP Three-Way)

**🌟 Simple**: Before talking, two people say "Hi!" "Hi back!" "Great, let's talk!"

**📚 Student**: The process TCP uses to establish a connection. Three steps: SYN (hello), SYN-ACK (hello back), ACK (let's start).

**🎓 Technical**: Connection establishment: Client sends SYN with initial sequence number. Server responds SYN-ACK with own sequence number. Client sends ACK. State transitions: CLOSED → SYN_SENT → ESTABLISHED (client), CLOSED → LISTEN → SYN_RECEIVED → ESTABLISHED (server).

**🔬 Advanced**: ISN (Initial Sequence Number) randomized for security. TCP options negotiated (MSS, window scaling, SACK, timestamps). SYN cookies defend against SYN floods. Fast Open (TFO) allows data in SYN. Simultaneous open possible but rare.

**🎬 Scenario**:
- **Web browser**: Every new HTTP connection starts with three-way handshake
- **SYN flood attack**: Attacker sends many SYNs, exhausts server resources
- **Firewall**: Only allows NEW connections from specific directions
- **Latency**: Handshake adds 1 RTT before data transfer (TFO mitigates)

---

### Headroom

**🌟 Simple**: Empty space at the front of a box where you can add labels without repacking.

**📚 Student**: Reserved space at the beginning of a packet buffer for adding headers without reallocating memory.

**🎓 Technical**: In sk_buff, the space between `head` and `data` pointers. Each layer uses skb_push() to add headers into headroom. Preallocating headroom avoids expensive buffer reallocation and data copying.

**🔬 Advanced**: LL_MAX_HEADER (96 bytes typically) reserves space for Ethernet + VLAN + encapsulation. skb_reserve() sets initial headroom. Insufficent headroom triggers skb_realloc_headroom() which allocates new buffer and copies data. Performance critical: headroom sizing affects zero-copy ability.

**🎬 Scenario**:
- **Packet transmission**: TCP layer pushes TCP header, IP layer pushes IP header, etc.
- **Tunneling**: VPN/VXLAN needs extra headroom for outer headers
- **Performance**: Insufficient headroom causes costly reallocations in hot path
- **Driver design**: NIC drivers must allocate appropriate headroom on RX

---

## I

### ICMP (Internet Control Message Protocol)

**🌟 Simple**: Special messages that say things like "can't deliver" or "are you there?"

**📚 Student**: A protocol for diagnostic and error messages. Used by ping, traceroute, and error reporting (destination unreachable, time exceeded).

**🎓 Technical**: Network-layer protocol (IP protocol 1). Types include Echo Request/Reply (ping), Destination Unreachable, Time Exceeded, Redirect. Not for user data, but network control and diagnostics. Often filtered by firewalls.

**🔬 Advanced**: ICMPv4 vs ICMPv6 (required for IPv6 NDP). Rate limiting prevents ICMP floods. PMTUD uses "Fragmentation Needed" with MTU value. Security: ICMP tunneling, smurf attacks. Kernel handling in ip_rcv() → icmp_rcv().

**🎬 Scenario**:
- **Ping**: Sends Echo Request, receives Echo Reply, measures RTT
- **Traceroute**: Uses TTL expiry to elicit Time Exceeded from each hop
- **PMTUD**: Large packet triggers "Fragmentation Needed", sender adjusts
- **Firewall**: Blocking all ICMP breaks PMTUD and diagnostics

---

### Interrupt

**🌟 Simple**: A doorbell that hardware rings to tell the CPU "Hey! Something happened!"

**📚 Student**: A signal from hardware (like network card) to CPU saying "new packet arrived!" CPU stops what it's doing to handle it.

**🎓 Technical**: Hardware interrupt (IRQ) triggers CPU to execute interrupt handler. For network cards: packet arrival → IRQ → kernel interrupt handler → process packet. Context switch overhead. Bottom half (softirq) for deferred processing.

**🔬 Advanced**: Hard IRQ (minimal work: disable interrupts, schedule softirq) vs softirq (packet processing). Interrupt affinity (IRQ → CPU mapping) via /proc/irq/. Interrupt coalescing reduces interrupt rate. NAPI reduces interrupts under load (polling mode). MSI/MSI-X for multiple queues.

**🎬 Scenario**:
- **Low traffic**: Each packet causes interrupt, immediate processing
- **High traffic**: Interrupt storm (100k+ per second), CPU overwhelmed
- **NAPI**: First packet interrupts, then poll mode until idle
- **Tuning**: Set interrupt affinity to distribute across CPUs

---

### IP (Internet Protocol)

**🌟 Simple**: The addressing system that lets messages find their way across the internet.

**📚 Student**: The protocol that routes packets across networks using IP addresses (like 192.168.1.1). Provides best-effort delivery (no guarantees).

**🎓 Technical**: Network-layer protocol providing addressing and routing. IPv4: 32-bit addresses, ~4 billion. IPv6: 128-bit addresses. Connectionless, unreliable. Fragmentation, TTL, header checksum (IPv4 only), protocol field (TCP=6, UDP=17).

**🔬 Advanced**: IPv4 header: version(4), IHL(4), ToS(8), length(16), ID(16), flags(3), fragment offset(13), TTL(8), protocol(8), checksum(16), src(32), dst(32), options. Routing: longest prefix match. ECMP for load balancing. Source routing (deprecated). IP options rarely used.

**🎬 Scenario**:
- **Routing**: Router looks up dest IP, forwards to next hop
- **NAT**: Router rewrites source IP for outbound packets
- **Firewall**: Filters based on source/dest IP addresses
- **IPv6 migration**: Dual stack or tunneling mechanisms

---

### iptables / netfilter

**🌟 Simple**: A security guard that decides which packages to let through, which to block, and which to redirect.

**📚 Student**: Linux firewall system. Rules decide what to do with each packet: accept, drop, reject, or modify.

**🎓 Technical**: Kernel framework (netfilter) with userspace tool (iptables). Tables (filter, nat, mangle, raw) with chains (PREROUTING, INPUT, FORWARD, OUTPUT, POSTROUTING). Rules match packets and apply targets (-j ACCEPT/DROP/REJECT/MASQUERADE/etc.).

**🔬 Advanced**: Hooks in network stack (NF_INET_*). Connection tracking integration. Stateful firewalling (state module). NAT types: SNAT, DNAT, MASQUERADE. Performance: linear rule traversal (nftables improves with better data structures). eBPF/XDP for higher performance alternatives.

**🎬 Scenario**:
- **Basic firewall**: Drop all incoming except established connections and port 22
- **NAT router**: MASQUERADE for sharing one public IP
- **Port forwarding**: DNAT redirect port 80 to internal server
- **Debugging**: `iptables -L -v -n` shows rules and packet counts

---

## J

### Jitter

**🌟 Simple**: Sometimes packages arrive at different times even though they were sent regularly. That's jitter.

**📚 Student**: Variation in packet arrival times. If packets should arrive every 10ms but come at 8ms, 12ms, 9ms, 11ms - that variation is jitter.

**🎓 Technical**: Variance in packet delay. Important for real-time applications (VoIP, gaming). Caused by queuing delays, routing changes, network congestion. Measured as standard deviation of latency. Jitter buffers smooth out variations.

**🔬 Advanced**: Packet Delay Variation (PDV) per ITU-T. Affects QoS metrics. Caused by statistical multiplexing, different queue priorities, packet reordering. Mitigation: traffic shaping, priority queuing, reducing bufferbloat. Measurement: one-way vs two-way jitter.

**🎬 Scenario**:
- **VoIP**: High jitter causes choppy audio, jitter buffer compensates
- **Gaming**: Jitter causes rubber-banding effect, affects playability
- **Video conferencing**: Jitter buffer trades latency for smoothness
- **Monitoring**: `ping -D` shows timestamps to calculate jitter

---

## K

### Keep-Alive

**🌟 Simple**: Sending a "are you still there?" message to make sure the connection is alive.

**📚 Student**: Periodic messages sent to check if the connection is still active, especially when no data is being transferred.

**🎓 Technical**: TCP keep-alive: periodic probes when idle to detect dead connections. Configured via socket options: SO_KEEPALIVE (enable), TCP_KEEPIDLE (time before first probe), TCP_KEEPINTVL (interval between probes), TCP_KEEPCNT (probe count before timeout).

**🔬 Advanced**: Application-level vs transport-level keep-alive. TCP keep-alive RFC 1122: off by default, configurable timeouts (default 2 hours idle). HTTP keep-alive (persistent connections) different concept. NAT/firewall timeout issues require keep-alive. Consider application-level heartbeats for faster detection.

**🎬 Scenario**:
- **SSH session**: Keep-alive prevents connection timeout through NAT/firewall
- **Database connection**: Detect if DB server crashed, reconnect
- **NAT traversal**: Keep-alive prevents NAT mapping from expiring
- **Resource management**: Detect dead clients, free server resources

---

## L

### Latency

**🌟 Simple**: How long it takes for a message to travel from sender to receiver.

**📚 Student**: The time delay between sending data and receiving it. Like the time it takes for a letter to be delivered. Lower is better.

**🎓 Technical**: Round-Trip Time (RTT) measures sender → receiver → sender. One-way latency is half (approximately). Components: serialization delay, propagation delay, queuing delay, processing delay. Critical for interactive applications.

**🔬 Advanced**: Latency breakdown: application (syscall overhead), kernel (stack traversal), qdisc (queuing), driver (ring buffer), NIC (serialization), wire (speed of light), plus same on receive. Bufferbloat increases latency. Modern techniques: XDP, kernel bypass reduce latency to <10μs.

**🎬 Scenario**:
- **Gaming**: 20ms latency feels responsive, 200ms unplayable
- **Trading**: Microseconds matter, use kernel bypass and hardware acceleration
- **Video call**: <150ms latency acceptable, >400ms causes talk-over
- **Debugging**: `ping` measures RTT, `mtr` shows per-hop latency

---

### Listen

**🌟 Simple**: Waiting at the door to see if anyone wants to talk to you.

**📚 Student**: A server socket state where it's waiting for clients to connect. Like a receptionist waiting for calls.

**🎓 Technical**: TCP state after calling `listen()` system call. Socket bound to address/port, ready to accept incoming connections. Backlog parameter specifies queue length for pending connections.

**🔬 Advanced**: LISTEN state maintains SYN queue (half-open connections) and accept queue (completed handshakes). SYN cookies when SYN queue full. `accept()` dequeues from accept queue. Per-port or per-address listen. SO_REUSEPORT for multiple processes listening on same port.

**🎬 Scenario**:
- **Web server**: Listens on port 80/443 for HTTP/HTTPS connections
- **SSH daemon**: Listens on port 22 for incoming SSH sessions
- **Load balancing**: SO_REUSEPORT allows multiple workers to accept
- **Port scan**: nmap detects LISTEN sockets by SYN probes

---

### Loopback

**🌟 Simple**: Talking to yourself - sending messages to your own computer.

**📚 Student**: A special network interface (usually 127.0.0.1) that lets programs on the same computer talk to each other.

**🎓 Technical**: Virtual network interface (`lo`) for local communication. IPv4: 127.0.0.0/8, typically 127.0.0.1. IPv6: ::1. No hardware involved. Fast path in kernel - doesn't go through full network stack.

**🔬 Advanced**: Loopback optimization: netif_rx_internal() skips driver layer. MTU typically 64KB (no physical medium). Used for IPC via sockets, testing, services binding to localhost. Not routed externally (RFC 1122).

**🎬 Scenario**:
- **Development**: Database on localhost:5432, web server on localhost:8080
- **Security**: Bind service to 127.0.0.1 to prevent external access
- **Container**: Each namespace has own loopback interface
- **Testing**: Packet capture on `lo` for debugging local communication

---

## M

### MAC Address (Media Access Control)

**🌟 Simple**: The serial number of your network card - unique to your device.

**📚 Student**: A hardware address (like 00:1A:2B:3C:4D:5E) that identifies a network interface. Used on local networks before IP addresses.

**🎓 Technical**: 48-bit identifier (6 bytes in hex). First 3 bytes: OUI (manufacturer). Last 3 bytes: device unique. Ethernet, WiFi use MAC addresses for frame delivery on LAN. ARP maps IP to MAC.

**🔬 Advanced**: Unicast vs multicast (LSB of first byte). Locally administered (bit 1 of first byte) vs globally unique. MAC spoofing possible (security concern). Switch learns MAC addresses (MAC table). VLAN tagging uses MAC layer.

**🎬 Scenario**:
- **Switch**: Forwards frame to port associated with destination MAC
- **DHCP**: Uses MAC address to assign consistent IP address
- **MAC filtering**: WiFi access control by MAC whitelist (weak security)
- **Debugging**: `arp -a` or `ip neigh` shows IP-to-MAC mappings

---

### MSS (Maximum Segment Size)

**🌟 Simple**: The biggest chunk of data that can fit in one package.

**📚 Student**: The largest amount of data TCP can send in one segment. Smaller than MTU because it doesn't include headers.

**🎓 Technical**: TCP option negotiated during handshake. Typically MTU - IP header (20) - TCP header (20) = 1460 bytes for standard 1500-byte Ethernet. Prevents fragmentation.

**🔬 Advanced**: MSS vs MTU: MSS is TCP payload, MTU is entire IP packet. MSS clamping by routers/firewalls for PMTUD issues. TCP options reduce MSS (timestamps, SACK take 12-40 bytes). Jumbo frames allow larger MSS. TSO/GSO send super-segments larger than MSS.

**🎬 Scenario**:
- **Standard Ethernet**: MTU 1500, MSS 1460 (IPv4)
- **PPPoE**: MTU 1492, MSS 1452 (PPPoE adds 8 bytes)
- **VPN**: Inner MSS reduced for tunnel overhead
- **Performance**: Larger MSS improves efficiency (fewer packets for same data)

---

### MTU (Maximum Transmission Unit)

**🌟 Simple**: The biggest package a delivery truck can carry.

**📚 Student**: The largest packet size that can be sent on a network. Ethernet is usually 1500 bytes.

**🎓 Technical**: Link-layer maximum packet size. Ethernet: 1500 bytes (standard), 9000 bytes (jumbo frames). If packet exceeds MTU, either fragmented (IPv4) or ICMP error returned (IPv6, or IPv4 with DF=1).

**🔬 Advanced**: Path MTU: minimum MTU along route. PMTUD discovers path MTU via ICMP "fragmentation needed". IPv6 requires minimum 1280 bytes. Tunneling reduces effective MTU. `ip link set dev eth0 mtu 9000` for jumbo frames.

**🎬 Scenario**:
- **VPN tunnel**: Outer headers reduce available MTU for inner packet
- **Jumbo frames**: Datacenters use MTU 9000 for better throughput
- **Fragmentation**: Large ping exceeds MTU, gets fragmented
- **Black hole**: Firewall blocks ICMP, PMTUD fails, connections hang

---

## N

### NAPI (New API)

**🌟 Simple**: Instead of ringing the doorbell for every letter, the mail carrier drops off a batch and you check periodically.

**📚 Student**: A technique where the network card switches from interrupting for each packet to letting the CPU poll for packets when traffic is high.

**🎓 Technical**: Hybrid interrupt/polling model for packet reception. Low traffic: interrupt-driven. High traffic: disable interrupts, poll for packets (budget-limited). Reduces interrupt overhead and improves throughput.

**🔬 Advanced**: Driver implements `napi_struct` with poll function. First packet interrupts, `napi_schedule()` adds to poll list. `net_rx_action()` polls devices with budget (default 64 packets). When queue empty, `napi_complete()` re-enables interrupts. Per-device and per-CPU NAPI contexts.

**🎬 Scenario**:
- **Low traffic**: Each packet interrupts, ~1μs latency
- **High traffic**: Polling processes 64 packets per poll, reduces interrupt load
- **DDoS**: NAPI prevents interrupt storm from overwhelming CPU
- **Tuning**: `/proc/sys/net/core/netdev_budget` adjusts processing budget

---

### NAT (Network Address Translation)

**🌟 Simple**: A translator that changes the "from" address on letters so many people can share one address.

**📚 Student**: Allows multiple devices on a private network to share one public IP address. Your router does this at home.

**🎓 Technical**: Rewrites IP addresses (and usually ports) in packet headers. SNAT: source NAT (outbound). DNAT: destination NAT (inbound/port forwarding). Connection tracking maintains translation state.

**🔬 Advanced**: Types: Full-cone, address-restricted, port-restricted, symmetric. Implemented via netfilter POSTROUTING (SNAT/MASQUERADE) and PREROUTING (DNAT). Port exhaustion (65k ports per IP). NAT traversal techniques: STUN, TURN, ICE, UPnP. Carrier-grade NAT (CGN/NAT44) issues.

**🎬 Scenario**:
- **Home router**: Private IPs (192.168.x.x) NATed to single public IP
- **Port forwarding**: External:80 → Internal:192.168.1.5:8080
- **VoIP/gaming**: NAT traversal required for P2P connections
- **Debugging**: `conntrack -L` shows NAT mappings

---

### Netlink

**🌟 Simple**: A special way for programs to talk to the kernel about network stuff.

**📚 Student**: A communication channel between user programs and the kernel for network configuration and information.

**🎓 Technical**: Socket-based interface for kernel-user communication. Used by `ip`, `tc`, `ss` commands. Families: NETLINK_ROUTE (routing), NETLINK_FIREWALL, NETLINK_NETFILTER. Asynchronous notifications.

**🔬 Advanced**: Messages contain `nlmsghdr` + payload. Multipart messages. Multicast groups for event notifications. Used by iproute2 (vs obsolete ifconfig/route). Generic Netlink for extensibility. Provides consistent API for network subsystems.

**🎬 Scenario**:
- **`ip addr show`**: Queries kernel via NETLINK_ROUTE
- **Route changes**: Kernel sends netlink notifications to subscribed processes
- **Container networking**: Namespaces use netlink for interface management
- **Monitoring**: Programs subscribe to netlink groups for network events

---

## O

### Out-of-Order Packets

**🌟 Simple**: Letters arriving in the wrong order - #3 arrives before #2.

**📚 Student**: Packets arriving at the destination in a different order than they were sent. TCP reorders them correctly.

**🎓 Technical**: Common in IP networks due to different paths, load balancing. TCP uses sequence numbers to detect and reorder. Receiver queues out-of-order data until missing segments arrive. SACK (Selective Acknowledgment) helps sender know what to retransmit.

**🔬 Advanced**: TCP maintains receive queue and out-of-order queue. `tcp_ofo_queue` holds future segments. Triggers fast retransmit (duplicate ACKs). SACK blocks indicate received ranges. Excessive reordering degrades performance (spurious retransmissions).

**🎬 Scenario**:
- **ECMP routing**: Packets take different paths, arrive out-of-order
- **WiFi**: Interference causes retransmissions at link layer, reordering
- **TCP recovery**: SACK allows efficient recovery of multiple losses
- **Performance**: Reordering can trigger congestion control unnecessarily

---

## P

### Packet

**🌟 Simple**: A piece of your message with an address label, sent across the network.

**📚 Student**: A unit of data transmitted over a network. Contains headers (addresses, instructions) and payload (actual data).

**🎓 Technical**: Network communication unit. Structure varies by layer: Ethernet frame (layer 2), IP packet (layer 3), TCP segment/UDP datagram (layer 4). Typical max size ~1500 bytes for Ethernet.

**🔬 Advanced**: Encapsulation: each layer adds headers. Packet vs frame vs segment vs datagram terminology. Packet lifetime (TTL). Packet storms. Zero-copy packet handling in kernel (sk_buff design).

**🎬 Scenario**:
- **Wireshark**: Capture and analyze packet headers and contents
- **Dropped packet**: Lost in transit, TCP retransmits, UDP doesn't
- **Packet size**: Small packets → overhead, large packets → fragmentation
- **Filtering**: Firewall examines packet headers to make decisions

---

### PMTUD (Path MTU Discovery)

**🌟 Simple**: Figuring out the biggest package size that can fit through all the roads to your destination.

**📚 Student**: A technique to find the largest packet size that can travel without being fragmented along the entire path.

**🎓 Technical**: Sends packets with DF (Don't Fragment) bit set. If too large, router returns ICMP "Fragmentation Needed" with MTU. Sender reduces packet size and retries. Avoids fragmentation overhead.

**🔬 Advanced**: RFC 1191 (IPv4), RFC 8201 (IPv6). Initial estimate based on interface MTU. Binary search or explicit values. Black hole problem: firewalls blocking ICMP break PMTUD. TCP MSS negotiation provides initial estimate. PLPMTUD (RFC 4821) uses probing without ICMP dependency.

**🎬 Scenario**:
- **VPN**: PMTUD discovers reduced MTU due to tunnel overhead
- **Black hole**: ICMP blocked, connections hang on large transfers
- **IPv6**: Minimum MTU 1280, routers don't fragment, PMTUD essential
- **Debugging**: `tracepath` shows MTU along path

---

### Port

**🌟 Simple**: Like apartment numbers - the building (computer) has one address, but many apartments (programs).

**📚 Student**: A number (0-65535) that identifies which program on a computer should receive the data. HTTP uses port 80, HTTPS uses 443.

**🎓 Technical**: Transport layer (TCP/UDP) 16-bit identifier. Well-known ports (0-1023) require root. Registered (1024-49151), dynamic/private (49152-65535). Socket identified by 5-tuple: protocol, src_ip, src_port, dst_ip, dst_port.

**🔬 Advanced**: Ephemeral port allocation: `/proc/sys/net/ipv4/ip_local_port_range`. Port exhaustion with many connections. SO_REUSEADDR/SO_REUSEPORT socket options. Port knocking for security. Service names in `/etc/services`.

**🎬 Scenario**:
- **Web server**: Listens on port 80 (HTTP) or 443 (HTTPS)
- **Client**: Uses random ephemeral port (e.g., 52341) for connection
- **Firewall**: Allows port 22 (SSH), blocks others
- **Port scan**: nmap checks which ports are open

---

### Protocol

**🌟 Simple**: Rules for how to write letters and talk to each other properly.

**📚 Student**: A set of rules defining how data is formatted and transmitted. Different protocols for different purposes (TCP for reliable, UDP for fast).

**🎓 Technical**: Specification for communication. Layers: Physical (Ethernet), Network (IP), Transport (TCP/UDP), Application (HTTP/DNS). Each has headers, state machines, error handling.

**🔬 Advanced**: Protocol stack: encapsulation of headers. OSI 7-layer vs TCP/IP 4-layer model. Protocol numbers: IANA registry (TCP=6, UDP=17, ICMP=1). Kernel implements protocols via `net_protocol` and `inet_protosw` structures.

**🎬 Scenario**:
- **Web browsing**: Uses TCP (reliable) for HTTP/HTTPS
- **Video streaming**: Uses UDP (low latency) for real-time data
- **VoIP**: RTP over UDP for audio, SIP over TCP/UDP for signaling
- **Custom**: SCTP, DCCP for specialized needs

---

## Q

### QoS (Quality of Service)

**🌟 Simple**: Making sure important messages get delivered faster than less important ones.

**📚 Student**: Techniques to prioritize certain types of traffic. Voice calls get priority over email downloads.

**🎓 Technical**: Mechanisms to provide different treatment to different traffic. Classification (identify traffic), marking (DSCP), queuing (priority queues), shaping (rate limiting), policing (drop excess).

**🔬 Advanced**: Linux tc (traffic control): classful qdiscs (HTB, HFSC), classless (pfifo_fast, fq_codel). DiffServ (DSCP marking, per-hop behavior), IntServ (RSVP, obsolete). AQM algorithms: RED, CoDel. Bufferbloat mitigation with fq_codel.

**🎬 Scenario**:
- **VoIP**: Marked EF, prioritized in queues, low latency ensured
- **Video streaming**: AF class, moderate priority
- **Bulk download**: Best effort, background class
- **Enterprise**: Different QoS for executives vs general users

---

### Queue Discipline (qdisc)

**🌟 Simple**: Different lines for different types of packages - express line for urgent ones.

**📚 Student**: The algorithm that decides in what order packets are sent when multiple packets are waiting.

**🎓 Technical**: Linux traffic control component. Default: pfifo_fast (3 priority bands). Others: TBF (token bucket), HTB (hierarchical), fq_codel (fair queue + delay control). Configured via `tc qdisc` commands.

**🔬 Advanced**: Classful (HTB, HFSC) allow hierarchical shaping. Classless (pfifo, bfifo, fq_codel) simple. Ingress qdisc for receive side. IFB (Intermediate Functional Block) for ingress shaping. Offload qdiscs (mq, mqprio) for multi-queue NICs.

**🎬 Scenario**:
- **Rate limiting**: TBF limits outbound to 1Mbps
- **Bufferbloat**: fq_codel reduces latency on congested links
- **Prioritization**: HTB gives voice traffic guaranteed bandwidth
- **Debugging**: `tc -s qdisc show dev eth0` shows statistics

---

## R

### Retransmission

**🌟 Simple**: Sending a letter again because it got lost the first time.

**📚 Student**: When TCP detects a packet was lost, it sends it again. Ensures reliable delivery.

**🎓 Technical**: TCP retransmits unacknowledged segments. Triggers: timeout (RTO - retransmission timeout) or fast retransmit (3 duplicate ACKs). Exponential backoff for successive retransmissions.

**🔬 Advanced**: RTO calculation: smoothed RTT + variance (Jacobson/Karels algorithm). Fast retransmit + fast recovery (NewReno). SACK-based recovery. Spurious retransmissions (unnecessary) detected via timestamps. TLP (Tail Loss Probe), RACK (Recent ACK) improve loss detection.

**🎬 Scenario**:
- **Packet loss**: WiFi interference drops packet, TCP retransmits after 200ms
- **Fast retransmit**: 3 duplicate ACKs trigger immediate retransmit
- **Timeout**: No ACK received, RTO fires, retransmit and double RTO
- **Performance**: Excessive retransmissions indicate network issues

---

### Routing

**🌟 Simple**: Figuring out which roads to take to get a letter to its destination.

**📚 Student**: The process of choosing a path for packets to travel from source to destination across multiple networks.

**🎓 Technical**: IP layer function. Routing table maps destination networks to next hop. Longest prefix match. Default route (0.0.0.0/0). Static routes (configured) vs dynamic (BGP, OSPF, RIP).

**🔬 Advanced**: FIB (Forwarding Information Base) vs RIB (Routing Information Base). Linux: `ip route` manages routes. Policy routing (`ip rule`). Multiple routing tables. ECMP (Equal-Cost Multi-Path) load balancing. Source routing. Kernel: `fib_lookup()` finds route.

**🎬 Scenario**:
- **Internet gateway**: Default route points to ISP router
- **Multi-homed**: Policy routing selects interface based on source IP
- **ECMP**: Traffic load-balanced across multiple equal-cost paths
- **Container**: Each namespace has independent routing table

---

### RTT (Round-Trip Time)

**🌟 Simple**: How long it takes to send a message and get a reply back.

**📚 Student**: The time for a packet to go from sender to receiver and back. Measured by ping. Lower is better.

**🎓 Technical**: Critical metric for TCP performance. Used to calculate RTO (retransmission timeout). TCP timestamps option measures RTT per segment. Affects throughput: `throughput = window_size / RTT`.

**🔬 Advanced**: Smoothed RTT (SRTT) and RTT variance (RTTVAR) tracked per connection. Karn's algorithm handles retransmission ambiguity. RTT affects congestion control (BBR uses RTT explicitly). Bufferbloat increases RTT under load.

**🎬 Scenario**:
- **Local network**: RTT <1ms, very responsive
- **Cross-country**: RTT ~50-100ms, noticeable delay
- **Satellite**: RTT ~600ms, significant latency
- **Diagnosis**: High RTT indicates congestion or long path

---

### RX (Receive)

**🌟 Simple**: Getting mail - the incoming direction.

**📚 Student**: Receiving data from the network. The path from network card to application.

**🎓 Technical**: Receive path: NIC → DMA → driver → NAPI poll → network stack → protocol handlers → socket queue → application recv(). Includes interrupt handling, packet processing, demultiplexing.

**🔬 Advanced**: Receive-side optimizations: RSS (hardware multi-queue), RPS (software steering), RFS (flow steering), GRO (aggregation), XDP (early processing). Per-queue and per-CPU processing. Backlog queues when locked.

**🎬 Scenario**:
- **High bandwidth**: Multi-queue RSS distributes across CPUs
- **Low latency**: XDP processes at driver level, bypasses stack
- **Monitoring**: `ethtool -S eth0 | grep rx` shows receive statistics
- **Drops**: `netstat -s | grep -i drop` identifies receive bottlenecks

---

## S

### SACK (Selective Acknowledgment)

**🌟 Simple**: Instead of saying "I got everything up to #5", you can say "I got #1-5, #7, #9-12 (missing 6 and 8)".

**📚 Student**: A TCP feature that lets the receiver tell the sender exactly which packets were received, not just "everything up to here". More efficient recovery from loss.

**🎓 Technical**: TCP option (RFC 2018) that includes ranges of received data in ACK. Allows sender to selectively retransmit only missing segments. Negotiated during handshake (SACK permitted option).

**🔬 Advanced**: SACK blocks (left edge, right edge pairs) in TCP options (max 3-4 blocks per ACK due to option space). Sender maintains scoreboard of SACKed data. Linux uses FACK (Forward Acknowledgment) algorithm with SACK. D-SACK (duplicate SACK) detects spurious retransmissions and reordering.

**🎬 Scenario**:
- **Multiple losses**: Packets 10, 15, 20 lost, SACK allows retransmit all three
- **No SACK**: Would require multiple RTTs to discover all losses
- **Reordering**: D-SACK indicates packet was reordered, not lost
- **Check**: `ss -ti` shows SACK status for connections

---

### Segmentation

**🌟 Simple**: Breaking a big message into smaller pieces to send.

**📚 Student**: Dividing data into chunks (segments) for transmission. TCP breaks your data into segments that fit in packets.

**🎓 Technical**: Transport layer function. TCP segments data based on MSS. Each segment gets sequence number. Receiver reassembles. TSO/GSO defer segmentation for efficiency.

**🔬 Advanced**: MSS determines segment size. TCP_NODELAY disables Nagle algorithm (may send small segments). Segmentation offload: kernel sends large segments, hardware/software splits late. Interaction with congestion window (cwnd in segments, not bytes).

**🎬 Scenario**:
- **File transfer**: 1MB file split into ~700 segments (MSS 1460)
- **TSO enabled**: Kernel sends 64KB super-segment, NIC segments
- **Small writes**: Without Nagle, each write becomes segment (overhead)
- **Jumbo frames**: Larger MTU allows larger segments, fewer overhead

---

### sk_buff (Socket Buffer)

**🌟 Simple**: A special container that holds a packet as it travels through the computer.

**📚 Student**: The kernel's data structure representing a network packet. Contains headers, data, and metadata. Used throughout the network stack.

**🎓 Technical**: Core networking structure. Contains pointers (head, data, tail, end) for buffer management, protocol headers, associated socket, device, timestamps, etc. Allows header manipulation without data copying.

**🔬 Advanced**: Zero-copy via pointer manipulation (skb_push/pull/put/trim). Clone (share data) vs copy. Paged data (skb_frag) for large buffers. Shared info (gso_size, nr_frags). Control buffer (cb[48]) for protocol state. Highly cache-optimized layout.

**🎬 Scenario**:
- **TX path**: skb allocated, data added, headers pushed at each layer
- **RX path**: Driver allocates skb, headers pulled as processed
- **Clone**: Multicast sends cloned skb to multiple sockets
- **Memory leak**: Forgetting kfree_skb() causes gradual memory exhaustion

---

### Socket

**🌟 Simple**: An endpoint for communication - like a telephone that programs use to talk over the network.

**📚 Student**: A programming interface for network communication. You create a socket, connect it, then send/receive data through it.

**🎓 Technical**: Communication endpoint, accessed via file descriptor. Created with `socket()`, configured with `bind()`, `connect()`, `listen()`. Types: STREAM (TCP), DGRAM (UDP), RAW (IP). Families: AF_INET (IPv4), AF_INET6 (IPv6), AF_UNIX (local).

**🔬 Advanced**: Kernel struct sock and socket. proto_ops dispatch to protocol-specific functions. Socket states (TCP_LISTEN, TCP_ESTABLISHED, etc.). Socket options (SO_REUSEADDR, SO_RCVBUF, TCP_NODELAY, etc.). Per-socket queues, locks, callbacks.

**🎬 Scenario**:
- **Web browser**: Creates socket, connects to server:80, sends HTTP request
- **Web server**: Creates socket, binds to port 80, listens, accepts connections
- **IPC**: AF_UNIX socket for fast local communication (Docker, systemd)
- **Raw socket**: Packet capture, custom protocols (requires root)

---

### SYN (Synchronize)

**🌟 Simple**: The first "hello!" when starting a conversation.

**📚 Student**: The first step in TCP connection establishment. Client sends SYN to server saying "let's connect".

**🎓 Technical**: TCP flag indicating connection initiation. SYN packet contains initial sequence number. Server responds with SYN-ACK. Part of three-way handshake: SYN → SYN-ACK → ACK.

**🔬 Advanced**: SYN packet includes TCP options: MSS, window scaling, SACK permitted, timestamps. ISN randomized for security. SYN queue holds half-open connections. SYN cookies defend against SYN floods. Simultaneous open: both sides send SYN.

**🎬 Scenario**:
- **Normal connection**: Client SYN → Server SYN-ACK → Client ACK → ESTABLISHED
- **SYN flood**: Attacker sends many SYNs, exhausts server resources
- **SYN cookies**: Server doesn't keep state until final ACK received
- **Firewall**: SYN packet starts NEW connection state in conntrack

---

## T

### TCP (Transmission Control Protocol)

**🌟 Simple**: A reliable way to send messages - makes sure everything arrives in order and nothing is lost.

**📚 Student**: Connection-oriented, reliable protocol. Guarantees delivery, order, and error checking. Used for web, email, file transfer. Slower than UDP but reliable.

**🎓 Technical**: Transport layer protocol (IP protocol 6). Connection establishment (3-way handshake), data transfer (seq/ack numbers), flow control (window), congestion control, connection termination (FIN). Full-duplex, byte-stream.

**🔬 Advanced**: State machine: CLOSED, LISTEN, SYN_SENT, SYN_RCVD, ESTABLISHED, FIN_WAIT_1/2, CLOSE_WAIT, CLOSING, LAST_ACK, TIME_WAIT. Features: Nagle algorithm, delayed ACK, sliding window, selective acknowledgment, timestamps, window scaling. Fast path for established connections.

**🎬 Scenario**:
- **HTTP**: Web browsing uses TCP for reliable page downloads
- **SSH**: Remote shell requires reliable, ordered delivery
- **Database**: Queries/results must arrive correctly and completely
- **Monitoring**: `ss -tan` shows TCP connections and states

---

### TFO (TCP Fast Open)

**🌟 Simple**: A shortcut that lets you start talking immediately instead of saying "hello" first.

**📚 Student**: A TCP enhancement that allows data to be sent with the initial SYN packet, saving one round-trip time.

**🎓 Technical**: Client includes TFO cookie in SYN + data. Server validates cookie and can accept data immediately without waiting for third ACK. Requires support from both endpoints. Reduces connection latency by ~1 RTT.

**🔬 Advanced**: Cookie generated by server, cached by client. SYN data processed if cookie valid, else standard 3-way handshake. Security considerations: SYN data can be replayed. Linux: `TCP_FASTOPEN` socket option, `tcp_fastopen` sysctl. Useful for short connections (HTTP requests).

**🎬 Scenario**:
- **HTTP**: GET request in SYN saves ~30ms on first request
- **Frequent connections**: Client reuses cached TFO cookie
- **CDN**: Akamai, Cloudflare use TFO to reduce latency
- **Enable**: `sysctl net.ipv4.tcp_fastopen=3` (client+server)

---

### TTL (Time To Live)

**🌟 Simple**: A countdown timer on packages so they don't wander forever if lost.

**📚 Student**: A number in each packet that decreases by 1 at each router. When it reaches 0, the packet is discarded. Prevents infinite loops.

**🎓 Technical**: IP header field (8 bits, 0-255). Decremented at each hop. Router drops packet when TTL=0 and sends ICMP Time Exceeded. Typical initial values: 64 (Linux), 128 (Windows), 255 (Cisco).

**🔬 Advanced**: IPv6 equivalent: Hop Limit. Traceroute uses TTL incrementally to discover path. OS fingerprinting via initial TTL. TTL in DNS for cache duration (different concept). MPLS uses TTL in label stack.

**🎬 Scenario**:
- **Traceroute**: TTL=1 gets first hop, TTL=2 gets second, etc.
- **Routing loop**: TTL prevents packets circulating forever
- **OS detection**: Initial TTL 64 suggests Linux, 128 suggests Windows
- **Long path**: Internet paths typically <30 hops, TTL 64 sufficient

---

### TX (Transmit)

**🌟 Simple**: Sending mail - the outgoing direction.

**📚 Student**: Sending data to the network. The path from application to network card.

**🎓 Technical**: Transmit path: Application send() → socket → protocol (TCP/UDP) → IP → qdisc → driver → NIC → DMA → wire. Includes packet construction, routing, queuing, hardware control.

**🔬 Advanced**: Transmit optimizations: TSO/GSO (segmentation offload), checksum offload, multi-queue, BQL (byte queue limits). Zero-copy: sendfile(), MSG_ZEROCOPY. Backpressure: flow control, congestion control, qdisc limits.

**🎬 Scenario**:
- **Bulk transfer**: GSO sends 64KB chunks, NIC segments
- **Real-time**: Small packets bypass TSO for low latency
- **Multi-queue**: Each CPU has dedicated TX queue
- **Monitoring**: `ethtool -S eth0 | grep tx` shows transmit statistics

---

## U

### UDP (User Datagram Protocol)

**🌟 Simple**: A fast way to send messages - fire and forget, no guarantees.

**📚 Student**: Connectionless, unreliable protocol. Fast, low overhead, but packets can be lost, duplicated, or reordered. Used for gaming, streaming, DNS.

**🎓 Technical**: Transport layer protocol (IP protocol 17). No connection setup, no acknowledgments, no retransmission. Simple header: src port, dst port, length, checksum. Applications handle reliability if needed.

**🔬 Advanced**: Stateless: no sequence numbers, no flow control. Checksum optional in IPv4 (required in IPv6). UDP-Lite allows partial checksums. Use cases: DNS (small, retry OK), QUIC (custom reliability), RTP (media streaming), TFTP, NTP.

**🎬 Scenario**:
- **DNS query**: Send UDP packet to port 53, expect reply (retry if timeout)
- **Video streaming**: Lost frame skipped, no retransmission (would be too late)
- **Gaming**: Player position updates, loss acceptable for speed
- **VoIP**: RTP over UDP for real-time audio

---

## V

### VLAN (Virtual LAN)

**🌟 Simple**: Dividing one network into separate sections that can't see each other, even on the same wire.

**📚 Student**: A technique to create multiple isolated networks on the same physical infrastructure. Devices in different VLANs can't communicate without a router.

**🎓 Technical**: IEEE 802.1Q standard adds 4-byte tag to Ethernet frames. Tag includes 12-bit VLAN ID (4096 VLANs). Switch forwards frames only to ports in same VLAN. Trunk ports carry multiple VLANs (tagged).

**🔬 Advanced**: Tag format: TPID (0x8100), PCP (3-bit priority), DEI (1-bit drop eligible), VID (12-bit VLAN ID). Native VLAN (untagged on trunk). QinQ (802.1ad) for nested VLANs. Linux: vconfig or ip link add type vlan.

**🎬 Scenario**:
- **Enterprise**: VLAN 10 for employees, VLAN 20 for guests, isolated
- **Datacenter**: Per-tenant VLANs for isolation
- **Home**: Management VLAN separate from user network
- **Configuration**: `ip link add link eth0 name eth0.10 type vlan id 10`

---

## W

### Window (TCP Window)

**🌟 Simple**: How much space you have in your mailbox - tells the sender how much they can send before waiting.

**📚 Student**: The amount of data the receiver can accept without acknowledgment. Flow control mechanism to prevent overwhelming the receiver.

**🎓 Technical**: Advertised in TCP header (16-bit field). Indicates available buffer space at receiver. Sender limits transmission to min(cwnd, rwnd). Window scaling option (RFC 1323) allows >64KB windows for high-bandwidth links.

**🔬 Advanced**: Receive window (rwnd): receiver's buffer. Congestion window (cwnd): sender's estimate of network capacity. Effective window = min(rwnd, cwnd). Zero window: receiver full, stops sender. Window probes when zero. Window update after application reads data.

**🎬 Scenario**:
- **Slow consumer**: App doesn't read, window shrinks, sender slows
- **Zero window**: Receiver advertises 0, sender sends probes
- **High BDP**: Long fat network needs large window, requires window scaling
- **Performance**: `ss -tim` shows current window sizes

---

### Wireshark

**🌟 Simple**: A tool that lets you see all the messages traveling on the network, like reading all the letters being delivered.

**📚 Student**: Network protocol analyzer. Captures packets and shows you all the details - headers, data, timing. Essential for debugging.

**🎓 Technical**: GUI packet capture tool using libpcap/WinPcap. Dissects protocols from Ethernet to application layer. Filters for specific traffic. Display filters, capture filters, statistics, flow graphs.

**🔬 Advanced**: Deep protocol dissection (1000+ protocols). Lua scripting for custom dissectors. Expert analysis for problems. TCP stream reassembly. Decryption (with keys). Live capture and offline analysis. Tshark for CLI.

**🎬 Scenario**:
- **Debug**: HTTP 404 error, capture shows server response details
- **Performance**: TCP window full, indicates receiver bottleneck
- **Security**: Detect ARP spoofing, identify malicious packets
- **Learning**: See TCP handshake, observe SACK in action

---

## X

### XDP (eXpress Data Path)

**🌟 Simple**: A super-fast path that lets you handle packages before they even enter the building.

**📚 Student**: A technology that processes packets at the earliest possible point (in the driver), allowing extremely fast packet processing.

**🎓 Technical**: eBPF programs run in driver context, before sk_buff allocation. Actions: XDP_DROP (discard), XDP_PASS (normal stack), XDP_TX (bounce back), XDP_REDIRECT (forward). Lowest latency path in Linux.

**🔬 Advanced**: Runs after DMA but before allocation overhead. Direct packet buffer access. XDP generic (software), native (driver), offloaded (NIC). AF_XDP for zero-copy user-space delivery. Used for DDoS mitigation, load balancing (Katran), packet routing.

**🎬 Scenario**:
- **DDoS mitigation**: Drop attack traffic at 10M+ pps without reaching stack
- **Load balancer**: Facebook Katran forwards packets in XDP at line rate
- **Packet filtering**: Firewall in XDP, bypass netfilter overhead
- **AF_XDP**: DPDK-like performance without kernel bypass

---

## Z

### Zero-Copy

**🌟 Simple**: Moving packages without unpacking and repacking them - just changing labels.

**📚 Student**: Techniques to transfer data without copying it multiple times. Improves performance by reducing CPU and memory overhead.

**🎓 Technical**: Avoiding data copies between kernel and user space, or between buffers. Techniques: sendfile() (file to socket), splice(), MSG_ZEROCOPY (send without copy), DMA directly from/to user buffers.

**🔬 Advanced**: sendfile(): page cache → socket without user space. splice(): pipe-based zero-copy. MSG_ZEROCOPY: pin user pages, DMA directly, notification on completion. sk_buff paged data. Hardware support (DMA, scatter-gather). Trade-off: complexity, memory pinning.

**🎬 Scenario**:
- **Web server**: sendfile() serves static files efficiently
- **Proxy**: splice() forwards data without copying
- **High-speed**: MSG_ZEROCOPY for 40Gbps+ without CPU bottleneck
- **Limitation**: Requires contiguous or scatter-gather capable hardware

---

## Cross-Reference: Terms by Scenario

### Web Browsing
- **TCP**: Reliable connection for HTTP/HTTPS
- **Port**: 80 for HTTP, 443 for HTTPS
- **Three-way handshake**: Establishes connection
- **ACK**: Confirms receipt of webpage data
- **MSS**: Determines packet size for page download
- **Keep-alive**: Reuses connection for multiple requests

### Gaming / Real-time
- **UDP**: Low latency, accepts some packet loss
- **Jitter**: Causes rubber-banding effect
- **Latency**: Critical for responsiveness
- **QoS**: Prioritizes game traffic
- **Port**: Game-specific ports (e.g., 27015 for Source games)

### Video Streaming
- **UDP**: RTP for media delivery
- **ECN**: Prevents bufferbloat
- **QoS**: Prioritizes video traffic
- **Jitter buffer**: Smooths playback
- **BBR**: Optimizes throughput and latency

### DDoS Attack/Mitigation
- **SYN flood**: Exhausts server backlog
- **SYN cookies**: Defends against SYN flood
- **XDP**: Drops attack traffic at wire speed
- **Connection tracking**: Can be exhausted
- **Rate limiting**: Throttles excessive traffic

### Performance Tuning
- **TSO/GSO**: Reduces CPU overhead for TX
- **GRO**: Reduces CPU overhead for RX
- **RSS/RPS**: Distributes load across CPUs
- **NAPI**: Reduces interrupt overhead
- **Zero-copy**: Eliminates data copying
- **Jumbo frames**: Larger MTU for efficiency

### Debugging/Troubleshooting
- **Wireshark**: Packet capture and analysis
- **Ping/ICMP**: Connectivity testing
- **Traceroute**: Path discovery
- **Checksum**: Detects corruption
- **Retransmission**: Indicates packet loss
- **SACK**: Shows selective loss recovery

### Containers/Virtualization
- **Network namespace**: Isolated network stack
- **VLAN**: Network segmentation
- **Loopback**: Per-namespace lo interface
- **Netlink**: Interface configuration
- **Routing**: Independent routing table per namespace

### Security
- **Firewall**: iptables/netfilter packet filtering
- **NAT**: Address hiding
- **Connection tracking**: Stateful firewall
- **ARP spoofing**: Man-in-the-middle attack
- **MAC filtering**: Access control (weak)
- **eBPF**: Programmable security policies

---

## Further Reading

For deeper understanding of these terms in context, see:
- [What is Kernel Networking?](./01-what-is-kernel-networking.md)
- [Packet Flow Through the Kernel](./03-packet-flow.md)
- [Key Functions and Structures](./04-key-functions.md)

---

*This glossary is a living document. Terms will be added as the documentation expands.*

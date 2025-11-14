# Packet Flow Through the Linux Kernel

This document traces the journey of a network packet as it flows through the Linux kernel.

## 🌟 The Simple Story (Ages 6-12)

### The Journey of a Message

Let's follow a message from your computer to your friend's computer!

#### Sending a Message (Outgoing Journey)

```
You → Type message in app (like Discord)
  ↓
📝 App puts message in an envelope
  ↓
📦 Kernel adds more envelopes (like Russian nesting dolls!)
  ↓
🏷️ Writes address on the outside
  ↓
🚚 Puts it on a truck (network card)
  ↓
📡 Truck drives through wires/WiFi to the internet
  ↓
🌐 Travels across the internet...
  ↓
📬 Arrives at your friend's computer
```

#### Receiving a Message (Incoming Journey)

```
📡 Message arrives at network card
  ↓
🔔 Network card rings a bell: "New message!"
  ↓
📦 Kernel worker picks it up
  ↓
✉️ Opens outer envelope (checks address)
  ↓
✉️ Opens next envelope (checks what type)
  ↓
✉️ Opens last envelope (gets actual message)
  ↓
📬 Puts message in app's mailbox
  ↓
🎉 Your friend sees your message!
```

### The Envelopes (Layers)

Imagine putting a letter inside multiple envelopes:
1. **Inner envelope**: Your actual message
2. **Middle envelope**: Instructions (TCP: "make sure this arrives!")
3. **Outer envelope**: Address (IP: "send to 192.168.1.5")
4. **Shipping label**: Neighbor's address (MAC: for the next hop)

Each "post office" (router/computer) along the way opens the outer envelope, reads where to send it next, and passes it along!

## 📚 The Student's Guide (Ages 13-18)

### The Two Journeys

Network communication has two directions:
- **TX (Transmit)**: Sending data OUT
- **RX (Receive)**: Getting data IN

Let's trace both!

### Sending Data (TX Path)

#### Step 1: Application Sends
Your application (like a web browser) calls a function:
```c
send(socket, "Hello", 5, 0);
```

#### Step 2: Socket Layer
- The kernel receives this system call
- Finds the socket you're using
- Prepares the data for the network

#### Step 3: Transport Layer (TCP/UDP)
- **If TCP**: Breaks data into segments, adds sequence numbers
- **If UDP**: Just wraps data quickly
- Adds source and destination **ports** (like apartment numbers)

```
Your data: "Hello"
After TCP: [TCP Header][Hello]
           ↑
           Contains: source port, dest port, sequence number
```

#### Step 4: Network Layer (IP)
- Adds IP header with addresses
- Figures out which interface to use (WiFi, Ethernet, etc.)

```
After IP: [IP Header][TCP Header][Hello]
          ↑
          Contains: source IP, destination IP
```

#### Step 5: Link Layer (Ethernet/WiFi)
- Adds MAC address (hardware address)
- Calculates checksum

```
Final: [Ethernet][IP Header][TCP Header][Hello][Checksum]
       ↑
       Contains: source MAC, dest MAC
```

#### Step 6: Device Driver
- Tells the network card to send it
- Network card puts electrical/radio signals on the wire/air!

### Receiving Data (RX Path)

#### Step 1: Packet Arrives
- Network card receives electrical signals
- Converts to digital data
- Stores in memory using DMA (Direct Memory Access)

#### Step 2: Interrupt
- Network card tells CPU: "Hey! New packet!"
- Or CPU checks periodically (polling)

#### Step 3: Link Layer Processing
- Kernel reads the Ethernet/WiFi header
- Checks: "Is this packet for me?" (MAC address)
- Removes the outer layer

```
Received: [Ethernet][IP Header][TCP Header][Hello][Checksum]
After:               [IP Header][TCP Header][Hello]
```

#### Step 4: Network Layer (IP)
- Checks IP address: "Yes, this is my IP!"
- Checks what protocol: "Oh, it's TCP!"
- Removes IP header

```
After:                          [TCP Header][Hello]
```

#### Step 5: Transport Layer (TCP/UDP)
- Checks port number: "Port 80? That's the web server!"
- Finds the right socket
- **If TCP**: Checks sequence number, sends acknowledgment
- Removes TCP header

```
After:                                      [Hello]
```

#### Step 6: Socket Buffer
- Puts data in the socket's receive buffer
- Wakes up the application waiting for data

#### Step 7: Application Receives
```c
recv(socket, buffer, 100, 0);  // Gets "Hello"
```

### Visual Flow Summary

```
SENDING (TX):
Application
    ↓ send()
  Socket Layer
    ↓
  TCP/UDP Layer     → Adds port numbers
    ↓
  IP Layer          → Adds IP addresses
    ↓
  Ethernet Layer    → Adds MAC addresses
    ↓
  Driver            → Controls hardware
    ↓
  Network Card      → Physical signals
    ↓
   📡 WIRE/WIFI 📡


RECEIVING (RX):
   📡 WIRE/WIFI 📡
    ↓
  Network Card      → Receives signals
    ↓ Interrupt/Poll
  Driver            → Handles hardware
    ↓
  Ethernet Layer    → Checks MAC, removes header
    ↓
  IP Layer          → Checks IP, removes header
    ↓
  TCP/UDP Layer     → Checks port, removes header
    ↓
  Socket Layer      → Finds right socket
    ↓
  Application       → recv()
```

## 🎓 The Practitioner's View (University Level)

### Detailed TX Path with Function Calls

```
User Space:
  Application
    ↓
    write() / send() / sendto() / sendmsg()
    ↓
  ════════════════════════════════════════
    System Call Interface
  ════════════════════════════════════════
    ↓
Kernel Space:

  Socket Layer (net/socket.c)
    ↓
    sys_sendto() / sys_sendmsg()
    ↓
    sock_sendmsg()
    ↓
    sock->ops->sendmsg()
    ↓

  Transport Layer (net/ipv4/)
    ↓
    TCP: tcp_sendmsg()
    UDP: udp_sendmsg()
    ↓
    [Build segment/datagram]
    ↓
    ip_queue_xmit() [TCP]
    ip_send_skb() [UDP]
    ↓

  Network Layer (net/ipv4/)
    ↓
    ip_local_out()
    ↓
    __ip_local_out()
    ↓
    NF_HOOK(NF_INET_LOCAL_OUT, ...) [iptables/netfilter]
    ↓
    dst_output() [routing decision]
    ↓
    ip_output()
    ↓
    ip_finish_output()
    ↓
    ip_finish_output2() [ARP, neighbor lookup]
    ↓

  Link Layer (net/core/dev.c)
    ↓
    dev_queue_xmit()
    ↓
    __dev_queue_xmit()
    ↓
    dev_hard_start_xmit()
    ↓
    netdev_start_xmit()
    ↓
    ops->ndo_start_xmit() [driver]
    ↓

  Device Driver (drivers/net/)
    ↓
    [Driver-specific transmission]
    ↓
    DMA setup, descriptor rings
    ↓
    Tell NIC to transmit
    ↓

Hardware:
    Network Interface Card
    ↓
    Physical Medium (Ethernet/WiFi)
```

### Detailed RX Path with Function Calls

```
Hardware:
    Physical Medium (Ethernet/WiFi)
    ↓
    Network Interface Card
    ↓
    DMA to memory (ring buffers)
    ↓
    Raise Interrupt (or NAPI poll)
    ↓

Kernel Space:

  Interrupt Handler (drivers/net/)
    ↓
    Disable interrupts
    ↓
    Schedule NAPI poll
    ↓

  NAPI Poll (New API - net/core/dev.c)
    ↓
    driver_poll() [driver-specific]
    ↓
    [For each packet in budget]
    ↓
    Build sk_buff
    ↓
    netif_receive_skb()
    ↓
    __netif_receive_skb()
    ↓
    __netif_receive_skb_core()
    ↓

  Link Layer Processing
    ↓
    [Check destination MAC]
    ↓
    [Identify protocol: IPv4, IPv6, ARP, etc.]
    ↓
    deliver_skb(pt_prev->func)
    ↓

  Network Layer (net/ipv4/ip_input.c)
    ↓
    ip_rcv()
    ↓
    NF_HOOK(NF_INET_PRE_ROUTING, ...) [iptables]
    ↓
    ip_rcv_finish()
    ↓
    ip_route_input() [routing decision]
    ↓
    dst->input() → ip_local_deliver()
    ↓
    ip_local_deliver_finish()
    ↓
    [Look up protocol: TCP=6, UDP=17, etc.]
    ↓

  Transport Layer
    ↓
    TCP: tcp_v4_rcv() (net/ipv4/tcp_ipv4.c)
    UDP: udp_rcv() (net/ipv4/udp.c)
    ↓
    [Look up socket by 4-tuple]
    (src IP, src port, dst IP, dst port)
    ↓
    tcp_v4_do_rcv()
    ↓
    tcp_rcv_established() [for established connections]
    ↓
    tcp_data_queue() [queue data]
    ↓

  Socket Layer
    ↓
    sock_queue_rcv_skb()
    ↓
    __skb_queue_tail(&sk->sk_receive_queue)
    ↓
    sk->sk_data_ready() [wake up waiting process]
    ↓

  ════════════════════════════════════════
    Process Wakeup
  ════════════════════════════════════════
    ↓
User Space:
    Application
    ↓
    recv() / read() / recvfrom() / recvmsg()
    ↓
    [Copy data from kernel to user space]
```

### Key Data Structures in the Flow

#### sk_buff (Socket Buffer)
The packet representation that travels through the stack:

```c
struct sk_buff {
    struct net_device    *dev;         // Incoming/outgoing device

    // Header pointers (offsets from head)
    __u16                transport_header;  // TCP/UDP header
    __u16                network_header;    // IP header
    __u16                mac_header;        // Ethernet header

    // Buffer pointers
    unsigned char        *head;        // Buffer start
    unsigned char        *data;        // Current data start
    unsigned char        *tail;        // Current data end
    unsigned char        *end;         // Buffer end

    unsigned int         len;          // Data length
    __u32                priority;     // QoS priority

    struct sock          *sk;          // Associated socket
    // ... many more fields
};
```

#### Socket (struct sock)
Represents a communication endpoint:

```c
struct sock {
    int                  sk_protocol;     // IPPROTO_TCP, IPPROTO_UDP
    unsigned short       sk_family;       // AF_INET, AF_INET6

    struct sk_buff_head  sk_receive_queue;   // Incoming data
    struct sk_buff_head  sk_write_queue;     // Outgoing data

    void                 (*sk_data_ready)(struct sock *sk);
    void                 (*sk_write_space)(struct sock *sk);

    __be16               sk_dport;        // Destination port
    __u16                sk_num;          // Local port
    __be32               sk_daddr;        // Destination IP
    // ... extensive fields
};
```

### Context Switches

Notice the transitions:

1. **User → Kernel** (TX): System call (overhead: ~100-300 ns)
2. **Interrupt → Kernel** (RX): Hardware interrupt (overhead: variable)
3. **Kernel → User** (RX): Return from system call / wakeup

Minimizing these transitions is crucial for performance!

## 🔬 The Deep Dive (Master's Level)

### The sk_buff Journey in Detail

#### TX Path: sk_buff Creation and Modification

```c
// 1. Allocate sk_buff
struct sk_buff *skb = sock_alloc_send_skb(sk, size, flags, &err);

// 2. Reserve space for headers
skb_reserve(skb, header_len);

// 3. Copy data from user space
err = skb_copy_to_page(sk, user_addr, skb, page, off, copy);

// 4. Transport layer adds header
skb_push(skb, sizeof(struct tcphdr));
th = tcp_hdr(skb);  // Returns pointer to pushed area
// Fill TCP header fields

// 5. Network layer adds header
skb_push(skb, sizeof(struct iphdr));
iph = ip_hdr(skb);
// Fill IP header fields

// 6. Link layer adds header
skb_push(skb, ETH_HLEN);
eth = eth_hdr(skb);
// Fill Ethernet header

// 7. Transmit
dev_queue_xmit(skb);
```

The beauty: **No data copying**, just pointer manipulation!

```
Initial after alloc:
head                                    end
 |                                       |
 v                                       v
 [  reserved for headers  ][  data  ][  ]
                           ^         ^
                          data     tail

After TCP header push:
head                                    end
 |                                       |
 v                                       v
 [  reserved  ][TCP hdr][  data  ][     ]
              ^                   ^
             data                tail

After IP header push:
head                                    end
 |                                       |
 v                                       v
 [  reserved][IP][TCP][  data  ][       ]
            ^                   ^
           data                tail
```

#### RX Path: sk_buff Processing

```c
// 1. Driver allocates sk_buff
skb = netdev_alloc_skb(dev, packet_size);
// DMA copies packet data into skb->data

// 2. Driver sets initial pointers
skb_put(skb, packet_length);  // Extends tail
skb_reset_mac_header(skb);
skb->protocol = eth_type_trans(skb, dev);
skb_reset_network_header(skb);

// 3. Pass to network stack
netif_receive_skb(skb);

// 4. IP layer processes
iph = ip_hdr(skb);
// Validate header, checksum
skb_pull(skb, ip_hdrlen);  // Remove IP header

// 5. TCP layer processes
th = tcp_hdr(skb);
// Validate, process options
skb_pull(skb, tcp_hdrlen);  // Remove TCP header

// 6. Queue to socket
skb_queue_tail(&sk->sk_receive_queue, skb);

// 7. Copy to user space (later, when app calls recv)
err = skb_copy_datagram_iter(skb, offset, iter, len);
```

### Performance Critical Sections

#### 1. Receive Path Optimizations

**NAPI (New API) Mechanism**:

```c
// Traditional interrupt-driven (bad under load):
for each packet:
    Interrupt
    Context switch to kernel
    Process one packet
    Return from interrupt
// → Huge interrupt overhead!

// NAPI (good under load):
First packet → Interrupt
Disable interrupts
Schedule polling
for each packet in budget:
    Poll for packet (no interrupt)
    Process packet
    if no more packets or budget exhausted:
        break
Re-enable interrupts
// → Amortize interrupt cost over many packets
```

Implementation:
```c
static int driver_poll(struct napi_struct *napi, int budget)
{
    int work = 0;

    while (work < budget) {
        struct sk_buff *skb = get_next_rx_packet();
        if (!skb)
            break;

        netif_receive_skb(skb);
        work++;
    }

    if (work < budget) {
        napi_complete(napi);
        enable_interrupts();
    }

    return work;
}

// Interrupt handler
static irqreturn_t driver_interrupt(int irq, void *dev_id)
{
    disable_interrupts();
    napi_schedule(&priv->napi);
    return IRQ_HANDLED;
}
```

**Receive Packet Steering (RPS)**:
Distribute processing across CPUs:

```c
// Hash packet 4-tuple to CPU
cpu = get_rps_cpu(skb->dev, skb, &rflow);

if (cpu != current_cpu) {
    // Enqueue to different CPU
    enqueue_to_backlog(skb, cpu, &rflow->last_qtail);
}
```

#### 2. Transmit Path Optimizations

**TCP Segmentation Offload (TSO/GSO)**:

```c
// Application sends large chunk
send(sock, big_data, 64KB, 0);

// Traditional approach (bad):
// Kernel splits into ~45 packets (1500 byte MTU)
// Process each packet through the stack (45x overhead)

// With GSO (good):
// Kernel creates ONE large skb
// Passes through stack once
// Hardware or software segments at the last moment
// → 1/45th the overhead!

skb_shinfo(skb)->gso_size = mss;
skb_shinfo(skb)->gso_type = SKB_GSO_TCPV4;
```

**Zero-Copy Transmission**:

```c
// Traditional (involves copy):
send(sock, user_buffer, size, 0);
// Kernel copies from user_buffer to kernel buffer

// Zero-copy with MSG_ZEROCOPY:
send(sock, user_buffer, size, MSG_ZEROCOPY);
// Kernel pins user pages
// DMA directly from user memory
// Notifies application when DMA completes

// Or sendfile (page cache to socket):
sendfile(sock_fd, file_fd, offset, count);
// No copy: pages sent directly from page cache
```

#### 3. Socket Lookup Optimization

Finding the right socket for incoming packets is critical:

```c
// Hash table lookup by 4-tuple
struct sock *__inet_lookup_established(net, hashinfo,
                                       saddr, sport,
                                       daddr, dport,
                                       dif, sdif)
{
    unsigned int hash = inet_ehashfn(net, daddr, dport,
                                      saddr, sport);
    struct inet_ehash_bucket *head = &hashinfo->ehash[hash];

    // RCU read-side lock (no traditional locking!)
    rcu_read_lock();

    sk_for_each_rcu(sk, &head->chain) {
        if (likely(INET_MATCH(sk, net, hash, daddr, saddr,
                             dport, sport, dif, sdif))) {
            goto found;
        }
    }
    sk = NULL;
found:
    rcu_read_unlock();
    return sk;
}
```

Using RCU (Read-Copy-Update) allows lock-free reads!

### Queueing Disciplines (QoS)

Packets don't always go straight to the device:

```c
// Transmit path with qdisc
dev_queue_xmit(skb)
    → __dev_queue_xmit()
    → qdisc = rcu_dereference(dev->qdisc)
    → q->enqueue(skb, q)  // Enqueue to qdisc
    → qdisc_run(q)        // Process qdisc
    → sch_direct_xmit()   // Dequeue and transmit
```

Types of qdiscs:
- **pfifo_fast**: Priority FIFO (default)
- **tbf**: Token Bucket Filter (rate limiting)
- **htb**: Hierarchical Token Bucket
- **fq_codel**: Fair Queue + Controlled Delay (modern default)

### Netfilter/iptables Integration

Packets pass through netfilter hooks:

```c
// NF_HOOK macro in the stack
static inline int NF_HOOK(uint8_t pf, unsigned int hook,
                          struct net *net, struct sock *sk,
                          struct sk_buff *skb, ...)
{
    int ret = nf_hook(pf, hook, net, sk, skb, ...);
    if (ret == 1)  // ACCEPT
        return okfn(net, sk, skb);  // Continue
    return ret;
}

// Example: IP receive path
ip_rcv(skb)
    → NF_HOOK(NFPROTO_IPV4, NF_INET_PRE_ROUTING, ...)
    → ip_rcv_finish()
    → ip_local_deliver()
    → NF_HOOK(NFPROTO_IPV4, NF_INET_LOCAL_IN, ...)
```

Hook points:
- NF_INET_PRE_ROUTING: Before routing decision
- NF_INET_LOCAL_IN: Destined for local machine
- NF_INET_FORWARD: To be forwarded
- NF_INET_LOCAL_OUT: From local machine
- NF_INET_POST_ROUTING: After routing decision

## 🧬 The Source Truth (PhD/Expert Level)

### Cache-Line Optimization

Network stack is heavily optimized for CPU cache:

```c
// sk_buff structure is carefully organized
struct sk_buff {
    // First cache line: hot fields accessed on every packet
    union {
        struct {
            struct sk_buff *next;
            struct sk_buff *prev;
            union {
                struct net_device *dev;
                unsigned long dev_scratch;
            };
        };
        struct rb_node rbnode;
        struct list_head list;
    };
    // ... arranged to minimize cache misses
} __aligned(8);  // Align to cache boundaries
```

### Per-CPU Data Structures

Avoid cache line bouncing:

```c
// Each CPU has its own queues
struct softnet_data {
    struct sk_buff_head input_pkt_queue;     // Backlog
    struct Qdisc           *output_queue;     // TX queue
    struct sk_buff_head    process_queue;     // RPS
    // ... per-CPU, no locking needed between CPUs
};

DEFINE_PER_CPU_ALIGNED(struct softnet_data, softnet_data);

// Access local CPU's data
struct softnet_data *sd = this_cpu_ptr(&softnet_data);
```

### Lock-Free Operations with RCU

```c
// Writer (rare):
void update_route(struct dst_entry *new_dst)
{
    struct dst_entry *old;

    old = rcu_dereference_protected(rt->dst, lockdep_is_held(&rt_lock));
    rcu_assign_pointer(rt->dst, new_dst);

    synchronize_rcu();  // Wait for readers
    kfree(old);
}

// Reader (common):
void forward_packet(struct sk_buff *skb)
{
    struct dst_entry *dst;

    rcu_read_lock();
    dst = rcu_dereference(rt->dst);
    // Use dst...
    rcu_read_unlock();
    // No traditional locking overhead!
}
```

### Memory Ordering and Barriers

Ensuring correct behavior on multicore systems:

```c
// In ring buffer producer (driver):
buffer[index] = skb;
smp_wmb();  // Write memory barrier
WRITE_ONCE(prod_index, index + 1);

// In ring buffer consumer (stack):
index = READ_ONCE(prod_index);
smp_rmb();  // Read memory barrier
skb = buffer[index];
```

### Instrumentation and Tracing

#### Tracepoints

```c
// Defined in the stack
DECLARE_TRACE(netif_receive_skb,
              TP_PROTO(struct sk_buff *skb),
              TP_ARGS(skb));

// Called when packet is received
trace_netif_receive_skb(skb);

// User can attach eBPF programs to this tracepoint
```

#### Performance Counters

```c
// Per-device statistics
struct rtnl_link_stats64 {
    __u64 rx_packets;
    __u64 tx_packets;
    __u64 rx_bytes;
    __u64 tx_bytes;
    __u64 rx_errors;
    __u64 tx_errors;
    // ...
};

// Accessed via /sys/class/net/eth0/statistics/
```

### Advanced Packet Processing: XDP (eXpress Data Path)

```c
// eBPF program attached to driver
SEC("xdp")
int xdp_filter(struct xdp_md *ctx)
{
    void *data_end = (void *)(long)ctx->data_end;
    void *data = (void *)(long)ctx->data;
    struct ethhdr *eth = data;

    // Bounds check (eBPF verifier requires this)
    if ((void *)(eth + 1) > data_end)
        return XDP_PASS;

    // Check protocol
    if (eth->h_proto == htons(ETH_P_IP)) {
        struct iphdr *iph = (void *)(eth + 1);
        if ((void *)(iph + 1) > data_end)
            return XDP_PASS;

        // Drop packets from specific IP
        if (iph->saddr == htonl(0x0a000001)) // 10.0.0.1
            return XDP_DROP;
    }

    return XDP_PASS;  // Pass to normal stack
}

// Return codes:
// XDP_DROP: Drop packet (no sk_buff allocation!)
// XDP_PASS: Pass to normal stack
// XDP_TX: Transmit back out same interface
// XDP_REDIRECT: Redirect to another interface
```

XDP advantages:
- Processes packets **before** sk_buff allocation
- Runs in driver context
- Can drop packets with minimal overhead
- Used for DDoS mitigation, load balancing

### Case Study: TCP Fast Path

The TCP stack has a "fast path" for the common case:

```c
int tcp_rcv_established(struct sock *sk, struct sk_buff *skb)
{
    struct tcp_sock *tp = tcp_sk(sk);

    // Fast path check
    if (tp->rcv_nxt == TCP_SKB_CB(skb)->seq &&  // Expected sequence
        !tcp_hdr(skb)->syn &&                     // Not SYN
        !tcp_hdr(skb)->fin &&                     // Not FIN
        !tcp_hdr(skb)->rst &&                     // Not RST
        tp->rcv_wnd &&                            // Window open
        // ... other conditions
       ) {
        // FAST PATH: just queue data
        tcp_data_queue(sk, skb);
        tcp_data_snd_check(sk);
        return 0;
    }

    // Slow path: handle out-of-order, flags, etc.
    tcp_slow_path_processing(...);
}
```

This optimization handles ~99% of packets in established connections!

### Understanding Packet Latency

Typical latencies in the stack (on modern hardware):

```
Component                    Latency
──────────────────────────────────────
NIC DMA                      ~200 ns
Interrupt delivery           ~500 ns
Driver processing            ~300 ns
IP layer                     ~100 ns
TCP layer                    ~200 ns
Socket wakeup                ~500 ns
Context switch to app        ~1-2 µs
──────────────────────────────────────
Total RX path                ~3-4 µs

For comparison:
L1 cache reference           0.5 ns
L2 cache reference           7 ns
Main memory reference        100 ns
Send packet over 1 Gbps      1,500 ns (for 1500 bytes)
Round trip in datacenter     ~500 µs
Round trip internet coast-to-coast    ~50 ms
```

Latency optimization targets:
- Keep data in cache (locality)
- Minimize context switches
- Batch processing where possible
- Use XDP for ultra-low latency (sub-µs)

## 📊 Visual Summary

### Complete Flow Diagram

```
═══════════════════════════════════════════════════════════════

                         TRANSMIT (TX)

═══════════════════════════════════════════════════════════════

User Space:

    Application: write()/send()
         │
         │ System Call
         ▼
─────────────────── Kernel Space Boundary ─────────────────────
         │
         ▼
    Socket Layer: sock_sendmsg()
         │
         ▼
    Transport Layer: tcp_sendmsg() / udp_sendmsg()
         │ [Create sk_buff]
         │ [Add TCP/UDP header]
         ▼
    Network Layer: ip_queue_xmit()
         │ [Add IP header]
         │ [Routing decision]
         ▼
    Netfilter: NF_INET_LOCAL_OUT
         │ [iptables rules]
         ▼
    QDisc: qdisc_enqueue()
         │ [Traffic shaping]
         ▼
    Link Layer: dev_queue_xmit()
         │ [Add Ethernet header]
         ▼
    Device Driver: ndo_start_xmit()
         │ [Setup DMA]
         ▼
    Hardware: Network Card
         │
         ▼
    Physical: Electrical/Radio Signals → Network

═══════════════════════════════════════════════════════════════

                         RECEIVE (RX)

═══════════════════════════════════════════════════════════════

    Physical: Network → Electrical/Radio Signals
         │
         ▼
    Hardware: Network Card
         │ [DMA to memory]
         │ [Raise interrupt or NAPI]
         ▼
    Device Driver: NAPI poll / Interrupt handler
         │ [Build sk_buff]
         ▼
    Link Layer: netif_receive_skb()
         │ [Check MAC address]
         │ [Identify protocol]
         ▼
    Network Layer: ip_rcv()
         │ [Validate IP header]
         │ [Routing decision]
         ▼
    Netfilter: NF_INET_LOCAL_IN
         │ [iptables rules]
         ▼
    Transport Layer: tcp_v4_rcv() / udp_rcv()
         │ [Socket lookup]
         │ [Validate segment]
         │ [Remove TCP/UDP header]
         ▼
    Socket Layer: sock_queue_rcv_skb()
         │ [Queue to socket]
         │ [Wake up process]
         ▼
─────────────────── Kernel Space Boundary ─────────────────────
         │
         │ Return from recv()
         ▼
User Space:

    Application: read()/recv()

═══════════════════════════════════════════════════════════════
```

## 📚 Summary & Key Takeaways

| Level | Understanding |
|-------|--------------|
| 🌟 Beginner | Packets are like letters in envelopes; each layer adds/removes an envelope |
| 📚 Student | Data flows through layers, each adding headers; receive is reverse of transmit |
| 🎓 Practitioner | sk_buff travels through the stack via function calls; context switches at boundaries |
| 🔬 Advanced | Zero-copy, NAPI polling, RCU locking, per-CPU data structures optimize the critical path |
| 🧬 Expert | Cache-aware design, XDP bypass, eBPF programmability, lock-free fast paths |

## 🔗 Related Documentation

- [What is Kernel Networking?](./01-what-is-kernel-networking.md)
- [Network Stack Architecture](./02-network-stack-architecture.md)
- [Key Functions and Structures](./04-key-functions.md)
- [Network Device Drivers](./05-network-drivers.md)

## ❓ Test Your Understanding

- 🌟 Can you explain why packets have multiple "envelopes"?
- 📚 What's the difference between the TX and RX paths?
- 🎓 At which layer does the kernel decide which socket receives a packet?
- 🔬 Why does NAPI use polling instead of interrupts under high load?
- 🧬 How does XDP achieve lower latency than the regular network stack?

---

*Understanding packet flow is fundamental to debugging network issues and optimizing performance!*

# Key Functions and Structures in Linux Kernel Networking

This document serves as a reference guide to the most important functions, structures, and concepts in the Linux kernel networking stack.

## 🌟 The Simple Story (Ages 6-12)

### The Important Helpers

Think of the Linux kernel networking like a big factory with many workers. Each worker has a specific job:

#### The Package (sk_buff)
- **What it is**: A box that holds your data as it travels
- **Special power**: Can grow and shrink without moving the data inside!
- **Like**: A magic envelope that can add labels without rewriting anything

#### The Socket (struct sock)
- **What it is**: Your mailbox where messages arrive
- **Job**: Keeps track of your conversations
- **Like**: A mailbox with your name on it

#### The Workers (Functions)
- **send()**: "Please send this letter!"
- **recv()**: "Give me my mail!"
- **ip_rcv()**: "I check addresses on packages"
- **tcp_sendmsg()**: "I make sure packages arrive safely"
- **dev_queue_xmit()**: "I put packages on trucks"

## 📚 The Student's Guide (Ages 13-18)

### Core Data Structures

#### 1. Socket Buffer (sk_buff)

The most important structure! Every packet is represented by an sk_buff.

```c
struct sk_buff {
    // Links for queues
    struct sk_buff *next;
    struct sk_buff *prev;

    // Associated socket
    struct sock *sk;

    // Network device
    struct net_device *dev;

    // Data pointers
    unsigned char *data;    // Current data position
    unsigned int len;       // Length of data

    // Header locations
    __u16 transport_header; // TCP/UDP header position
    __u16 network_header;   // IP header position
    __u16 mac_header;       // Ethernet header position
};
```

**Why it matters**:
- Represents a packet throughout its journey
- Allows adding headers without copying data
- Can be cloned (shared data) or copied

#### 2. Socket (struct sock)

Represents one end of a network connection.

```c
struct sock {
    // Protocol information
    unsigned short sk_family;    // AF_INET, AF_INET6
    int sk_protocol;             // TCP, UDP, etc.

    // Address information
    __be16 sk_dport;            // Destination port
    __u16 sk_num;               // Local port
    __be32 sk_daddr;            // Destination IP address

    // Data queues
    struct sk_buff_head sk_receive_queue;  // Incoming packets
    struct sk_buff_head sk_write_queue;    // Outgoing packets

    // Callbacks
    void (*sk_data_ready)(struct sock *sk);     // Data arrived
    void (*sk_write_space)(struct sock *sk);    // Space to write
};
```

**Why it matters**:
- Your application's connection is represented by this
- Stores all state for a communication channel
- Manages queues of incoming and outgoing data

#### 3. Network Device (struct net_device)

Represents a network interface (eth0, wlan0, etc.).

```c
struct net_device {
    char name[IFNAMSIZ];        // Interface name "eth0"

    unsigned char *dev_addr;     // MAC address
    unsigned int mtu;            // Max packet size

    // Operations
    const struct net_device_ops *netdev_ops;

    // Queues
    struct netdev_queue *_tx;    // Transmit queues
    struct netdev_rx_queue *_rx; // Receive queues

    unsigned long state;         // UP, DOWN, etc.
};
```

**Why it matters**:
- Represents your network hardware
- Different functions for WiFi vs Ethernet vs loopback
- Controls sending and receiving at the lowest level

### Important Functions

#### Transmit (Sending) Path

```c
// 1. System call entry
send(socket_fd, data, length, flags)
    ↓

// 2. Socket layer
sock_sendmsg(socket, msg)
    ↓

// 3. Transport layer
tcp_sendmsg(sock, msg, size)      // For TCP
udp_sendmsg(sock, msg, size)      // For UDP
    ↓

// 4. Network layer
ip_queue_xmit(sock, sk_buff)      // Add IP header
    ↓

// 5. Device layer
dev_queue_xmit(sk_buff)           // Send to device
    ↓

// 6. Driver
ndo_start_xmit(sk_buff, device)   // Hardware specific
```

#### Receive (Receiving) Path

```c
// 1. Hardware interrupt
[Network card receives packet]
    ↓

// 2. Driver
netif_rx(sk_buff)                 // Give to kernel
    ↓

// 3. Link layer
netif_receive_skb(sk_buff)        // Process packet
    ↓

// 4. Network layer
ip_rcv(sk_buff, ...)              // Process IP
    ↓

// 5. Transport layer
tcp_v4_rcv(sk_buff)               // For TCP
udp_rcv(sk_buff, ...)             // For UDP
    ↓

// 6. Socket layer
sock_queue_rcv_skb(sock, sk_buff) // Queue to socket
    ↓

// 7. System call return
recv(socket_fd, buffer, length, flags)
```

### Working with sk_buff

#### Adding Headers (Transmit)

```c
// Start with data
[        data        ]
     ^           ^
    data        tail

// Add TCP header
skb_push(skb, sizeof(struct tcphdr));
[TCP header][  data  ]
^                   ^
data               tail

// Add IP header
skb_push(skb, sizeof(struct iphdr));
[IP][TCP header][data]
^                     ^
data                 tail
```

#### Removing Headers (Receive)

```c
// Start with full packet
[Eth][IP][TCP][data]
^                  ^
data              tail

// Remove Ethernet header
skb_pull(skb, ETH_HLEN);
     [IP][TCP][data]
     ^             ^
    data          tail

// Remove IP header
skb_pull(skb, ip_hdrlen);
         [TCP][data]
         ^        ^
        data     tail
```

## 🎓 The Practitioner's View (University Level)

### Critical Functions by Subsystem

#### Socket Layer (net/socket.c)

```c
// Create a socket
SYSCALL_DEFINE3(socket, int, family, int, type, int, protocol)
    → sock_create(family, type, protocol, &sock)
    → Returns file descriptor

// Bind to address
SYSCALL_DEFINE3(bind, int, fd, struct sockaddr *, addr, int, addrlen)
    → sock->ops->bind(sock, addr, addrlen)

// Send data
SYSCALL_DEFINE6(sendto, int, fd, void *, buff, size_t, len, ...)
    → sock_sendmsg(sock, &msg)
    → sock->ops->sendmsg(sock, msg, len)

// Receive data
SYSCALL_DEFINE6(recvfrom, int, fd, void *, ubuf, size_t, size, ...)
    → sock_recvmsg(sock, &msg, flags)
    → sock->ops->recvmsg(sock, msg, size, flags)
```

**Key Point**: The socket layer provides a protocol-independent interface. The `ops` pointer dispatches to protocol-specific implementations.

#### TCP Layer (net/ipv4/tcp*.c)

```c
// TCP send
int tcp_sendmsg(struct sock *sk, struct msghdr *msg, size_t size)
{
    // 1. Check connection state
    if (sk->sk_state != TCP_ESTABLISHED)
        return -EPIPE;

    // 2. Build sk_buff with data
    skb = sk_stream_alloc_skb(sk, size, ...);

    // 3. Copy data from user
    skb_add_data(skb, &msg->msg_iter, copy);

    // 4. Push to send queue
    tcp_push(sk, flags, mss_now, TCP_NAGLE_PUSH);

    return copied;
}

// TCP receive
int tcp_v4_rcv(struct sk_buff *skb)
{
    // 1. Get TCP header
    struct tcphdr *th = tcp_hdr(skb);

    // 2. Find socket by (src_ip, src_port, dst_ip, dst_port)
    sk = __inet_lookup_skb(&tcp_hashinfo, skb,
                           th->source, th->dest);

    // 3. Process based on state
    if (sk->sk_state == TCP_ESTABLISHED)
        tcp_rcv_established(sk, skb);
    else
        tcp_rcv_state_process(sk, skb);

    return 0;
}

// TCP connection establishment
int tcp_v4_connect(struct sock *sk, struct sockaddr *uaddr, int addr_len)
{
    // 1. Set up addresses
    inet_sk(sk)->daddr = daddr;
    inet_sk(sk)->dport = dport;

    // 2. Set up routing
    rt = ip_route_connect(...);

    // 3. Send SYN
    tcp_connect(sk);

    return 0;
}
```

**State Machine**: TCP connections go through states:
- LISTEN → SYN_SENT → ESTABLISHED → FIN_WAIT → CLOSED

#### IP Layer (net/ipv4/ip*.c)

```c
// IP receive
int ip_rcv(struct sk_buff *skb, struct net_device *dev,
           struct packet_type *pt, struct net_device *orig_dev)
{
    struct iphdr *iph = ip_hdr(skb);

    // 1. Validate header
    if (iph->version != 4)
        goto drop;
    if (ip_fast_csum((u8 *)iph, iph->ihl))
        goto drop;

    // 2. Netfilter hook
    return NF_HOOK(NFPROTO_IPV4, NF_INET_PRE_ROUTING,
                   net, NULL, skb, dev, NULL,
                   ip_rcv_finish);
drop:
    kfree_skb(skb);
    return NET_RX_DROP;
}

// IP send
int ip_queue_xmit(struct sock *sk, struct sk_buff *skb,
                  struct flowi *fl)
{
    struct iphdr *iph;

    // 1. Build IP header
    skb_push(skb, sizeof(struct iphdr));
    iph = ip_hdr(skb);

    iph->version = 4;
    iph->ihl = 5;
    iph->tos = inet_sk(sk)->tos;
    iph->tot_len = htons(skb->len);
    iph->id = htons(inet_sk(sk)->inet_id++);
    iph->frag_off = 0;
    iph->ttl = ip_select_ttl(inet_sk(sk), &rt->dst);
    iph->protocol = sk->sk_protocol;
    iph->saddr = fl4->saddr;
    iph->daddr = fl4->daddr;

    // 2. Calculate checksum
    ip_send_check(iph);

    // 3. Send
    return ip_local_out(net, sk, skb);
}

// Routing decision
int ip_route_input(struct sk_buff *skb, __be32 daddr, __be32 saddr,
                   u8 tos, struct net_device *dev)
{
    // Look up routing table
    // Determine: local delivery, forward, or drop
    // Set skb->dst
}
```

#### Device Layer (net/core/dev.c)

```c
// Transmit to device
int dev_queue_xmit(struct sk_buff *skb)
{
    struct net_device *dev = skb->dev;
    struct netdev_queue *txq;

    // 1. Select queue
    txq = netdev_pick_tx(dev, skb, NULL);

    // 2. Queue to qdisc (traffic control)
    if (dev->qdisc)
        rc = __dev_queue_xmit(skb, txq);

    return rc;
}

// Receive from device
int netif_receive_skb(struct sk_buff *skb)
{
    // 1. RPS (Receive Packet Steering)
    cpu = get_rps_cpu(skb->dev, skb, &rflow);

    // 2. Deliver to protocol handlers
    return __netif_receive_skb(skb);
}

static int __netif_receive_skb_core(struct sk_buff *skb)
{
    struct packet_type *ptype;

    // 1. Determine protocol (IPv4, IPv6, ARP, etc.)
    type = skb->protocol;

    // 2. Deliver to protocol handler
    list_for_each_entry_rcu(ptype, &ptype_base[ntohs(type)], list) {
        if (ptype->type == type)
            deliver_skb(skb, ptype, orig_dev);
    }
}
```

### Important Data Structure Details

#### sk_buff Memory Layout

```c
struct sk_buff {
    /* First cache line (64 bytes on x86-64) */
    union {
        struct {
            struct sk_buff *next;           // 8 bytes
            struct sk_buff *prev;           // 8 bytes
            union {
                struct net_device *dev;     // 8 bytes
            };
        };
    };

    struct sock *sk;                        // 8 bytes
    union {
        ktime_t tstamp;                     // 8 bytes
    };

    char cb[48];                            // 48 bytes (control buffer)
    /* Total: 88 bytes - spans into 2nd cache line */

    /* Second cache line */
    unsigned long _skb_refdst;              // 8 bytes
    void (*destructor)(struct sk_buff *);   // 8 bytes
    // ...

    /* Headers and data pointers */
    unsigned char *head;     // Start of allocated buffer
    unsigned char *data;     // Start of valid data
    unsigned char *tail;     // End of valid data
    unsigned char *end;      // End of allocated buffer

    unsigned int len;        // Length of data
    unsigned int data_len;   // Length in paged data

    __u16 mac_header;        // Offset from head
    __u16 network_header;    // Offset from head
    __u16 transport_header;  // Offset from head
};
```

**Memory Diagram**:
```
Allocated memory:
┌─────────────────────────────────────────────────────────┐
│ headroom │ data │ tailroom │ skb_shared_info │
└─────────────────────────────────────────────────────────┘
^          ^      ^          ^
head       data   tail       end

Headroom: Space for adding headers
Data: Actual packet data
Tailroom: Space for expanding data
skb_shared_info: Metadata (for fragmented packets)
```

#### Protocol Operations Structure

```c
// Each protocol family implements these
struct proto_ops {
    int family;
    struct module *owner;
    int (*release)(struct socket *sock);
    int (*bind)(struct socket *sock, struct sockaddr *addr, int addrlen);
    int (*connect)(struct socket *sock, struct sockaddr *addr, int addrlen, int flags);
    int (*socketpair)(struct socket *sock1, struct socket *sock2);
    int (*accept)(struct socket *sock, struct socket *newsock, int flags);
    int (*getname)(struct socket *sock, struct sockaddr *addr, int *addrlen, int peer);
    int (*sendmsg)(struct socket *sock, struct msghdr *m, size_t total_len);
    int (*recvmsg)(struct socket *sock, struct msghdr *m, size_t total_len, int flags);
    // ...
};

// Example: TCP operations
const struct proto_ops inet_stream_ops = {
    .family        = PF_INET,
    .release       = inet_release,
    .bind          = inet_bind,
    .connect       = inet_stream_connect,
    .socketpair    = sock_no_socketpair,
    .accept        = inet_accept,
    .getname       = inet_getname,
    .sendmsg       = inet_sendmsg,
    .recvmsg       = inet_recvmsg,
    // ...
};
```

## 🔬 The Deep Dive (Master's Level)

### sk_buff Advanced Operations

#### Cloning vs Copying

```c
// Clone: Share data, copy metadata
struct sk_buff *skb_clone(struct sk_buff *skb, gfp_t gfp_mask)
{
    struct sk_buff *n;

    n = kmem_cache_alloc(skbuff_head_cache, gfp_mask);
    memcpy(n, skb, offsetof(struct sk_buff, tail));

    // Share the same data buffer
    skb->cloned = 1;
    atomic_inc(&(skb_shinfo(skb)->dataref));

    return n;  // Points to same data as skb
}

// Copy: Deep copy everything
struct sk_buff *skb_copy(const struct sk_buff *skb, gfp_t gfp_mask)
{
    struct sk_buff *n;

    n = alloc_skb(skb->end - skb->head + skb->data_len, gfp_mask);
    skb_copy_from_linear_data(skb, n->data, skb->len);
    // ... copy all metadata

    return n;  // Independent copy
}
```

**When to use**:
- **Clone**: Packet needs to go to multiple destinations (multicast)
- **Copy**: Need to modify data and keep original

#### Paged Data (Scatter-Gather)

For large transfers, data can be in pages:

```c
struct skb_shared_info {
    unsigned char nr_frags;       // Number of fragments
    unsigned short gso_size;      // For TSO/GSO
    unsigned short gso_type;

    skb_frag_t frags[MAX_SKB_FRAGS];  // Page fragments

    struct sk_buff *frag_list;    // List of additional sk_buffs
};

// Add a page fragment
skb_fill_page_desc(skb, i, page, offset, size);
```

**Benefits**:
- Zero-copy from user space or page cache
- Large packets without large linear buffers
- Important for sendfile() and DMA

#### Headroom Management

```c
// Allocate with headroom for headers
skb = alloc_skb(data_len + header_space, GFP_KERNEL);
skb_reserve(skb, header_space);  // Reserve headroom

// Later, add headers without reallocation
skb_push(skb, tcp_hdrlen);  // Add TCP header
skb_push(skb, ip_hdrlen);   // Add IP header
skb_push(skb, ETH_HLEN);    // Add Ethernet header
```

**Typical headroom**:
```
LL_MAX_HEADER = 96 bytes (enough for Ethernet + VLAN + etc.)
```

### Socket Lookup Performance

#### Hash Table Design

```c
struct inet_hashinfo {
    struct inet_ehash_bucket *ehash;  // Established connections
    unsigned int ehash_mask;           // Hash mask
    unsigned int ehash_locks;          // Lock granularity

    struct inet_bind_hashbucket *bhash;  // Bound sockets
    unsigned int bhash_size;

    struct inet_listen_hashbucket lhash2[INET_LHTABLE_SIZE];  // Listeners
};

// Hash function (4-tuple)
static inline unsigned int inet_ehashfn(struct net *net,
                                        __be32 laddr, __u16 lport,
                                        __be32 faddr, __be16 fport)
{
    return jhash_3words((__force __u32)laddr,
                       (__force __u32)faddr,
                       ((__u32)lport) << 16 | (__force __u32)fport,
                       net_hash_mix(net));
}
```

**Optimization**: Hash table sized to minimize collisions
- Typical: 16K to 256K buckets
- RCU-protected for lockless reads
- Per-bucket locking for writes

### Memory Management

#### sk_buff Allocation

```c
// Fast path: from cache
static inline struct sk_buff *alloc_skb(unsigned int size, gfp_t priority)
{
    return __alloc_skb(size, priority, 0, NUMA_NO_NODE);
}

struct sk_buff *__alloc_skb(unsigned int size, gfp_t gfp_mask,
                            int flags, int node)
{
    struct sk_buff *skb;
    u8 *data;

    // 1. Allocate sk_buff structure from cache
    skb = kmem_cache_alloc_node(skbuff_head_cache, gfp_mask, node);

    // 2. Allocate data buffer
    size = SKB_DATA_ALIGN(size) + SKB_DATA_ALIGN(sizeof(struct skb_shared_info));
    data = kmalloc_reserve(size, gfp_mask, node, &pfmemalloc);

    // 3. Initialize
    skb->head = data;
    skb->data = data;
    skb_reset_tail_pointer(skb);
    skb->end = skb->tail + size;

    return skb;
}
```

**Memory pools**:
- `skbuff_head_cache`: sk_buff structures (256 bytes each)
- Data buffers: Various sizes (typically 256, 512, 1024, 2048, 4096 bytes)
- NUMA-aware allocation for better cache locality

### Concurrency and Synchronization

#### Socket Locking

```c
void lock_sock(struct sock *sk)
{
    might_sleep();

    if (!spin_trylock_bh(&sk->sk_lock.slock)) {
        // Slow path: someone else holds lock
        __lock_sock(sk);
    }
}

// Usage in TCP
int tcp_sendmsg(struct sock *sk, struct msghdr *msg, size_t size)
{
    lock_sock(sk);  // Acquire socket lock

    // ... send logic ...

    release_sock(sk);  // Release socket lock
}
```

**Lock hierarchy**:
1. Socket lock: Protects socket state
2. Backlog lock: Protects packet backlog queue
3. Hash table locks: Protect hash bucket chains

#### RCU in Network Stack

```c
// Reader side (common case)
void lookup_socket(...)
{
    rcu_read_lock();

    sk = __inet_lookup(&tcp_hashinfo, ...);
    if (sk) {
        // Use sk
    }

    rcu_read_unlock();
    // No traditional locking!
}

// Writer side (rare)
void delete_socket(struct sock *sk)
{
    spin_lock(&hash_bucket->lock);

    hlist_del_rcu(&sk->sk_node);

    spin_unlock(&hash_bucket->lock);

    synchronize_rcu();  // Wait for all readers
    sock_put(sk);       // Free
}
```

**Why RCU**: Allows lockless reads (no cache line bouncing)

## 🧬 The Source Truth (PhD/Expert Level)

### Kernel Version Differences

Important changes across kernel versions:

#### 2.6.x → 3.x
- NAPI becomes standard (was optional)
- RPS/RFS added
- Early XDP experiments

#### 4.x
- XDP introduced (4.8+)
- TCP BBR congestion control (4.9+)
- Improved lock scalability

#### 5.x
- TCP/UDP GRO (Generic Receive Offload) improvements
- BPF socket lookup (5.2+)
- Multi-path TCP (5.6+)

#### 6.x
- BIG TCP (5.19+, larger GSO/GRO)
- Further XDP enhancements
- io_uring zero-copy networking

### Critical Source Files Reference

```
Core Structures:
include/linux/skbuff.h          - sk_buff definition
include/net/sock.h              - struct sock
include/linux/netdevice.h       - struct net_device

Socket Layer:
net/socket.c                    - Socket system calls
include/linux/net.h             - Socket definitions

TCP:
net/ipv4/tcp.c                  - Main TCP logic
net/ipv4/tcp_input.c            - TCP input processing
net/ipv4/tcp_output.c           - TCP output processing
net/ipv4/tcp_ipv4.c             - IPv4-specific TCP
include/net/tcp.h               - TCP definitions

UDP:
net/ipv4/udp.c                  - UDP implementation
include/net/udp.h               - UDP definitions

IP:
net/ipv4/ip_input.c             - IP input
net/ipv4/ip_output.c            - IP output
net/ipv4/route.c                - Routing
include/net/ip.h                - IP definitions

Device Layer:
net/core/dev.c                  - Device handling
net/core/skbuff.c               - sk_buff operations
drivers/net/*                   - Device drivers

Netfilter:
net/netfilter/*                 - Firewall/NAT
net/ipv4/netfilter/*            - IPv4 netfilter

eBPF/XDP:
kernel/bpf/*                    - BPF subsystem
net/core/filter.c               - Network BPF
```

### Performance Critical Code Paths

#### Fast Path Indicators

Look for these patterns in the code:

```c
// Branch prediction hints
if (likely(condition)) {
    // Fast path
} else {
    // Slow path
}

if (unlikely(error_condition)) {
    goto error_handling;
}

// Example from TCP
if (likely(TCP_SKB_CB(skb)->seq == tp->rcv_nxt &&
           !tcp_hdr(skb)->syn &&
           !tcp_hdr(skb)->fin &&
           // ... more fast path conditions
          )) {
    // Fast path: in-order data
    tcp_data_queue_fast(sk, skb);
} else {
    // Slow path: out-of-order, flags, etc.
    tcp_data_queue_slow(sk, skb);
}
```

#### Cache Line Optimization

```c
// Structures marked for cache alignment
struct inet_ehash_bucket {
    struct hlist_nulls_head chain;
} ____cacheline_aligned_in_smp;

// Per-CPU data (no cache bouncing)
DEFINE_PER_CPU_ALIGNED(struct softnet_data, softnet_data);
```

### Advanced Features

#### TCP Fast Open (TFO)

```c
// Client: Send data with SYN
int tcp_sendmsg_fastopen(struct sock *sk, struct msghdr *msg,
                         int *copied, size_t size)
{
    struct tcp_sock *tp = tcp_sk(sk);
    struct sockaddr *uaddr = msg->msg_name;

    // Send SYN + data
    if (tp->fastopen_req) {
        tcp_connect(sk);  // Sends SYN with data
    }

    return tcp_sendmsg_locked(sk, msg, size);
}

// Server: Accept data from SYN
int tcp_conn_request(struct sock *sk, struct sk_buff *skb)
{
    if (tcp_fastopen_cookie_valid(foc)) {
        // Create socket immediately (no 3-way handshake wait)
        child = tcp_fastopen_create_child(sk, skb, req);
    }
}
```

**Benefit**: Saves 1 RTT on connection establishment

#### Segmentation Offloads

```c
// TSO (TCP Segmentation Offload)
netdev_features_t features = dev->features;

if (features & NETIF_F_TSO) {
    // Hardware can segment
    skb_shinfo(skb)->gso_size = mss;
    skb_shinfo(skb)->gso_segs = DIV_ROUND_UP(len, mss);
    // Send large packet to hardware
} else if (features & NETIF_F_SG) {
    // Software segmentation (GSO)
    skb = tcp_gso_segment(skb, features);
}
```

#### Early Demux

Fast path for routing established connections:

```c
// In ip_rcv_finish
void ip_rcv_finish(struct sk_buff *skb)
{
    const struct iphdr *iph = ip_hdr(skb);

    // Early demux: try to find socket early
    if (sysctl_ip_early_demux) {
        const struct net_protocol *ipprot;
        int protocol = iph->protocol;

        ipprot = rcu_dereference(inet_protos[protocol]);
        if (ipprot && ipprot->early_demux) {
            ipprot->early_demux(skb);
            // Sets skb->sk early - avoids routing lookup!
        }
    }

    return dst_input(skb);
}
```

**Benefit**: Skip routing table lookup for established connections

### Debugging and Tracing

#### Dynamic Tracing with eBPF

```c
// Example: Trace all TCP retransmissions
SEC("tracepoint/tcp/tcp_retransmit_skb")
int trace_retransmit(struct trace_event_raw_tcp_event_sk_skb *ctx)
{
    struct sock *sk = (struct sock *)ctx->skaddr;
    __u16 sport = ctx->sport;
    __u16 dport = ctx->dport;

    bpf_printk("Retransmit: %d -> %d\n", sport, dport);

    return 0;
}
```

#### Static Tracepoints

```c
// In tcp_retransmit_skb
int tcp_retransmit_skb(struct sock *sk, struct sk_buff *skb, int segs)
{
    // ... retransmission logic ...

    trace_tcp_retransmit_skb(sk, skb);  // Tracepoint

    return err;
}
```

### Research Topics

#### 1. Kernel Bypass Technologies

- **DPDK**: User-space drivers, poll-mode
- **io_uring**: Async I/O with ring buffers
- **AF_XDP**: XDP with user-space delivery

#### 2. Congestion Control Evolution

```c
// Pluggable congestion control
struct tcp_congestion_ops {
    void (*init)(struct sock *sk);
    void (*release)(struct sock *sk);
    void (*cong_avoid)(struct sock *sk, u32 ack, u32 acked);
    u32  (*undo_cwnd)(struct sock *sk);
    void (*pkts_acked)(struct sock *sk, const struct ack_sample *sample);
    // ...
};

// Available algorithms:
// - Reno (classic)
// - Cubic (Linux default)
// - BBR (Google, latency-based)
// - DCTCP (datacenter)
```

#### 3. Multi-Path TCP (MPTCP)

Using multiple network paths simultaneously:
- Implemented as a socket type (IPPROTO_MPTCP)
- Transparent to applications
- Path management, scheduling, congestion control

## 📊 Quick Reference Tables

### System Call to Function Mapping

| System Call | Socket Layer | TCP | UDP | IP Layer |
|------------|--------------|-----|-----|----------|
| `socket()` | `sock_create()` | - | - | - |
| `bind()` | `sock->ops->bind()` | `inet_bind()` | `inet_bind()` | - |
| `connect()` | `sock->ops->connect()` | `tcp_v4_connect()` | `ip_connect()` | - |
| `send()` | `sock_sendmsg()` | `tcp_sendmsg()` | `udp_sendmsg()` | `ip_queue_xmit()` |
| `recv()` | `sock_recvmsg()` | `tcp_recvmsg()` | `udp_recvmsg()` | - |

### sk_buff Manipulation Functions

| Function | Purpose | Direction |
|----------|---------|-----------|
| `alloc_skb()` | Allocate new sk_buff | Both |
| `kfree_skb()` | Free sk_buff | Both |
| `skb_clone()` | Share data, copy metadata | Both |
| `skb_copy()` | Deep copy | Both |
| `skb_push()` | Add header (move data pointer back) | TX |
| `skb_pull()` | Remove header (move data pointer forward) | RX |
| `skb_put()` | Add data to tail | RX |
| `skb_trim()` | Remove data from tail | Both |
| `skb_reserve()` | Reserve headroom | TX |

### Header Access Macros

| Macro | Layer | Returns |
|-------|-------|---------|
| `eth_hdr(skb)` | Ethernet | `struct ethhdr *` |
| `ip_hdr(skb)` | IP | `struct iphdr *` |
| `ipv6_hdr(skb)` | IPv6 | `struct ipv6hdr *` |
| `tcp_hdr(skb)` | TCP | `struct tcphdr *` |
| `udp_hdr(skb)` | UDP | `struct udphdr *` |

## 📚 Summary

| Level | Key Takeaway |
|-------|-------------|
| 🌟 Beginner | sk_buff is the magic box, functions are workers with specific jobs |
| 📚 Student | Three main structures (sk_buff, sock, net_device) and clear TX/RX function chains |
| 🎓 Practitioner | Protocol-independent socket layer dispatches to protocol-specific implementations |
| 🔬 Advanced | Zero-copy via pointer manipulation, RCU for lockless reads, cache-aware design |
| 🧬 Expert | eBPF hooks, early demux, pluggable algorithms, kernel version evolution |

## 🔗 Related Documentation

- [What is Kernel Networking?](./01-what-is-kernel-networking.md)
- [Packet Flow Through the Kernel](./03-packet-flow.md)
- [Network Device Drivers](./05-network-drivers.md)

---

*This reference should serve as your go-to guide for understanding the building blocks of kernel networking!*

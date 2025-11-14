# Linux Network Engineering and Debugging

## What is a Linux Network Engineer?

A Linux Network Engineer is a specialized role that combines deep knowledge of networking protocols, Linux operating systems, and system administration. Unlike general network engineers who primarily work with network devices (routers, switches, firewalls), Linux Network Engineers focus on the intersection of Linux systems and networking—managing, optimizing, and troubleshooting network functionality at the operating system and kernel level.

### Core Competencies

**System-Level Networking:**
- Deep understanding of the Linux network stack from application layer down to hardware
- Kernel networking subsystems: netfilter, tc (traffic control), routing tables, network namespaces
- Socket programming and system calls (socket(), bind(), connect(), send(), recv())
- Network device drivers and hardware interface management

**Protocol Mastery:**
- TCP/IP stack implementation and behavior in Linux
- Deep packet inspection and protocol analysis
- Understanding of congestion control algorithms, flow control mechanisms
- IPv4/IPv6 dual-stack environments
- Advanced routing protocols (BGP, OSPF) implementation on Linux

**Performance and Optimization:**
- Network throughput optimization (TCP window scaling, buffer tuning)
- Interrupt handling and CPU affinity for network interfaces
- Zero-copy techniques, kernel bypass (DPDK, XDP)
- Connection tracking optimization for high-traffic environments

**Security:**
- iptables/nftables firewall configuration and optimization
- Network segmentation using network namespaces and VRFs
- IPsec VPN configuration and troubleshooting
- Intrusion detection and packet filtering

**Automation and Infrastructure:**
- Scripting (Python, Bash, Perl) for network automation
- Configuration management (Ansible, Puppet, Chef) for network services
- Container networking (Docker, Kubernetes CNI plugins)
- Cloud networking (AWS VPC, GCP VPC, Azure VNet)

**Debugging and Troubleshooting:**
- Advanced packet capture and analysis
- Kernel tracing and debugging tools
- Performance profiling and bottleneck identification
- Root cause analysis of complex network issues

---

## The Depth of Linux Network Engineering

### Layer 1: Application and Service Level

**DNS Services:**
- BIND, dnsmasq, CoreDNS configuration and troubleshooting
- Understanding DNS query flow, caching mechanisms, DNSSEC
- Debugging DNS resolution issues (systemd-resolved, /etc/nsswitch.conf)

**Load Balancers and Proxies:**
- HAProxy, Nginx, Envoy configuration
- Layer 4 vs Layer 7 load balancing
- Connection pooling, health checks, session persistence
- SSL/TLS termination and certificate management

**Network Services:**
- DHCP servers (dhcpd, dnsmasq)
- NTP time synchronization
- SNMP monitoring and SNMP trap handling
- Syslog network logging

### Layer 2: System Configuration

**Interface Management:**
- NetworkManager, systemd-networkd, netplan
- Interface bonding (802.3ad LACP, active-backup modes)
- VLAN tagging (802.1Q) and trunk configuration
- Bridge configuration for virtualization

**Routing:**
- Static routes vs dynamic routing
- Policy-based routing (iproute2 rules)
- Multi-path routing (ECMP)
- Source-based routing

**Traffic Control (tc):**
- Queuing disciplines (pfifo_fast, fq_codel, HTB, TBF)
- Traffic shaping and rate limiting
- Classification and filtering
- Priority queuing (QoS implementation)

### Layer 3: Kernel and Network Stack

**Netfilter/iptables/nftables:**
- Connection tracking (conntrack) and state management
- NAT (SNAT, DNAT, masquerading)
- Packet filtering chains and rule optimization
- Custom chains and complex rule sets

**Kernel Network Parameters (sysctl):**
```bash
# TCP tuning
net.ipv4.tcp_window_scaling
net.ipv4.tcp_timestamps
net.ipv4.tcp_sack
net.ipv4.tcp_congestion_control

# Buffer sizes
net.core.rmem_max
net.core.wmem_max
net.ipv4.tcp_rmem
net.ipv4.tcp_wmem

# Connection tracking
net.netfilter.nf_conntrack_max
net.netfilter.nf_conntrack_tcp_timeout_established

# Routing
net.ipv4.ip_forward
net.ipv4.conf.all.rp_filter
```

**Network Namespaces:**
- Isolation of network stacks for containers
- Virtual ethernet pairs (veth)
- Moving interfaces between namespaces
- Debugging multi-tenant environments

### Layer 4: Hardware and Drivers

**Network Interface Cards (NICs):**
- Driver configuration and tuning
- Multi-queue NICs and RSS (Receive Side Scaling)
- Interrupt coalescing and moderation
- Ring buffer sizing

**Hardware Offloading:**
- TSO (TCP Segmentation Offload)
- GSO (Generic Segmentation Offload)
- GRO (Generic Receive Offload)
- Checksum offloading

---

## Network Debugging: The Essential Skill

Network debugging is where Linux Network Engineers truly demonstrate their expertise. Issues can manifest at any layer, from application timeouts to kernel panics, requiring a systematic approach and deep tooling knowledge.

### Debugging Philosophy

**Top-Down Approach:**
1. Verify the application behavior
2. Check service configuration
3. Verify system networking configuration
4. Inspect kernel behavior
5. Examine hardware and drivers

**Bottom-Up Approach:**
1. Verify physical connectivity
2. Check link state and errors
3. Verify routing and forwarding
4. Check firewall rules
5. Test application connectivity

**Divide and Conquer:**
- Isolate the problem domain (client, network, server)
- Reproduce the issue consistently
- Eliminate variables systematically
- Document findings and test hypotheses

---

## Essential Debugging Tools

### Traditional Network Tools

#### ping - ICMP Echo Testing
```bash
# Basic connectivity test
ping -c 4 192.168.1.1

# Test with specific packet size (MTU discovery)
ping -M do -s 1472 example.com

# Specify source interface
ping -I eth0 192.168.1.1

# Flood ping (requires root, use carefully)
ping -f 192.168.1.1
```

**What it tells you:**
- Network reachability
- Round-trip time (latency)
- Packet loss
- MTU issues (with -M do flag)

**Limitations:**
- ICMP may be filtered/blocked
- Only tests basic IP connectivity
- Doesn't test TCP/UDP services

#### traceroute/mtr - Path Analysis
```bash
# Traditional traceroute
traceroute example.com

# MTR (My TraceRoute) - combines ping + traceroute
mtr --report --report-cycles 10 example.com

# Show IP addresses and AS numbers
mtr --aslookup example.com

# TCP-based traceroute (useful when ICMP is blocked)
traceroute -T -p 443 example.com
```

**What it tells you:**
- Network path and hops
- Where packet loss occurs
- Latency at each hop
- Routing changes over time (MTR)

**MTR advantages:**
- Real-time continuous monitoring
- Statistical aggregation
- Better for identifying intermittent issues

#### netstat - Network Statistics (Deprecated)
```bash
# Show all listening TCP ports
netstat -tlnp

# Show all connections with programs
netstat -tunap

# Display routing table
netstat -rn

# Show interface statistics
netstat -i
```

**Note:** netstat is deprecated. Use `ss` instead.

#### ss - Socket Statistics (Modern Replacement)
```bash
# Show all TCP sockets
ss -ta

# Show listening sockets
ss -tln

# Show process information
ss -tlnp

# Show socket memory usage
ss -tm

# Filter by state
ss -t state established

# Show detailed socket information
ss -tei

# Filter by port
ss -tan '( dport = :80 or sport = :80 )'

# Show TCP info (congestion control, RTT, etc.)
ss -ti
```

**Why ss is better:**
- Faster (directly queries kernel)
- More filtering options
- Better structured output
- Active development and support

#### ip - Network Configuration
```bash
# Show all interfaces
ip link show

# Show IP addresses
ip addr show

# Show routing table
ip route show

# Show routing table for specific destination
ip route get 8.8.8.8

# Show neighbor table (ARP)
ip neigh show

# Show network statistics
ip -s link

# Monitor route changes in real-time
ip monitor route
```

**Advanced usage:**
```bash
# Policy routing - show all routing tables
ip rule show

# Show specific routing table
ip route show table 100

# Network namespace operations
ip netns list
ip netns exec <namespace> <command>
```

#### tcpdump - Packet Capture
```bash
# Capture on specific interface
tcpdump -i eth0

# Capture with more verbosity and no name resolution
tcpdump -vvv -nn -i eth0

# Save to file
tcpdump -i eth0 -w capture.pcap

# Read from file
tcpdump -r capture.pcap

# Capture specific host
tcpdump -i eth0 host 192.168.1.100

# Capture specific port
tcpdump -i eth0 port 80

# Capture TCP flags (SYN packets)
tcpdump -i eth0 'tcp[tcpflags] & tcp-syn != 0'

# Show ASCII content
tcpdump -i eth0 -A

# Show hex and ASCII
tcpdump -i eth0 -XX

# Limit packet size captured (snapshot length)
tcpdump -i eth0 -s 96

# Advanced filters
tcpdump -i eth0 'tcp port 80 and (src host 192.168.1.100 or dst host 192.168.1.100)'

# Capture only SYN-ACK packets
tcpdump -i eth0 'tcp[13] = 18'
```

**BPF Filter Examples:**
```bash
# HTTP GET requests
tcpdump -i eth0 -A 'tcp port 80 and (tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x47455420)'

# DNS queries
tcpdump -i eth0 'udp port 53'

# Packets larger than 1000 bytes
tcpdump -i eth0 'greater 1000'

# Exclude SSH traffic
tcpdump -i eth0 'not port 22'
```

#### ethtool - Interface Diagnostics
```bash
# Show interface settings
ethtool eth0

# Show driver information
ethtool -i eth0

# Show statistics
ethtool -S eth0

# Show ring buffer sizes
ethtool -g eth0

# Set ring buffer sizes
ethtool -G eth0 rx 4096 tx 4096

# Show offload settings
ethtool -k eth0

# Disable offload feature
ethtool -K eth0 gso off

# Test interface (blink LED)
ethtool -p eth0 10
```

**Key metrics to watch:**
- RX/TX errors
- Dropped packets
- Collisions
- Link speed and duplex
- Ring buffer overruns

### Modern Kernel Tracing Tools

#### strace - System Call Tracing
```bash
# Trace a running process
strace -p <PID>

# Trace network system calls
strace -e trace=network -p <PID>

# Trace file operations
strace -e trace=file -p <PID>

# Show timing information
strace -T -p <PID>

# Count system calls
strace -c -p <PID>

# Trace a command from start
strace -o output.txt curl http://example.com

# Trace with timestamps
strace -tt -p <PID>

# Follow forks
strace -f -p <PID>
```

**Network-related syscalls to watch:**
- socket(), bind(), listen(), accept(), connect()
- send(), sendto(), sendmsg()
- recv(), recvfrom(), recvmsg()
- setsockopt(), getsockopt()
- poll(), epoll_wait(), select()

**Limitations:**
- Only traces system calls (user→kernel boundary)
- Cannot see kernel-internal operations
- High overhead (not for production profiling)

#### ftrace - Function Tracer
Location: `/sys/kernel/debug/tracing/`

```bash
# Enable function tracing
cd /sys/kernel/debug/tracing
echo function > current_tracer

# Set filter for network functions
echo 'tcp_*' > set_ftrace_filter

# Start tracing
echo 1 > tracing_on

# View trace
cat trace

# Stop tracing
echo 0 > tracing_on

# Clear trace buffer
echo > trace

# Trace specific function
echo tcp_sendmsg > set_ftrace_filter

# Trace function graph
echo function_graph > current_tracer
echo tcp_sendmsg > set_graph_function

# Enable events
echo 1 > events/net/enable
echo 1 > events/sock/enable

# Available events
ls events/
```

**Network-related trace events:**
- net:netif_receive_skb
- net:net_dev_xmit
- net:netif_rx
- sock:inet_sock_set_state
- tcp:tcp_probe

**Example: Tracing packet processing:**
```bash
cd /sys/kernel/debug/tracing
echo 1 > events/net/netif_receive_skb/enable
echo 1 > events/net/net_dev_xmit/enable
cat trace_pipe
```

#### perf - Performance Analysis
```bash
# Profile the system
perf top

# Record system activity
perf record -a -g sleep 10

# View recorded data
perf report

# Trace network events
perf trace -e 'net:*' -a

# Trace specific syscalls
perf trace -e connect,sendto,recvfrom

# Record with call graphs
perf record -a -g -e net:netif_receive_skb

# Show statistics for network functions
perf stat -e 'net:*' sleep 10

# Trace scheduler events (useful for network latency)
perf sched record -- sleep 10
perf sched latency
```

**Advanced perf usage:**
```bash
# Flame graphs for network functions
perf record -F 99 -a -g -- sleep 30
perf script | ./stackcollapse-perf.pl | ./flamegraph.pl > network.svg

# Trace off-CPU time (blocking)
perf record -e sched:sched_stat_sleep -e sched:sched_switch -a -g -- sleep 10

# Hardware counters
perf stat -e cycles,instructions,cache-misses -a sleep 10
```

### eBPF and Modern Tracing

#### What is eBPF?

Extended Berkeley Packet Filter (eBPF) is a revolutionary technology that allows running sandboxed programs in the Linux kernel without changing kernel source code or loading kernel modules. For network debugging, eBPF provides:

- **Zero overhead** when not in use
- **Production-safe** tracing (verified programs)
- **Deep visibility** into kernel operations
- **Real-time** analysis without packet capture
- **Programmable** filtering and aggregation

#### BCC (BPF Compiler Collection)

Pre-built tools for common tasks:

```bash
# Trace TCP connections
/usr/share/bcc/tools/tcpconnect

# Show TCP connection latency
/usr/share/bcc/tools/tcpconnlat

# Trace TCP retransmissions
/usr/share/bcc/tools/tcpretrans

# Show TCP session details
/usr/share/bcc/tools/tcplife

# Trace TCP drops
/usr/share/bcc/tools/tcpdrop

# Track TCP send congestion
/usr/share/bcc/tools/tcptop

# Trace DNS queries
/usr/share/bcc/tools/dnslatency

# Network throughput by socket
/usr/share/bcc/tools/sockstat

# Trace accept() latency
/usr/share/bcc/tools/tcpaccept
```

**Example: Debugging TCP retransmissions**
```bash
# Show all TCP retransmissions
tcpretrans

# Output example:
# TIME     PID    IP LADDR:LPORT          T> RADDR:RPORT          STATE
# 01:23:45 1234   4  10.0.0.5:45678       R> 93.184.216.34:80    ESTABLISHED

# This shows when retransmissions happen, which connections,
# and the connection state
```

**Example: Finding slow TCP connections**
```bash
# Show connections taking > 10ms to establish
tcpconnlat 10

# Output shows:
# - Process name and PID
# - Source and destination IP:port
# - Connection latency in milliseconds
```

#### bpftrace - High-Level eBPF Language

```bash
# Count syscalls by process
bpftrace -e 'tracepoint:syscalls:sys_enter_* { @[comm] = count(); }'

# Trace TCP connect with details
bpftrace -e '
  kprobe:tcp_connect {
    printf("TCP connect by PID %d\n", pid);
  }'

# Show distribution of TCP send sizes
bpftrace -e '
  kprobe:tcp_sendmsg {
    @send_bytes = hist(arg2);
  }'

# Trace packet drops
bpftrace -e '
  tracepoint:skb:kfree_skb {
    @[stack] = count();
  }'

# Measure TCP RTT
bpftrace -e '
  kprobe:tcp_rcv_established {
    $sk = (struct sock *)arg0;
    $tp = (struct tcp_sock *)$sk;
    @rtt_us = hist($tp->srtt_us >> 3);
  }'

# Network interface receive tracing
bpftrace -e '
  tracepoint:net:netif_receive_skb {
    @pkts[args->name] = count();
  }'
```

**Real-world bpftrace example - Finding network bottlenecks:**
```bash
# Trace time spent in network stack functions
bpftrace -e '
  kprobe:__netif_receive_skb_core,
  kprobe:ip_rcv,
  kprobe:tcp_v4_rcv {
    @start[tid] = nsecs;
  }

  kretprobe:__netif_receive_skb_core,
  kretprobe:ip_rcv,
  kretprobe:tcp_v4_rcv /@start[tid]/ {
    @time_ns[probe] = hist(nsecs - @start[tid]);
    delete(@start[tid]);
  }'
```

#### pwru - eBPF-Based Packet Tracer

Modern tool for tracing packets through the kernel:

```bash
# Trace packets to specific IP
pwru --filter-dst-ip 8.8.8.8

# Trace packets on specific port
pwru --filter-dst-port 443

# Show full packet details
pwru --output-tuple --output-skb

# Filter by network namespace
pwru --filter-netns 12345

# Trace dropped packets
pwru --filter-track-skb-by-stackid
```

**What makes pwru special:**
- Traces packet journey through kernel functions
- Shows exactly where packets are dropped
- Lower overhead than tcpdump
- Can filter at kernel level before copying to userspace

### Firewall and Connection Tracking Debugging

#### iptables/nftables Tracing

**iptables TRACE target:**
```bash
# Enable tracing for specific packets
iptables -t raw -A PREROUTING -p tcp --dport 80 -j TRACE

# View trace in kernel log
xtables-monitor --trace
# or
tail -f /var/log/kern.log | grep TRACE
```

**nftables tracing:**
```bash
# Add tracing rule
nft add rule ip filter input tcp dport 80 meta nftrace set 1

# Monitor traces
nft monitor trace

# More specific tracing with rate limiting
nft add rule ip filter input ip saddr 192.168.1.100 tcp dport 80 limit rate 1/minute meta nftrace set 1
```

**What tracing shows:**
- Each rule the packet matches
- Each chain the packet traverses
- Verdict for each rule (accept, drop, etc.)
- Tables and chains in order

#### conntrack - Connection Tracking

```bash
# Show all connections
conntrack -L

# Show connection count
conntrack -C

# Watch connections in real-time
conntrack -E

# Show specific protocol
conntrack -L -p tcp

# Show connections to specific IP
conntrack -L -d 192.168.1.100

# Show connection statistics
conntrack -S

# Delete specific connection
conntrack -D -p tcp --orig-port-dst 80 -d 192.168.1.100
```

**Debug connection tracking issues:**
```bash
# Check current table usage
cat /proc/sys/net/netfilter/nf_conntrack_count
cat /proc/sys/net/netfilter/nf_conntrack_max

# If table full, increase size
echo 262144 > /proc/sys/net/netfilter/nf_conntrack_max

# Or tune timeout values
cat /proc/sys/net/netfilter/nf_conntrack_tcp_timeout_established
```

**Real scenario - NAT debugging:**
```bash
# Watch NAT translations in real-time
conntrack -E -o timestamp -p tcp -n

# Shows:
# - Original source:port → destination:port
# - Translated source:port → destination:port
# - Connection state changes
# - Timestamps
```

### Advanced Network Debugging Workflows

#### Scenario 1: Application Can't Connect

**Symptom:** Application times out connecting to server

**Debugging steps:**
```bash
# 1. Verify DNS resolution
dig example.com
nslookup example.com

# 2. Test basic connectivity
ping example.com

# 3. Test specific port
telnet example.com 443
# or
nc -zv example.com 443

# 4. Check if application is trying to connect
strace -e trace=network -p <PID>
# Look for: connect() calls and their return values

# 5. Check routing
ip route get <IP>

# 6. Check firewall rules
iptables -L -n -v
nft list ruleset

# 7. Capture packets
tcpdump -i any -nn host <IP> -w debug.pcap
# Then analyze in Wireshark looking for:
# - SYN sent?
# - SYN-ACK received?
# - RST packets?
# - Retransmissions?

# 8. Check connection tracking
conntrack -L | grep <IP>
```

#### Scenario 2: Slow Network Performance

**Symptom:** High latency or low throughput

**Debugging steps:**
```bash
# 1. Check interface errors
ip -s link show eth0
ethtool -S eth0 | grep -i error

# 2. Check ring buffer drops
ethtool -S eth0 | grep -i drop

# 3. Check TCP stats
ss -tiepo
# Look for:
# - Retransmissions (retrans)
# - RTT (rtt)
# - Congestion window (cwnd)

# 4. Check for packet loss
mtr --report example.com

# 5. Profile TCP connections
/usr/share/bcc/tools/tcplife

# 6. Check for retransmissions
/usr/share/bcc/tools/tcpretrans

# 7. Check congestion control
sysctl net.ipv4.tcp_congestion_control
ss -ti | grep -i cubic

# 8. Trace latency
/usr/share/bcc/tools/tcpconnlat

# 9. Check system load
perf top
# Look for:
# - High softirq CPU usage
# - Network driver functions
# - Kernel network stack functions

# 10. Check buffer tuning
sysctl net.core.rmem_max
sysctl net.core.wmem_max
sysctl net.ipv4.tcp_rmem
sysctl net.ipv4.tcp_wmem
```

#### Scenario 3: Packet Loss/Drops

**Symptom:** Packets disappearing

**Debugging steps:**
```bash
# 1. Check interface drops
netstat -i
ip -s link show

# 2. Check which layer drops occur
ethtool -S eth0 | grep drop

# 3. Trace kernel drops with eBPF
/usr/share/bcc/tools/tcpdrop

# 4. Check if firewall dropping
iptables -L -v -n | grep DROP
nft list ruleset

# 5. Enable firewall logging
iptables -I INPUT -j LOG --log-prefix "IPT-INPUT-DROP: "

# 6. Trace with ftrace
cd /sys/kernel/debug/tracing
echo 1 > events/skb/kfree_skb/enable
cat trace_pipe

# 7. Use bpftrace to find drop location
bpftrace -e 'tracepoint:skb:kfree_skb { @[kstack] = count(); }'

# 8. Check conntrack drops
dmesg | grep conntrack
conntrack -S

# 9. Check for buffer overruns
sar -n DEV 1 10
```

#### Scenario 4: High Connection Count Issues

**Symptom:** Service degradation under load

**Debugging steps:**
```bash
# 1. Count current connections
ss -s

# 2. Show connections by state
ss -tan | awk '{print $1}' | sort | uniq -c

# 3. Check for TIME_WAIT buildup
ss -tan | grep TIME_WAIT | wc -l

# 4. Check socket limits
ulimit -n
cat /proc/sys/fs/file-max

# 5. Check per-process limits
cat /proc/<PID>/limits | grep files

# 6. Check connection tracking table
cat /proc/sys/net/netfilter/nf_conntrack_count
cat /proc/sys/net/netfilter/nf_conntrack_max

# 7. Check ephemeral port range
cat /proc/sys/net/ipv4/ip_local_port_range

# 8. Monitor connection lifecycle
/usr/share/bcc/tools/tcplife

# 9. Check for socket leaks
lsof -p <PID> | grep socket

# 10. Profile socket allocation
perf record -e 'syscalls:sys_enter_socket' -a -g
```

#### Scenario 5: Intermittent Connectivity

**Symptom:** Connections work sometimes, fail other times

**Debugging steps:**
```bash
# 1. Continuous monitoring with MTR
mtr --report-cycles 1000 example.com > mtr-report.txt

# 2. Monitor routing changes
ip monitor route

# 3. Monitor neighbor (ARP) changes
ip monitor neigh

# 4. Check for interface flapping
journalctl -f -u NetworkManager
# or
dmesg -w | grep eth0

# 5. Monitor link state
watch -n 1 'ethtool eth0 | grep Link'

# 6. Continuous packet capture
tcpdump -i eth0 -G 60 -w capture-%Y%m%d-%H%M%S.pcap

# 7. Monitor connection tracking events
conntrack -E -o timestamp

# 8. Check for DNS issues
while true; do dig example.com | grep "Query time"; sleep 1; done

# 9. Monitor with eBPF
/usr/share/bcc/tools/tcpdrop &
/usr/share/bcc/tools/tcpretrans &

# 10. Log all connection attempts
bpftrace -e '
  kprobe:tcp_connect {
    printf("%s: TCP connect attempt\n", strftime("%H:%M:%S", nsecs));
  }
  kretprobe:tcp_connect {
    printf("%s: TCP connect returned %d\n", strftime("%H:%M:%S", nsecs), retval);
  }'
```

---

## Advanced Topics

### Network Namespaces Debugging

```bash
# List all namespaces
ip netns list

# Execute command in namespace
ip netns exec <namespace> bash

# Debug namespace connectivity
ip netns exec ns1 ping 192.168.1.2

# Show namespace routing
ip netns exec ns1 ip route

# Show namespace interfaces
ip netns exec ns1 ip link

# Capture packets in namespace
ip netns exec ns1 tcpdump -i eth0

# Check processes in namespace
ip netns identify <PID>
```

**Container networking debugging:**
```bash
# Find container's network namespace
docker inspect <container> | grep NetworkMode
docker inspect <container> | grep SandboxKey

# Enter container's network namespace
nsenter --net=/var/run/docker/netns/<ns> bash

# Or use docker exec
docker exec <container> ip addr
docker exec <container> ss -tan
```

### XDP (eXpress Data Path) Debugging

XDP programs run at the earliest point in the network stack (NIC driver).

```bash
# Check if XDP program attached
ip link show eth0

# Load XDP program
ip link set dev eth0 xdp obj program.o

# Unload XDP program
ip link set dev eth0 xdp off

# Debug XDP programs
bpftool prog list
bpftool map list
bpftool prog show id <ID>
```

### Traffic Control (tc) Debugging

```bash
# Show qdisc configuration
tc qdisc show dev eth0

# Show filter rules
tc filter show dev eth0

# Show class configuration
tc class show dev eth0

# Show statistics
tc -s qdisc show dev eth0
tc -s class show dev eth0

# Monitor in real-time
watch -n 1 'tc -s qdisc show dev eth0'
```

---

## Performance Tuning Based on Debugging Findings

### When you find high retransmissions:
```bash
# Increase TCP buffer sizes
sysctl -w net.core.rmem_max=134217728
sysctl -w net.core.wmem_max=134217728
sysctl -w net.ipv4.tcp_rmem="4096 87380 67108864"
sysctl -w net.ipv4.tcp_wmem="4096 65536 67108864"

# Enable window scaling
sysctl -w net.ipv4.tcp_window_scaling=1

# Adjust congestion control
sysctl -w net.ipv4.tcp_congestion_control=bbr
```

### When you find packet drops at NIC:
```bash
# Increase ring buffer size
ethtool -G eth0 rx 4096 tx 4096

# Enable multi-queue
ethtool -L eth0 combined 4

# Adjust interrupt coalescing
ethtool -C eth0 rx-usecs 50
```

### When you find connection tracking table full:
```bash
# Increase table size
sysctl -w net.netfilter.nf_conntrack_max=1048576

# Reduce timeouts
sysctl -w net.netfilter.nf_conntrack_tcp_timeout_established=600
sysctl -w net.netfilter.nf_conntrack_tcp_timeout_time_wait=30
```

### When you find TIME_WAIT issues:
```bash
# Enable reuse (safe for clients)
sysctl -w net.ipv4.tcp_tw_reuse=1

# Reduce timeout
sysctl -w net.ipv4.tcp_fin_timeout=30
```

---

## Essential Skills Summary

A proficient Linux Network Engineer should be able to:

1. **Diagnose** network issues at any layer using appropriate tools
2. **Trace** packets through the entire stack from application to wire
3. **Profile** network performance and identify bottlenecks
4. **Understand** kernel networking internals and configuration
5. **Optimize** system parameters based on workload characteristics
6. **Automate** network configuration and monitoring
7. **Secure** network services and implement defense in depth
8. **Design** scalable network architectures on Linux
9. **Debug** complex multi-tenant and containerized environments
10. **Communicate** findings clearly with evidence and data

The depth of Linux network engineering is vast—from understanding how a single packet traverses hardware rings, through kernel networking layers, netfilter hooks, routing decisions, socket buffers, to finally reaching an application. Mastery requires both theoretical knowledge and extensive hands-on debugging experience.

---

## Quick Reference: Tool Selection Guide

| Problem | First Tool | Detailed Analysis |
|---------|-----------|-------------------|
| Can't connect | ping, telnet | tcpdump, strace |
| Slow performance | mtr, ss | perf, tcplife |
| Packet drops | ip -s link, ethtool -S | tcpdrop, ftrace |
| High retransmissions | ss -ti | tcpretrans, tcpdump |
| Connection issues | ss -tan | conntrack, strace |
| DNS problems | dig, nslookup | tcpdump port 53 |
| Routing issues | ip route get | ip monitor, mtr |
| Firewall blocking | iptables -L -v | nft monitor trace |
| Interface problems | ethtool, ip link | dmesg, driver logs |
| Kernel issues | dmesg | ftrace, bpftrace |

**General debugging workflow:**
1. Define the problem clearly
2. Reproduce consistently
3. Start with simple tools (ping, ss, ip)
4. Narrow down the layer
5. Use targeted tools for that layer
6. Collect evidence (packet captures, traces)
7. Form hypothesis
8. Test and validate
9. Document findings

---

## Further Resources

**Books:**
- "TCP/IP Illustrated" by W. Richard Stevens
- "Linux Kernel Networking: Implementation and Theory" by Rami Rosen
- "BPF Performance Tools" by Brendan Gregg

**Online:**
- Brendan Gregg's Blog: brendangregg.com
- Linux Network Developers: netdevconf.org
- Kernel networking documentation: www.kernel.org/doc/Documentation/networking/

**Tools Documentation:**
- man pages: man 8 ip, man 8 ss, man 8 tcpdump
- bcc tools: github.com/iovisor/bcc
- bpftrace: github.com/iovisor/bpftrace

**Practice:**
- Set up network namespaces and experiment
- Create intentional misconfigurations and debug them
- Analyze packet captures from real issues
- Write your own eBPF programs for custom tracing
- Contribute to open source networking projects

# Hands-On Network Tracing and Debugging Exercises

This guide provides practical exercises and real-world scenarios to master Linux network tracing and debugging tools. Each section includes simple starter exercises progressing to real-world production scenarios.

---

## Table of Contents

1. [tcpdump: Packet Capture Exercises](#tcpdump-packet-capture-exercises)
2. [strace: System Call Tracing Exercises](#strace-system-call-tracing-exercises)
3. [ftrace: Kernel Function Tracing Exercises](#ftrace-kernel-function-tracing-exercises)
4. [perf: Performance Analysis Exercises](#perf-performance-analysis-exercises)
5. [eBPF/BCC Tools Exercises](#ebpfbcc-tools-exercises)
6. [bpftrace: One-Liner Exercises](#bpftrace-one-liner-exercises)
7. [Real-World Production Scenarios](#real-world-production-scenarios)
8. [Debugging Workflow Challenges](#debugging-workflow-challenges)

---

## tcpdump: Packet Capture Exercises

### Exercise 1: Basic Packet Capture

**Objective:** Learn to capture and analyze basic network traffic.

```bash
# 1. Capture 10 packets on any interface
tcpdump -i any -c 10

# 2. Capture with more detail (no DNS resolution, verbose)
tcpdump -i any -nn -vv -c 10

# 3. Capture only HTTP traffic
tcpdump -i any -nn port 80 -c 20

# 4. Save to file for later analysis
tcpdump -i any -nn port 80 -w http_traffic.pcap -c 50
```

**Questions to answer:**
- What protocols did you see?
- Which IP addresses were communicating?
- Can you identify the source and destination ports?

**Expected output example:**
```
15:30:45.123456 IP 192.168.1.100.45678 > 93.184.216.34.80: Flags [S], seq 123456789
15:30:45.145678 IP 93.184.216.34.80 > 192.168.1.100.45678: Flags [S.], seq 987654321
```

### Exercise 2: HTTP Traffic Analysis

**Objective:** Extract useful information from HTTP traffic.

```bash
# 1. Capture HTTP traffic with ASCII output
tcpdump -i any -nn -A 'tcp port 80 and (tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x47455420)'

# This captures HTTP GET requests. Let's break it down:
# - 'tcp port 80' = only port 80
# - 'tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x47455420' = matches "GET " in ASCII

# 2. Capture HTTP POST requests
tcpdump -i any -nn -A 'tcp port 80 and (tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x504f5354)'

# 3. Capture and extract HTTP User-Agent
tcpdump -i any -nn -A 'tcp port 80' | grep -i "user-agent"

# 4. Generate some HTTP traffic to capture
curl http://example.com &
tcpdump -i any -nn -A 'tcp port 80' -c 20
```

**What to look for:**
- HTTP request methods (GET, POST)
- User-Agent strings
- Host headers
- Request/response timing

### Exercise 3: Connection Troubleshooting

**Objective:** Analyze TCP handshake and identify connection issues.

```bash
# 1. Capture only SYN packets (connection attempts)
tcpdump -i any 'tcp[tcpflags] & tcp-syn != 0' -nn

# 2. Capture full three-way handshake
tcpdump -i any 'tcp[tcpflags] & (tcp-syn|tcp-ack) != 0' -nn -c 30

# 3. Capture RST packets (connection resets)
tcpdump -i any 'tcp[tcpflags] & tcp-rst != 0' -nn

# 4. Full connection lifecycle capture
tcpdump -i any -nn 'host 8.8.8.8' -w connection_test.pcap &
TCPDUMP_PID=$!
ping -c 3 8.8.8.8
sleep 5
kill $TCPDUMP_PID
tcpdump -r connection_test.pcap -nn
```

**Analysis questions:**
- Did you see SYN → SYN-ACK → ACK sequence?
- Were there any retransmissions?
- Did connections close cleanly (FIN) or abruptly (RST)?

### Exercise 4: Advanced Filtering

**Objective:** Master complex capture filters.

```bash
# 1. Capture traffic from specific subnet
tcpdump -i any 'net 192.168.1.0/24' -nn

# 2. Capture DNS queries only (queries, not responses)
tcpdump -i any 'udp port 53 and udp[10] & 0x80 = 0' -nn

# 3. Capture packets larger than 1000 bytes
tcpdump -i any 'greater 1000' -nn

# 4. Exclude SSH traffic from capture
tcpdump -i any 'not port 22' -nn

# 5. Capture traffic between two specific hosts
tcpdump -i any 'host 192.168.1.10 and host 192.168.1.20' -nn

# 6. Capture all traffic except from your own machine
MY_IP=$(ip -4 addr show eth0 | grep -oP '(?<=inet\s)\d+(\.\d+){3}')
tcpdump -i any "not host $MY_IP" -nn
```

### Real-World Scenario 1: Debugging Slow Website

**Problem:** Users report website is loading slowly.

**Solution steps:**
```bash
# 1. Capture traffic to the web server
tcpdump -i any -nn -s0 'host webserver.example.com and port 80' -w slow_web.pcap

# 2. While capturing, test the website
time curl -v http://webserver.example.com

# 3. Stop capture and analyze timing
tcpdump -r slow_web.pcap -nn -tttt

# 4. Look for:
# - Time between SYN and SYN-ACK (network latency)
# - Time between request and first response byte (server processing)
# - Multiple retransmissions (packet loss)
# - Large gaps between packets (bandwidth issue)

# 5. Calculate connection establishment time
tcpdump -r slow_web.pcap -nn -tttt | grep 'Flags \[S\]' | head -2
# Subtract the timestamps to get handshake time
```

**What you're measuring:**
- Network RTT (Round Trip Time)
- Server response time
- Data transfer rate
- Packet loss indicators

### Real-World Scenario 2: Investigating Failed Connections

**Problem:** Application randomly fails to connect to database.

```bash
# 1. Start continuous monitoring
tcpdump -i any -nn 'host db.example.com and port 3306' -w db_connections.pcap &

# 2. Run your application and wait for failure
# (let it run for several minutes)

# 3. Stop capture and analyze
tcpdump -r db_connections.pcap -nn | grep -E '(Flags \[S\]|Flags \[R\])'

# Expected patterns:
# SUCCESS: SYN → SYN-ACK → ACK
# REFUSED: SYN → RST-ACK (port closed or firewalled)
# TIMEOUT: SYN → SYN (retransmit) → SYN (retransmit) → no response
# REJECTED: SYN → RST (connection rejected by application)

# 4. Count connection attempts vs successes
echo "SYN packets (attempts):"
tcpdump -r db_connections.pcap -nn 'tcp[tcpflags] & tcp-syn != 0 and tcp[tcpflags] & tcp-ack = 0' | wc -l
echo "SYN-ACK packets (successful):"
tcpdump -r db_connections.pcap -nn 'tcp[tcpflags] & tcp-syn != 0 and tcp[tcpflags] & tcp-ack != 0' | wc -l
```

---

## strace: System Call Tracing Exercises

### Exercise 1: Basic Process Tracing

**Objective:** Understand what system calls a program makes.

```bash
# 1. Trace a simple command
strace ls -la

# 2. Trace with timing information
strace -T ls -la

# 3. Count system calls
strace -c ls -la

# 4. Trace only file operations
strace -e trace=open,read,write,close ls -la

# 5. Trace a running process (find process ID first)
# In one terminal:
sleep 100 &
echo $!  # Note the PID

# In another terminal:
strace -p <PID>
```

**What to observe:**
- Which files are being opened?
- Which system calls are called most frequently?
- How long do different operations take?

### Exercise 2: Network System Calls

**Objective:** Trace network-related system calls.

```bash
# 1. Trace network operations of curl
strace -e trace=network curl http://example.com

# 2. More detailed with timing
strace -T -e trace=network curl http://example.com

# 3. Save output to file for analysis
strace -e trace=network -o curl_trace.txt curl http://example.com
cat curl_trace.txt

# 4. Trace socket operations in detail
strace -e trace=socket,connect,sendto,recvfrom,setsockopt,getsockopt curl http://example.com
```

**Expected output:**
```
socket(AF_INET, SOCK_STREAM, IPPROTO_TCP) = 3
connect(3, {sa_family=AF_INET, sin_port=htons(80), sin_addr=inet_addr("93.184.216.34")}, 16) = 0
sendto(3, "GET / HTTP/1.1\r\nHost: example.com\r\n...", 78, MSG_NOSIGNAL, NULL, 0) = 78
recvfrom(3, "HTTP/1.1 200 OK\r\n...", 16384, 0, NULL, NULL) = 1256
```

### Exercise 3: Debugging Connection Failures

**Objective:** Find why an application can't connect.

```bash
# 1. Trace and look for errors
strace -e trace=network curl http://192.168.1.250:8080 2>&1 | grep -E '(ETIMEDOUT|ECONNREFUSED|EHOSTUNREACH)'

# Common error codes:
# ECONNREFUSED = Connection refused (port not listening)
# ETIMEDOUT = Connection timed out (no route or firewall)
# EHOSTUNREACH = No route to host
# ENETUNREACH = Network unreachable

# 2. Trace DNS resolution
strace -e trace=network,open,read curl http://invalid-domain-test.com 2>&1 | grep -A5 -B5 'getaddrinfo\|resolve'

# 3. Full network trace with error codes
strace -v -e trace=network nc -v google.com 443
```

### Exercise 4: Following Child Processes

**Objective:** Trace programs that spawn child processes.

```bash
# 1. Trace bash and all its children
strace -f bash -c 'echo "Testing" | grep Test'

# 2. Trace SSH connection (includes child processes)
strace -f -e trace=network ssh user@hostname 'echo test'

# 3. Count syscalls across all children
strace -f -c bash -c 'ls | wc -l'
```

### Real-World Scenario 3: Application Can't Read Config File

**Problem:** Application fails to start with "config file not found" error.

```bash
# 1. Trace what files the application tries to open
strace -e trace=open,openat,stat ./myapp 2>&1 | grep -i config

# 2. More detailed - see exact paths and error codes
strace -e trace=open,openat,stat -v ./myapp 2>&1 | tee app_trace.log

# 3. Look for patterns in the output:
# openat(AT_FDCWD, "/etc/myapp/config.yml", O_RDONLY) = -1 ENOENT (No such file or directory)
# openat(AT_FDCWD, "/home/user/.myapp/config.yml", O_RDONLY) = -1 ENOENT (No such file or directory)
# openat(AT_FDCWD, "./config.yml", O_RDONLY) = 3

# 4. Find successful and failed opens
echo "=== Files successfully opened ==="
grep 'openat.*= [0-9]' app_trace.log

echo "=== Files that failed to open ==="
grep 'openat.*= -1' app_trace.log
```

### Real-World Scenario 4: Why is This Process Hanging?

**Problem:** Process appears to hang, not responding.

```bash
# 1. Find the PID
ps aux | grep myapp

# 2. Attach strace to see what it's waiting on
strace -p <PID>

# Common hang scenarios you'll see:

# Waiting on network:
# recvfrom(3, ...)  # Stuck here = waiting for data

# Waiting on file lock:
# flock(3, LOCK_EX) # Stuck here = another process has the lock

# Waiting on user input:
# read(0, ...)      # Stuck here = waiting for stdin

# Waiting on mutex/semaphore:
# futex(...)        # Stuck here = waiting for another thread

# 3. Add timing to see how long it waits
strace -T -p <PID>

# 4. For detailed debugging, trace all threads
strace -f -p <PID>
```

---

## ftrace: Kernel Function Tracing Exercises

### Exercise 1: Getting Started with ftrace

**Objective:** Learn basic ftrace operations.

```bash
# 1. Navigate to ftrace directory
cd /sys/kernel/debug/tracing

# 2. Check available tracers
cat available_tracers

# 3. Set function tracer
echo function > current_tracer

# 4. Start tracing
echo 1 > tracing_on

# 5. Generate some activity
ping -c 3 google.com

# 6. Stop tracing
echo 0 > tracing_on

# 7. View trace
head -50 trace

# 8. Clear trace buffer
echo > trace

# 9. Disable tracing
echo nop > current_tracer
```

### Exercise 2: Tracing Specific Functions

**Objective:** Trace network-related kernel functions.

```bash
cd /sys/kernel/debug/tracing

# 1. List available functions to trace
cat available_filter_functions | grep tcp | head -20

# 2. Set filter to trace only TCP functions
echo 'tcp_*' > set_ftrace_filter

# 3. Enable function tracer
echo function > current_tracer
echo 1 > tracing_on

# 4. Generate TCP traffic
curl http://example.com > /dev/null

# 5. Stop and view
echo 0 > tracing_on
head -100 trace

# 6. Clear filter
echo > set_ftrace_filter
echo nop > current_tracer
```

### Exercise 3: Function Graph Tracing

**Objective:** See function call hierarchies and timing.

```bash
cd /sys/kernel/debug/tracing

# 1. Enable function graph tracer
echo function_graph > current_tracer

# 2. Set specific function to graph
echo tcp_sendmsg > set_graph_function

# 3. Start tracing
echo 1 > tracing_on

# 4. Generate traffic
echo "test" | nc -w 1 example.com 80

# 5. Stop and view
echo 0 > tracing_on
cat trace

# Output shows call hierarchy with timing:
# 3)               |  tcp_sendmsg() {
# 3)   0.123 us    |    lock_sock();
# 3)   0.456 us    |    tcp_send_mss();
# 3)   2.345 us    |  }
```

### Exercise 4: Event Tracing

**Objective:** Trace kernel events related to networking.

```bash
cd /sys/kernel/debug/tracing

# 1. See available events
ls events/
ls events/net/
ls events/sock/

# 2. Enable network receive events
echo 1 > events/net/netif_receive_skb/enable

# 3. Enable socket state change events
echo 1 > events/sock/inet_sock_set_state/enable

# 4. Start tracing
echo 1 > tracing_on

# 5. Generate traffic
ping -c 5 8.8.8.8

# 6. View trace
cat trace_pipe  # Real-time view (Ctrl+C to stop)
# or
echo 0 > tracing_on
cat trace

# 7. Disable events
echo 0 > events/net/netif_receive_skb/enable
echo 0 > events/sock/inet_sock_set_state/enable
```

### Real-World Scenario 5: Packet Drop Investigation

**Problem:** Packets are being dropped somewhere in the kernel.

```bash
cd /sys/kernel/debug/tracing

# 1. Enable packet drop event
echo 1 > events/skb/kfree_skb/enable

# 2. Start tracing with stack traces
echo 1 > options/stacktrace
echo 1 > tracing_on

# 3. Reproduce the issue
# (run your application or test that causes drops)

# 4. View results
echo 0 > tracing_on
cat trace

# 5. Analyze the stack trace to see WHERE packets were dropped
# Look for patterns like:
# - netfilter functions (firewall drop)
# - tcp_validate_incoming (invalid packets)
# - __udp4_lib_rcv (UDP checksum errors)

# 6. Cleanup
echo 0 > events/skb/kfree_skb/enable
echo 0 > options/stacktrace
```

---

## perf: Performance Analysis Exercises

### Exercise 1: System-Wide Profiling

**Objective:** Find performance bottlenecks.

```bash
# 1. Profile the entire system for 10 seconds
perf top

# 2. Record system activity
perf record -a -g sleep 10

# 3. View the results
perf report

# 4. Record with higher sampling rate
perf record -F 999 -a -g sleep 10
perf report
```

**What to look for:**
- Functions with high percentage (hotspots)
- Kernel vs userspace time
- Call graphs showing function relationships

### Exercise 2: Network Event Tracing

**Objective:** Trace network-specific events.

```bash
# 1. List network-related events
perf list | grep net

# 2. Trace all network events
perf trace -e 'net:*' -a

# 3. Trace specific network events
perf trace -e net:netif_receive_skb,net:net_dev_xmit

# 4. Record network events with stack traces
perf record -e net:netif_receive_skb -a -g sleep 5
perf report

# 5. Count network events
perf stat -e 'net:*' sleep 10
```

### Exercise 3: Application Performance Analysis

**Objective:** Profile a specific application.

```bash
# 1. Profile curl downloading a file
perf record -g curl http://example.com

# 2. View report
perf report

# 3. Profile with network events
perf record -e cycles,net:netif_receive_skb -g curl http://example.com

# 4. Analyze where time is spent
perf report --sort=symbol,dso

# 5. Generate flamegraph (if you have flamegraph tools)
perf script | ./stackcollapse-perf.pl | ./flamegraph.pl > curl.svg
```

### Exercise 4: TCP Analysis

**Objective:** Analyze TCP performance and behavior.

```bash
# 1. Trace TCP connection events
perf trace -e 'tcp:*' curl http://example.com

# 2. Record TCP events with details
perf record -e 'tcp:*' -a -g -- curl http://example.com
perf script

# 3. Count TCP events
perf stat -e 'tcp:tcp_probe,tcp:tcp_retransmit_skb' -a -- curl http://slow-server.com
```

### Real-World Scenario 6: High CPU Usage in Network App

**Problem:** Web server consuming 100% CPU.

```bash
# 1. First, identify the process
ps aux | grep nginx

# 2. Profile it for 30 seconds
perf record -p <PID> -g sleep 30

# 3. View the report
perf report

# 4. Look for:
# - Functions taking most CPU time
# - Is it in userspace (nginx code) or kernel (syscalls)?
# - Call graphs showing the path to hot functions

# 5. If it's network-related, check network events
perf record -e cycles,net:*,tcp:* -p <PID> -g sleep 30
perf report

# 6. Generate annotated source/assembly
perf annotate <function_name>

# This shows which exact instructions are slow
```

---

## eBPF/BCC Tools Exercises

### Exercise 1: Using Pre-Built BCC Tools

**Objective:** Master the essential BCC tools.

```bash
# First, install BCC tools (Ubuntu/Debian)
# sudo apt-get install bpfcc-tools linux-headers-$(uname -r)

# 1. Trace new TCP connections
/usr/share/bcc/tools/tcpconnect

# Expected output:
# PID    COMM         IP SADDR            DADDR            DPORT
# 1234   curl         4  192.168.1.100    93.184.216.34    80

# 2. Trace TCP connection latency (time to establish connection)
/usr/share/bcc/tools/tcpconnlat

# Expected output:
# PID    COMM         IP SADDR            DADDR            DPORT LAT(ms)
# 1234   curl         4  192.168.1.100    93.184.216.34    80    15.23

# 3. Trace TCP retransmissions (indicates packet loss)
/usr/share/bcc/tools/tcpretrans

# 4. Show TCP connection lifespan
/usr/share/bcc/tools/tcplife

# 5. Trace TCP drops
/usr/share/bcc/tools/tcpdrop

# 6. Monitor DNS latency
/usr/share/bcc/tools/dnslatency
```

### Exercise 2: Investigating Connection Issues

**Objective:** Use BCC tools to debug connectivity.

```bash
# Scenario: Some connections to your service are slow

# 1. In one terminal, monitor connection latency
/usr/share/bcc/tools/tcpconnlat 10

# This shows only connections taking > 10ms

# 2. In another terminal, monitor for retransmissions
/usr/share/bcc/tools/tcpretrans

# 3. Check for drops
/usr/share/bcc/tools/tcpdrop

# 4. Monitor overall connection lifecycle
/usr/share/bcc/tools/tcplife -w

# -w flag includes wider output with more details

# 5. Generate test traffic and observe
for i in {1..10}; do
    curl -s http://your-server.com > /dev/null &
done
```

### Exercise 3: Advanced Filtering

**Objective:** Filter BCC tool output for specific analysis.

```bash
# 1. Trace connections to specific port
/usr/share/bcc/tools/tcpconnect -P 443

# 2. Trace connections from specific PID
/usr/share/bcc/tools/tcpconnect -p 1234

# 3. Show connection latency with minimum threshold
/usr/share/bcc/tools/tcpconnlat 50  # Only show >50ms

# 4. Monitor retransmissions with loss rate calculation
/usr/share/bcc/tools/tcpretrans -l  # -l shows loss percentage

# 5. Track TCP by process
/usr/share/bcc/tools/tcptop

# This shows TCP send/receive by process, like top for network
```

### Real-World Scenario 7: Intermittent Connection Drops

**Problem:** Users report occasional connection failures.

```bash
# 1. In terminal 1: Monitor for TCP drops
/usr/share/bcc/tools/tcpdrop | tee tcp_drops.log

# 2. In terminal 2: Monitor retransmissions
/usr/share/bcc/tools/tcpretrans | tee tcp_retrans.log

# 3. In terminal 3: Monitor connection latency spikes
/usr/share/bcc/tools/tcpconnlat 100 | tee high_latency.log

# 4. Let these run for several hours or during peak times

# 5. Analyze the logs
echo "=== Total drops ==="
wc -l tcp_drops.log

echo "=== Retransmissions by IP ==="
awk '{print $5}' tcp_retrans.log | sort | uniq -c | sort -rn

echo "=== High latency connections ==="
awk '{print $7, $8}' high_latency.log | sort -rn | head -20

# 6. Correlate timestamps to find patterns
# Are drops happening at same time as retransmissions?
# Are they to specific IP addresses?
```

---

## bpftrace: One-Liner Exercises

### Exercise 1: Basic One-Liners

**Objective:** Learn bpftrace syntax with simple examples.

```bash
# 1. Count system calls by program name
bpftrace -e 'tracepoint:syscalls:sys_enter_* { @[comm] = count(); }'

# Run for a few seconds, then Ctrl+C to see results

# 2. Count system calls by syscall name
bpftrace -e 'tracepoint:syscalls:sys_enter_* { @[probe] = count(); }'

# 3. Trace process execution
bpftrace -e 'tracepoint:syscalls:sys_enter_execve { printf("%s -> %s\n", comm, str(args->filename)); }'

# 4. Count file opens by filename
bpftrace -e 'tracepoint:syscalls:sys_enter_openat { @[str(args->filename)] = count(); }'
```

### Exercise 2: Network One-Liners

**Objective:** Quick network debugging with one-liners.

```bash
# 1. Count packets received per interface
bpftrace -e 'tracepoint:net:netif_receive_skb { @pkts[args->name] = count(); }'

# 2. Histogram of TCP send sizes
bpftrace -e 'kprobe:tcp_sendmsg { @send_bytes = hist(arg2); }'

# 3. Trace TCP connections
bpftrace -e 'kprobe:tcp_connect { printf("TCP connect by PID %d (%s)\n", pid, comm); }'

# 4. Count network system calls by process
bpftrace -e 'tracepoint:syscalls:sys_enter_send*,
             tracepoint:syscalls:sys_enter_recv* { @[comm] = count(); }'

# 5. Show TCP state changes
bpftrace -e 'tracepoint:sock:inet_sock_set_state {
    printf("%s: %d -> %d\n", comm, args->oldstate, args->newstate);
}'
```

### Exercise 3: Performance Analysis

**Objective:** Measure latency and performance.

```bash
# 1. Measure read() latency distribution
bpftrace -e 'tracepoint:syscalls:sys_enter_read { @ts[tid] = nsecs; }
             tracepoint:syscalls:sys_exit_read /@ts[tid]/ {
                 @latency_us = hist((nsecs - @ts[tid]) / 1000);
                 delete(@ts[tid]);
             }'

# 2. Measure TCP connection establishment time
bpftrace -e 'kprobe:tcp_connect { @ts[tid] = nsecs; }
             kretprobe:tcp_connect /@ts[tid]/ {
                 @connect_ms = hist((nsecs - @ts[tid]) / 1000000);
                 delete(@ts[tid]);
             }'

# 3. Show slowest system calls
bpftrace -e 'tracepoint:syscalls:sys_enter_* { @start[tid] = nsecs; }
             tracepoint:syscalls:sys_exit_* /@start[tid]/ {
                 @time[probe] = hist(nsecs - @start[tid]);
                 delete(@start[tid]);
             }'
```

### Exercise 4: Custom Tracing Scripts

**Objective:** Write more complex bpftrace programs.

```bash
# 1. Create a script to trace HTTP-like traffic
cat > trace_http.bt <<'EOF'
#!/usr/bin/env bpftrace

BEGIN {
    printf("Tracing TCP send/recv. Hit Ctrl-C to end.\n");
}

kprobe:tcp_sendmsg {
    $sk = (struct sock *)arg0;
    $size = arg2;

    printf("SEND: PID=%d size=%d\n", pid, $size);
}

kprobe:tcp_cleanup_rbuf {
    $sk = (struct sock *)arg0;
    $copied = arg1;

    printf("RECV: PID=%d size=%d\n", pid, $copied);
}

END {
    printf("Tracing ended\n");
}
EOF

chmod +x trace_http.bt
sudo ./trace_http.bt

# 2. Create a script to track TCP retransmissions
cat > track_retrans.bt <<'EOF'
#!/usr/bin/env bpftrace

BEGIN {
    printf("Tracking TCP retransmissions\n");
}

kprobe:tcp_retransmit_skb {
    $sk = (struct sock *)arg0;
    $inet = (struct inet_sock *)$sk;

    printf("Retrans: PID=%d COMM=%s\n", pid, comm);
    @retrans[comm] = count();
}

END {
    printf("\nRetransmission counts by process:\n");
    print(@retrans);
    clear(@retrans);
}
EOF

chmod +x track_retrans.bt
sudo ./track_retrans.bt
```

### Real-World Scenario 8: Finding Network Bottlenecks

**Problem:** Need to identify which part of network stack is slow.

```bash
# Create a comprehensive network latency tracker
cat > network_latency.bt <<'EOF'
#!/usr/bin/env bpftrace

BEGIN {
    printf("Measuring network stack latency...\n");
}

// Track packet receive path
kprobe:__netif_receive_skb_core {
    @start_rx[tid] = nsecs;
}

kretprobe:__netif_receive_skb_core /@start_rx[tid]/ {
    @rx_latency_us = hist((nsecs - @start_rx[tid]) / 1000);
    delete(@start_rx[tid]);
}

// Track IP layer processing
kprobe:ip_rcv {
    @start_ip[tid] = nsecs;
}

kretprobe:ip_rcv /@start_ip[tid]/ {
    @ip_latency_us = hist((nsecs - @start_ip[tid]) / 1000);
    delete(@start_ip[tid]);
}

// Track TCP layer processing
kprobe:tcp_v4_rcv {
    @start_tcp[tid] = nsecs;
}

kretprobe:tcp_v4_rcv /@start_tcp[tid]/ {
    @tcp_latency_us = hist((nsecs - @start_tcp[tid]) / 1000);
    delete(@start_tcp[tid]);
}

END {
    printf("\n=== RX Path Latency (microseconds) ===\n");
    print(@rx_latency_us);
    printf("\n=== IP Layer Latency (microseconds) ===\n");
    print(@ip_latency_us);
    printf("\n=== TCP Layer Latency (microseconds) ===\n");
    print(@tcp_latency_us);
}
EOF

chmod +x network_latency.bt
sudo ./network_latency.bt
```

---

## Real-World Production Scenarios

### Scenario 1: Website Timeout Investigation

**Problem:** Website intermittently returns 504 Gateway Timeout.

**Complete debugging workflow:**

```bash
# Phase 1: Confirm the problem
# ============================
# 1. Reproduce and capture
tcpdump -i any -nn -s0 'host webapp.example.com' -w timeout_investigation.pcap &
TCPDUMP_PID=$!

# Try to reproduce
for i in {1..100}; do
    time curl -v http://webapp.example.com
    sleep 1
done

kill $TCPDUMP_PID

# 2. Analyze packet capture
tcpdump -r timeout_investigation.pcap -nn | grep -E '(SYN|FIN|RST)' | head -50

# Phase 2: Check application behavior
# ====================================
# 3. Find the web server process
ps aux | grep nginx

# 4. Trace the web server's network calls
strace -p <NGINX_PID> -e trace=network -f 2>&1 | tee nginx_trace.log

# Watch for:
# - accept() calls (new connections)
# - send()/sendto() (responses being sent)
# - Errors like EAGAIN, EWOULDBLOCK

# Phase 3: Check kernel-level behavior
# =====================================
# 5. Monitor TCP retransmissions
/usr/share/bcc/tools/tcpretrans | grep webapp

# 6. Check for connection drops
/usr/share/bcc/tools/tcpdrop | grep webapp

# 7. Monitor connection latency
/usr/share/bcc/tools/tcpconnlat 100

# Phase 4: Deep analysis
# ======================
# 8. Create bpftrace script to trace full request lifecycle
cat > webapp_trace.bt <<'EOF'
#!/usr/bin/env bpftrace

kprobe:tcp_v4_connect {
    @conn_start[tid] = nsecs;
    printf("Connection attempt: PID=%d\n", pid);
}

kretprobe:tcp_v4_connect /@conn_start[tid]/ {
    $duration = (nsecs - @conn_start[tid]) / 1000000;
    printf("Connection established: %d ms\n", $duration);
    delete(@conn_start[tid]);
}

kprobe:tcp_sendmsg {
    @send_start[tid] = nsecs;
}

kretprobe:tcp_sendmsg /@send_start[tid]/ {
    $duration = (nsecs - @send_start[tid]) / 1000000;
    if ($duration > 100) {
        printf("SLOW SEND: %d ms\n", $duration);
    }
    delete(@send_start[tid]);
}
EOF

sudo bpftrace webapp_trace.bt

# Phase 5: System-level checks
# =============================
# 9. Check for connection tracking table overflow
cat /proc/sys/net/netfilter/nf_conntrack_count
cat /proc/sys/net/netfilter/nf_conntrack_max

# 10. Check socket statistics
ss -s

# 11. Check for socket buffer issues
sysctl net.core.rmem_max
sysctl net.core.wmem_max

# Phase 6: Results analysis
# =========================
# Compile findings:
echo "=== Investigation Summary ==="
echo "1. Packet capture analysis:"
echo "   - Total connections: $(tcpdump -r timeout_investigation.pcap -nn 'tcp[tcpflags] & tcp-syn != 0' | wc -l)"
echo "   - RST packets: $(tcpdump -r timeout_investigation.pcap -nn 'tcp[tcpflags] & tcp-rst != 0' | wc -l)"

echo "2. Application trace:"
echo "   - Network errors: $(grep -c 'EAGAIN\|EWOULDBLOCK' nginx_trace.log)"

echo "3. Retransmissions detected: $(grep -c webapp /path/to/tcpretrans.log)"
```

### Scenario 2: High Network Latency Debug

**Problem:** Application experiencing high network latency.

```bash
# Complete workflow
# =================

# 1. Baseline measurement
ping -c 100 remote-server.com | tee ping_baseline.txt
mtr --report --report-cycles 100 remote-server.com > mtr_report.txt

# 2. Detailed path analysis
traceroute -T -n remote-server.com

# 3. Application-level latency
time curl -v http://remote-server.com

# 4. TCP connection latency breakdown
/usr/share/bcc/tools/tcpconnlat 1 | grep remote-server

# 5. Monitor for retransmissions (indicates packet loss)
/usr/share/bcc/tools/tcpretrans > retrans_log.txt &
RETRANS_PID=$!

# Generate traffic
for i in {1..50}; do curl http://remote-server.com; done

kill $RETRANS_PID

# 6. Check TCP info for existing connections
ss -tiepo | grep remote-server

# Look for:
# - rto: Retransmission timeout
# - rtt: Round-trip time
# - cwnd: Congestion window
# - retrans: Retransmission count

# 7. Kernel-level latency analysis with bpftrace
cat > latency_breakdown.bt <<'EOF'
#!/usr/bin/env bpftrace

BEGIN {
    printf("Breaking down network latency\n");
}

// Measure time from application send to TCP layer
kprobe:sock_sendmsg {
    @app_send[tid] = nsecs;
}

kprobe:tcp_sendmsg /@app_send[tid]/ {
    @app_to_tcp_us = hist((nsecs - @app_send[tid]) / 1000);
    delete(@app_send[tid]);
}

// Measure time in TCP layer
kprobe:tcp_sendmsg {
    @tcp_send[tid] = nsecs;
}

kprobe:ip_queue_xmit /@tcp_send[tid]/ {
    @tcp_layer_us = hist((nsecs - @tcp_send[tid]) / 1000);
    delete(@tcp_send[tid]);
}

// Measure time to device transmission
kprobe:ip_queue_xmit {
    @ip_xmit[tid] = nsecs;
}

kprobe:dev_queue_xmit /@ip_xmit[tid]/ {
    @ip_to_dev_us = hist((nsecs - @ip_xmit[tid]) / 1000);
    delete(@ip_xmit[tid]);
}

END {
    printf("\n=== App to TCP (us) ===\n"); print(@app_to_tcp_us);
    printf("\n=== TCP Processing (us) ===\n"); print(@tcp_layer_us);
    printf("\n=== IP to Device (us) ===\n"); print(@ip_to_dev_us);
}
EOF

sudo bpftrace latency_breakdown.bt

# 8. Interface statistics
ethtool -S eth0 | grep -E '(error|drop|overrun)'

# 9. Ring buffer check
ethtool -g eth0
```

### Scenario 3: Connection Leak Detection

**Problem:** Server runs out of file descriptors, suspect connection leak.

```bash
# Investigation workflow
# ======================

# 1. Check current connection state
ss -tan | awk '{print $1}' | sort | uniq -c
# Look for excessive TIME_WAIT, CLOSE_WAIT, ESTABLISHED

# 2. Monitor connection lifecycle over time
/usr/share/bcc/tools/tcplife -w > tcplife_log.txt &
TCPLIFE_PID=$!

# Let it run for 10 minutes during normal operation
sleep 600
kill $TCPLIFE_PID

# 3. Analyze connection durations
echo "=== Long-lived connections ==="
awk '$8 > 300' tcplife_log.txt | sort -k8 -rn | head -20
# $8 is lifespan in seconds

# 4. Identify processes with many connections
cat > connection_tracker.bt <<'EOF'
#!/usr/bin/env bpftrace

BEGIN {
    printf("Tracking TCP connection open/close\n");
}

tracepoint:sock:inet_sock_set_state {
    if (args->newstate == 1) {  // TCP_ESTABLISHED
        @open[args->family, args->sport] = count();
        @by_process[comm] = count();
    }
    if (args->newstate == 7) {  // TCP_CLOSE
        @close[args->family, args->sport] = count();
    }
}

interval:s:10 {
    printf("\n=== Connection counts by process ===\n");
    print(@by_process);
    clear(@by_process);
}

END {
    printf("\n=== Opens by port ===\n"); print(@open);
    printf("\n=== Closes by port ===\n"); print(@close);
}
EOF

sudo bpftrace connection_tracker.bt

# 5. Check for CLOSE_WAIT accumulation (app not closing sockets)
watch -n 5 'ss -tan | grep CLOSE_WAIT | wc -l'

# 6. Find process with leaked connections
lsof -nP -iTCP | awk '{print $1}' | sort | uniq -c | sort -rn

# 7. Trace socket operations for suspicious process
PID=$(lsof -nP -iTCP | awk '{print $1, $2}' | sort | uniq -c | sort -rn | head -1 | awk '{print $3}')
echo "Tracing PID: $PID"
strace -p $PID -e trace=socket,close,shutdown -f

# 8. Check file descriptor limits
cat /proc/$PID/limits | grep files
lsof -p $PID | wc -l
```

### Scenario 4: Packet Loss Diagnosis

**Problem:** Users report degraded video quality, suspect packet loss.

```bash
# Comprehensive packet loss investigation
# ========================================

# 1. Confirm packet loss exists
mtr --report --report-cycles 1000 video-server.example.com > mtr_extended.txt

# Analyze results
grep "Loss%" mtr_extended.txt

# 2. Check interface statistics for drops
ip -s link show eth0
# Look for RX dropped, TX dropped

# 3. Detailed interface stats
ethtool -S eth0 | grep -i drop

# 4. Monitor kernel packet drops with reason
cat > packet_drops.bt <<'EOF'
#!/usr/bin/env bpftrace

#include <linux/skbuff.h>

BEGIN {
    printf("Tracking packet drops\n");
}

tracepoint:skb:kfree_skb {
    @drop_location[kstack] = count();
    @drop_count = count();
}

interval:s:10 {
    printf("Drops in last 10s: %d\n", @drop_count);
    @drop_count = 0;
}

END {
    printf("\n=== Drop locations ===\n");
    print(@drop_location);
}
EOF

sudo bpftrace packet_drops.bt &

# 5. Monitor for buffer overruns
watch -n 1 'ethtool -S eth0 | grep -E "(rx_.*_errors|rx_dropped|rx_missed)"'

# 6. Check receive buffer settings
sysctl net.core.netdev_max_backlog
sysctl net.core.rmem_max

# 7. Monitor ring buffer drops
ethtool -S eth0 | grep -E "rx_queue_.*_drops"

# 8. Check for conntrack drops
dmesg | grep -i "nf_conntrack: table full"
cat /proc/sys/net/netfilter/nf_conntrack_count
cat /proc/sys/net/netfilter/nf_conntrack_max

# 9. Capture packets around drop events
tcpdump -i any -nn -s0 -w packet_loss_capture.pcap &
TCPDUMP_PID=$!

# Generate test traffic
iperf3 -c video-server.example.com -t 60

kill $TCPDUMP_PID

# 10. Analyze capture for retransmissions
tcpdump -r packet_loss_capture.pcap -nn | grep -E "(retrans|dup ack)"

# 11. UDP packet loss (if applicable for video)
iperf3 -c video-server.example.com -u -b 10M -t 30
# Check the packet loss percentage in output
```

---

## Debugging Workflow Challenges

### Challenge 1: Mystery Connection Refusal

**Setup:** Create this scenario:
```bash
# Start a service on a port
python3 -m http.server 8000 &

# Add iptables rule to drop new connections
sudo iptables -A INPUT -p tcp --dport 8000 -j DROP
```

**Your task:** Using only tracing tools, determine why connections to port 8000 fail.

**Solution approach:**
```bash
# 1. Try to connect
curl http://localhost:8000  # This will hang

# 2. Capture packets
sudo tcpdump -i lo -nn port 8000 &
curl http://localhost:8000

# What you'll see: SYN packets sent, no SYN-ACK response

# 3. Trace with strace
strace curl http://localhost:8000
# Look for connect() call hanging

# 4. Check firewall
sudo iptables -L -v -n | grep 8000

# Cleanup
sudo iptables -D INPUT -p tcp --dport 8000 -j DROP
killall python3
```

### Challenge 2: Slow DNS Resolution

**Setup:**
```bash
# Add a slow DNS server to your config
echo "nameserver 1.2.3.4" | sudo tee /etc/resolv.conf.d/slowdns
```

**Task:** Identify that DNS is the bottleneck.

**Solution:**
```bash
# 1. Time a request
time curl http://example.com

# 2. Trace DNS lookups
strace -e trace=socket,connect,sendto,recvfrom curl http://example.com 2>&1 | grep -A5 -B5 '53'

# 3. Monitor DNS queries
sudo tcpdump -i any -nn port 53 &
curl http://example.com

# 4. Measure DNS latency directly
dig example.com

# 5. Use BCC to trace DNS
sudo /usr/share/bcc/tools/dnslatency

# Cleanup
sudo rm /etc/resolv.conf.d/slowdns
```

### Challenge 3: Container Network Debug

**Setup:** Create a container with network issues
```bash
docker run -d --name test_container nginx
```

**Task:** Debug why you can't connect to the container.

**Solution:**
```bash
# 1. Get container IP
docker inspect test_container | grep IPAddress

# 2. Try to connect
curl http://<container_ip>

# 3. Check container network namespace
PID=$(docker inspect -f '{{.State.Pid}}' test_container)
sudo nsenter -t $PID -n ip addr

# 4. Trace in container namespace
sudo nsenter -t $PID -n tcpdump -i any port 80 &
curl http://<container_ip>

# 5. Check iptables rules for docker
sudo iptables -t nat -L -n -v

# 6. Check bridge network
docker network inspect bridge

# Cleanup
docker stop test_container
docker rm test_container
```

---

## Practice Lab Setup

### Create a Practice Environment

```bash
# 1. Create network namespace for safe testing
sudo ip netns add testnet

# 2. Create virtual ethernet pair
sudo ip link add veth0 type veth peer name veth1

# 3. Move one end to namespace
sudo ip link set veth1 netns testnet

# 4. Configure interfaces
sudo ip addr add 10.0.0.1/24 dev veth0
sudo ip link set veth0 up

sudo ip netns exec testnet ip addr add 10.0.0.2/24 dev veth1
sudo ip netns exec testnet ip link set veth1 up

# 5. Test connectivity
ping -c 3 10.0.0.2

# 6. Run server in namespace
sudo ip netns exec testnet python3 -m http.server 8000 &

# 7. Practice tracing
sudo tcpdump -i veth0 -nn &
curl http://10.0.0.2:8000

# Cleanup
sudo ip netns del testnet
sudo ip link del veth0
```

---

## Additional Resources

### Online Practice Platforms
- **TryHackMe**: tcpdump room, network analysis challenges
- **LabEx**: Interactive Linux network labs
- **Wireshark Sample Captures**: https://wiki.wireshark.org/SampleCaptures

### Books and Guides
- "BPF Performance Tools" by Brendan Gregg
- "TCP/IP Illustrated" by W. Richard Stevens
- Official kernel documentation: /Documentation/trace/

### Tools Documentation
- tcpdump: https://www.tcpdump.org/manpages/tcpdump.1.html
- bpftrace: https://github.com/iovisor/bpftrace/blob/master/docs/tutorial_one_liners.md
- BCC: https://github.com/iovisor/bcc/blob/master/docs/tutorial.md
- Brendan Gregg's blog: http://www.brendangregg.com/

---

## Next Steps

After completing these exercises:

1. **Practice regularly** - Set up cron jobs to collect traces daily
2. **Build a toolkit** - Create custom bpftrace scripts for your environment
3. **Document findings** - Keep a log of interesting traces and what they revealed
4. **Contribute** - Share your scripts and findings with the community
5. **Stay current** - Follow kernel development, new tracing features

The key to mastery is practice. These exercises provide the foundation—now apply them to real problems in your environment.

# Linux TCP Slow-Start Pacing

To run the automated batch sweep directly, skip to [Appendix A: Batch Sweep](#appendix-a-batch-sweep). The script is self-contained and rebuilds the topology before every run.

## Network Topology Setup

Create three network namespaces (sender lpr-a, intermediate forwarder lpr-r, receiver lpr-b) connected via two veth pairs:
- lpr-a 10.0.1.1: TCP sender running Reno with FQ pacing.
- lpr-r 10.0.1.2 and 10.0.2.1: Intermediate forwarder with IPv4 forwarding and 50 ms delay each way (100 ms total RTT).
- lpr-b 10.0.2.2: TCP receiver.

```bash
sudo ip -all netns delete

sudo ip netns add lpr-a
sudo ip netns add lpr-r
sudo ip netns add lpr-b

sudo ip link add a0 netns lpr-a type veth peer name ra netns lpr-r
sudo ip link add rb netns lpr-r type veth peer name b0 netns lpr-b

sudo ip -n lpr-a addr add 10.0.1.1/24 dev a0
sudo ip -n lpr-r addr add 10.0.1.2/24 dev ra
sudo ip -n lpr-r addr add 10.0.2.1/24 dev rb
sudo ip -n lpr-b addr add 10.0.2.2/24 dev b0

sudo ip -n lpr-a link set a0 up
sudo ip -n lpr-r link set ra up
sudo ip -n lpr-r link set rb up
sudo ip -n lpr-b link set b0 up

sudo ip netns exec lpr-r sysctl -qw net.ipv4.ip_forward=1

sudo ip -n lpr-a route add default via 10.0.1.2
sudo ip -n lpr-b route add default via 10.0.2.1

sudo tc -n lpr-r qdisc add dev ra root netem delay 50ms limit 1000000
sudo tc -n lpr-r qdisc add dev rb root netem delay 50ms limit 1000000
sudo tc -n lpr-a qdisc add dev a0 root fq initial_quantum 1514 quantum 1514 timer_slack 0

sudo ip netns exec lpr-a sysctl -qw net.ipv4.tcp_congestion_control=reno net.ipv4.tcp_autocorking=0 net.ipv4.tcp_no_metrics_save=1
```

Pre-warm ARP so cold ARP resolution does not add a 50 ms offset to the handshake:

```bash
sudo ip netns exec lpr-a ping -c 1 -W 1 10.0.2.2 >/dev/null 2>&1
```

## Configuration Toggles

The four progressive configurations are applied cumulatively as needed:

```bash
# 1. default: Base topology is already the default baseline

# 2. buf64m (built on default: expand socket buffers to 64 MB):
sudo ip netns exec lpr-b sysctl -qw net.ipv4.tcp_rmem="4096 131072 67108864"
sudo ip netns exec lpr-a sysctl -qw net.ipv4.tcp_wmem="4096 16384 67108864"

# 3. offload_off (built on buf64m: disable NIC offloads):
sudo ip netns exec lpr-a ethtool -K a0 tso off gso off
sudo ip netns exec lpr-r ethtool -K ra gro off
sudo ip netns exec lpr-b ethtool -K b0 gro off

# 4. quickack (built on offload_off: enable immediate ACKs):
sudo ip -n lpr-b route change default via 10.0.2.1 quickack 1
```

Parameter adjustments:
```bash
# Slow-start pacing ratio (e.g. 139 for 2 ln 2, 200 for 2.0)
sudo ip netns exec lpr-a sysctl -qw net.ipv4.tcp_pacing_ss_ratio=139

# Initial congestion window initcwnd (e.g. 10 or 1)
sudo ip -n lpr-a route change default via 10.0.1.2 initcwnd 10
```

## Single Transfer and Capture

Start nc receiver on lpr-b, capture packets on intermediate forwarder lpr-r, transfer 128 MB, and export traces with [tcptrace](http://tcptrace.org/):

```bash
sudo ip netns exec lpr-b nc -l -p 5001 >/dev/null &
sudo ip netns exec lpr-r tcpdump -i ra -nn -s 96 -q -w packets.pcap tcp port 5001 &
sleep 0.2
head -c 128M /dev/zero | sudo ip netns exec lpr-a nc -N 10.0.2.2 5001
sleep 0.2
sudo pkill -INT tcpdump

# Export xplot traces
tcptrace -n -l -r -S -N -zx -zy packets.pcap > tcptrace.txt
java -jar ../xpl/jPlot.jar a2b_tsg.xpl
java -jar ../xpl/jPlot.jar a2b_owin.xpl
```

jPlot legend:
- `a2b_tsg.xpl` (time-sequence graph): white dots are transmitted packets, green line is cumulative ACK, yellow line is advertised receive window limit (ACK + rwin), purple S marks SACK blocks, red R marks retransmissions.
- `a2b_owin.xpl` (outstanding-window graph): white line is bytes in flight, yellow line is advertised receive window limit.

## Network Teardown

```bash
sudo ip -all netns delete
```

## Appendix A: Batch Sweep

Automated sweep across all 112 configurations: default, buf64m, offload_off, quickack across iw1 and iw10, 128M transfer, gains 100, 120, 139, 150, 200, 400, 800, and 2 repeats:

```bash
sudo bash -c '
cleanup() {
    pkill -9 nc 2>/dev/null || true
    pkill -INT tcpdump 2>/dev/null || true
    for ns in lpr-a lpr-r lpr-b; do ip netns del "$ns" 2>/dev/null || true; done
}

setup() {
    ip netns add lpr-a
    ip netns add lpr-r
    ip netns add lpr-b

    ip link add a0 netns lpr-a type veth peer name ra netns lpr-r
    ip link add rb netns lpr-r type veth peer name b0 netns lpr-b

    ip -n lpr-a addr add 10.0.1.1/24 dev a0
    ip -n lpr-r addr add 10.0.1.2/24 dev ra
    ip -n lpr-r addr add 10.0.2.1/24 dev rb
    ip -n lpr-b addr add 10.0.2.2/24 dev b0

    ip -n lpr-a link set a0 up
    ip -n lpr-r link set ra up
    ip -n lpr-r link set rb up
    ip -n lpr-b link set b0 up

    ip netns exec lpr-r sysctl -qw net.ipv4.ip_forward=1

    ip -n lpr-a route add default via 10.0.1.2
    ip -n lpr-b route add default via 10.0.2.1

    tc -n lpr-r qdisc add dev ra root netem delay 50ms limit 1000000
    tc -n lpr-r qdisc add dev rb root netem delay 50ms limit 1000000
    tc -n lpr-a qdisc add dev a0 root fq initial_quantum 1514 quantum 1514 timer_slack 0

    ip netns exec lpr-a sysctl -qw net.ipv4.tcp_congestion_control=reno net.ipv4.tcp_autocorking=0 net.ipv4.tcp_no_metrics_save=1

    ip netns exec lpr-a ping -c 1 -W 1 10.0.2.2 >/dev/null 2>&1 || true
}

for cfg in default buf64m offload_off quickack; do
    for iw in 1 10; do
        for size in 128; do
            for gain in 100 120 139 150 200 400 800; do
                for r in 1 2; do
                    cleanup
                    sleep 0.2
                    setup

                    case $cfg in
                        buf64m)
                            ip netns exec lpr-b sysctl -qw net.ipv4.tcp_rmem="4096 131072 67108864"
                            ip netns exec lpr-a sysctl -qw net.ipv4.tcp_wmem="4096 16384 67108864"
                            ;;
                        offload_off)
                            ip netns exec lpr-b sysctl -qw net.ipv4.tcp_rmem="4096 131072 67108864"
                            ip netns exec lpr-a sysctl -qw net.ipv4.tcp_wmem="4096 16384 67108864"
                            ip netns exec lpr-a ethtool -K a0 tso off gso off
                            ip netns exec lpr-r ethtool -K ra gro off
                            ip netns exec lpr-b ethtool -K b0 gro off
                            ;;
                        quickack)
                            ip netns exec lpr-b sysctl -qw net.ipv4.tcp_rmem="4096 131072 67108864"
                            ip netns exec lpr-a sysctl -qw net.ipv4.tcp_wmem="4096 16384 67108864"
                            ip netns exec lpr-a ethtool -K a0 tso off gso off
                            ip netns exec lpr-r ethtool -K ra gro off
                            ip netns exec lpr-b ethtool -K b0 gro off
                            ip -n lpr-b route change default via 10.0.2.1 quickack 1
                            ;;
                    esac

                    ip -n lpr-a route change default via 10.0.1.2 initcwnd $iw
                    ip netns exec lpr-a sysctl -qw net.ipv4.tcp_pacing_ss_ratio=$gain

                    dir="data/iw${iw}/${size}M/$cfg/g${gain}-r${r}"
                    mkdir -p "$dir"
                    echo "[$cfg] iw=$iw ${size}M g=$gain r=$r"

                    ip netns exec lpr-b nc -l -p 5001 >/dev/null &
                    ip netns exec lpr-r tcpdump -i ra -nn -s 96 -q -w "$dir/packets.pcap" tcp port 5001 &
                    sleep 0.2
                    head -c ${size}M /dev/zero | ip netns exec lpr-a nc -N 10.0.2.2 5001
                    sleep 0.2
                    pkill -INT tcpdump
                    wait

                    (cd "$dir" && tcptrace -n -l -r -S -N -zx -zy packets.pcap > tcptrace.txt)
                done
            done
        done
    done
done

cleanup
'
```

## Appendix B: Trace Analysis

```python
import os, re, subprocess

def analyze(path):
    p_tcp = os.path.join(path, "tcptrace.txt")
    p_pcap = os.path.join(path, "packets.pcap")
    if not (os.path.exists(p_tcp) and os.path.exists(p_pcap)):
        return None
    with open(p_tcp) as f:
        txt = f.read()
    m_time = re.search(r"elapsed time:\s+0:00:([\d\.]+)", txt)
    m_uniq = re.search(r"unique bytes sent:\s+(\d+)", txt)
    m_act = re.search(r"actual data bytes:\s+(\d+)", txt)
    m_rex = re.search(r"rexmt data pkts:\s+(\d+)", txt)
    elapsed = float(m_time.group(1)) if m_time else 0.0
    uniq = int(m_uniq.group(1)) if m_uniq else 0
    act = int(m_act.group(1)) if m_act else 0
    rexmt = int(m_rex.group(1)) if m_rex else 0
    eff = (uniq / act * 100.0) if act else 0.0
    goodput = (uniq * 8.0) / (elapsed * 1e6) if elapsed else 0.0
    cmd = f"tshark -r {p_pcap} -Y 'ip.src==10.0.1.1 && tcp.len > 0' -T fields -e frame.time_relative"
    out = subprocess.check_output(cmd, shell=True, text=True)
    ts = [float(l.strip()) for l in out.strip().split("\n") if l.strip()]
    def top3(t_start, t_end):
        w_ts = [t for t in ts if t_start <= t <= t_end]
        if len(w_ts) < 2:
            return [0.0, 0.0, 0.0]
        gaps = sorted([(w_ts[i+1] - w_ts[i]) * 1000.0 for i in range(len(w_ts)-1)], reverse=True)
        res = [round(g, 1) for g in gaps[:3]]
        while len(res) < 3:
            res.append(0.0)
        return res
    return (elapsed, rexmt, eff, goodput, top3(elapsed * 0.1, elapsed * 0.2), top3(elapsed * 0.4, elapsed * 0.6), top3(elapsed * 0.8, elapsed * 0.9))

def sort_key(item):
    path, name = item
    tokens = re.split(r'(\d+)', name)
    return [(0, int(t)) if t.isdigit() else (1, t.lower()) for t in tokens]

runs = []
if os.path.exists("tcptrace.txt") and os.path.exists("packets.pcap"):
    runs.append((".", os.path.basename(os.path.abspath("."))))
else:
    for root, dirs, files in os.walk("."):
        if "tcptrace.txt" in files and "packets.pcap" in files:
            runs.append((root, os.path.relpath(root, ".")))
runs.sort(key=sort_key)

col_w = max(10, max(len(name) for _, name in runs)) if runs else 10
header = f"{'Run':{col_w}s} | {'Time(s)':7s} | {'Rexmt':5s} | {'Efficiency':10s} | {'Goodput':12s} | {'10-20% Top 3 (ms)':20s} | {'40-60% Top 3 (ms)':20s} | {'80-90% Top 3 (ms)':20s}"
print(header)
print("-" * len(header))

for path, name in runs:
    res = analyze(path)
    if res:
        el, rx, ef, gp, e_g, m_g, l_g = res
        print(f"{name:{col_w}s} | {el:7.2f} | {rx:5d} | {ef:9.2f}% | {gp:9.2f} Mbps | {str(e_g):20s} | {str(m_g):20s} | {str(l_g):20s}", flush=True)
```

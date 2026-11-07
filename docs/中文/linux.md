# Linux TCP 慢启动 Pacing

如需直接运行自动化批量扫参，可跳至 [附录 A：批量扫参](#附录-a批量扫参)。脚本完全自包含，每次运行前会自动重建网络拓扑。

## 网络拓扑搭建

创建三个网络命名空间（发送端 lpr-a、中间转发节点 lpr-r、接收端 lpr-b），通过两对虚拟网卡连接：
- lpr-a 10.0.1.1：TCP 发送端，使用 Reno 拥塞控制与 FQ Pacing。
- lpr-r 10.0.1.2 与 10.0.2.1：中间转发节点，开启 IPv4 转发，配置双向各 50 ms 延迟模拟真实链路（总 RTT 100 ms）。
- lpr-b 10.0.2.2：TCP 接收端。

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

预热 ARP 表，避免冷启动 ARP 解析给三次握手额外引入 50 ms 偏移：

```bash
sudo ip netns exec lpr-a ping -c 1 -W 1 10.0.2.2 >/dev/null 2>&1
```

## 配置切换

四组递进配置按需累加执行：

```bash
# 1. default: 拓扑建好后即为默认基准

# 2. buf64m (在 default 基础上扩展套接字缓冲区至 64 MB):
sudo ip netns exec lpr-b sysctl -qw net.ipv4.tcp_rmem="4096 131072 67108864"
sudo ip netns exec lpr-a sysctl -qw net.ipv4.tcp_wmem="4096 16384 67108864"

# 3. offload_off (在 buf64m 基础上关闭硬件卸载):
sudo ip netns exec lpr-a ethtool -K a0 tso off gso off
sudo ip netns exec lpr-r ethtool -K ra gro off
sudo ip netns exec lpr-b ethtool -K b0 gro off

# 4. quickack (在 offload_off 基础上开启即时 ACK):
sudo ip -n lpr-b route change default via 10.0.2.1 quickack 1
```

测试参数调整：
```bash
# 慢启动 Pacing 增益比例 (如 139 对应 2 ln 2，200 对应 2.0)
sudo ip netns exec lpr-a sysctl -qw net.ipv4.tcp_pacing_ss_ratio=139

# 初始拥塞窗口 initcwnd (如 10 或 1)
sudo ip -n lpr-a route change default via 10.0.1.2 initcwnd 10
```

## 单次传输与抓包

在接收端启动 nc 接收，在中间节点抓包，传输 128 MB 并用 [tcptrace](http://tcptrace.org/) 导出轨迹：

```bash
sudo ip netns exec lpr-b nc -l -p 5001 >/dev/null &
sudo ip netns exec lpr-r tcpdump -i ra -nn -s 96 -q -w packets.pcap tcp port 5001 &
sleep 0.2
head -c 128M /dev/zero | sudo ip netns exec lpr-a nc -N 10.0.2.2 5001
sleep 0.2
sudo pkill -INT tcpdump

# 导出 xplot 轨迹
tcptrace -n -l -r -S -N -zx -zy packets.pcap > tcptrace.txt
java -jar ../../xpl/jPlot.jar a2b_tsg.xpl
java -jar ../../xpl/jPlot.jar a2b_owin.xpl
```

jPlot 轨迹图例：
- `a2b_tsg.xpl`（时序图）：白点为发送数据包，绿线为接收端累计 ACK，黄线为接收窗口上限（ACK + rwin），紫色 S 为 SACK 块，红色 R 为重传。
- `a2b_owin.xpl`（在途窗口图）：白线为在途字节数，黄线为接收窗口上限。

## 清理拓扑

```bash
sudo ip -all netns delete
```

## 附录 A：批量扫参

112 组配置全矩阵自动化扫参脚本，覆盖 default、buf64m、offload_off、quickack 四种环境，iw1 与 iw10 两种初始窗口，128M 传输大小，100、120、139、150、200、400、800 七种增益，每组重复 2 次：

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

## 附录 B：轨迹统计分析

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

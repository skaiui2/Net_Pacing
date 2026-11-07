# SSCP 慢启动 Pacing

研究分支：https://github.com/skaiui2/SSCP/tree/research/slowstart-nextslot-q16-20261022

## 源码实现与测试环境

- 发包间隔计算、发送时间规划与定时器放行：[sscp.c 第 1267-1435 行](https://github.com/skaiui2/SSCP/blob/research/slowstart-nextslot-q16-20261022/sscp.c#L1267-L1435)
- 增益常量与时间定义：[sscp.h 第 59-62 行](https://github.com/skaiui2/SSCP/blob/research/slowstart-nextslot-q16-20261022/sscp.h#L59-L62)
- 虚拟网络测试载体（1 µs 步长时钟，双向各 50 ms FIFO 延迟）：[test/virtual/vnet.c](https://github.com/skaiui2/SSCP/blob/research/slowstart-nextslot-q16-20261022/test/virtual/vnet.c)

## 扫参范围与结果

在 1 GiB 传输、100 ms RTT、0 丢包环境下，开展了覆盖 1.0 到 8.0 的 25 组增益详细扫参（在 1.0 到 2.0 之间高密采样）：
- 扫参增益列表：`g100`、`g108_6`、`g110`、`g113_6`、`g118_6`、`g120`、`g123_6`、`g128_6`、`g133_6`、`g138_6`（2 ln 2）、`g143_6`、`g148_6`、`g150`、`g153_6`、`g158_6`、`g163_6`、`g168_6`、`g175`、`g200`、`g250`、`g300`、`g400`、`g600`、`g800` 以及 `native`。
- 扫参表明：增益低于 2 ln 2 时传输完成时间显著变长；增益高于 2 ln 2 时完成时间收益迅速进入边际递减，但发包跨度急剧塌陷并重现严重突发。在完全分散数据包与维持全速窗口增长之间，最优参数落在 2 ln 2（≈ 1.386）附近。

## 本地编译与切片复现

运行 1 GiB 传输并截取前 16 MB 轨迹供 [jPlot](../../xpl/jPlot.jar) 查看：

```sh
git clone -b research/slowstart-nextslot-q16-20261022 https://github.com/skaiui2/SSCP.git
cd SSCP

cmake -S test -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build --target sscp_research -j

python test/research/run_sweep.py --mode ss --gain-id g138_6 --bytes 1073741824 --bin build/sscp_research --out results/g138_6
python test/slice_xpl.py results/g138_6/xpl/a2b_tsg.xpl --max-mb 16 -o g138_6_16mb.xpl
```

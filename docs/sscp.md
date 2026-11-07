# SSCP Slow-Start Pacing

Research branch: https://github.com/skaiui2/SSCP/tree/research/slowstart-nextslot-q16-20261022

## Source Code References

- Interval calculation, timestamp scheduling, and timer expiration: [sscp.c lines 1267-1435](https://github.com/skaiui2/SSCP/blob/research/slowstart-nextslot-q16-20261022/sscp.c#L1267-L1435)
- Gain constants and timing definitions: [sscp.h lines 59-62](https://github.com/skaiui2/SSCP/blob/research/slowstart-nextslot-q16-20261022/sscp.h#L59-L62)
- Virtual network harness (1 µs clock, dual 50 ms FIFO queues): [test/virtual/vnet.c](https://github.com/skaiui2/SSCP/blob/research/slowstart-nextslot-q16-20261022/test/virtual/vnet.c)

## Parameter Sweep (25 Gains)

Under 1 GiB transfer, 100 ms RTT, and 0 loss:
- Swept gains: `g100`, `g108_6`, `g110`, `g113_6`, `g118_6`, `g120`, `g123_6`, `g128_6`, `g133_6`, `g138_6` (2 ln 2), `g143_6`, `g148_6`, `g150`, `g153_6`, `g158_6`, `g163_6`, `g168_6`, `g175`, `g200`, `g250`, `g300`, `g400`, `g600`, `g800`, and `native`.
- Sweep findings: Gains below 2 ln 2 prolong completion time; gains above 2 ln 2 yield diminishing completion returns while collapsing packet dispersion into early bursts. Balancing full RTT dispersion against unimpeded window growth points to 2 ln 2 (≈ 1.386).

## Reproduce and Slice

Run the 1 GiB transfer and slice the first 16 MB trace for [jPlot](../xpl/jPlot.jar):

```sh
git clone -b research/slowstart-nextslot-q16-20261022 https://github.com/skaiui2/SSCP.git
cd SSCP

cmake -S test -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build --target sscp_research -j

python test/research/run_sweep.py --mode ss --gain-id g138_6 --bytes 1073741824 --bin build/sscp_research --out results/g138_6
python test/slice_xpl.py results/g138_6/xpl/a2b_tsg.xpl --max-mb 16 -o g138_6_16mb.xpl
```

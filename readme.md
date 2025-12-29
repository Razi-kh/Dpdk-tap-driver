# High-Speed Protocol Stack  
## DPDK TAP Driver Performance Analysis

---

## 1. DPDK Installation and Build Configuration

First, DPDK is downloaded and installed. Then, the build environment is configured using **Meson** with function instrumentation enabled.


![DPDK build configuration](images/Picture1.png)

```bash
meson setup build \
  -Dexamples=all \
  -Dlibdir=lib \
  -Denable_trace_fp=true \
  -Dc_args="-finstrument-functions"
```
Dc_args="-finstrument-functions" flag guarantees that function entry and exit points are marked during compilation; otherwise, **LTTng traces will be empty**.
---

## 2. Huge Pages Configuration

Huge pages are configured to provide large contiguous memory regions required for high-speed packet processing in DPDK.
```bash
echo 1024 > /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages
mkdir /mnt/huge
mount -t hugetlbfs pagesize=1GB /mnt/huge
```
![Huge pages configuration](images/Picture2.png)

---

## 3. Kernel Version Check and TAP Interface Setup

The kernel version corresponding to `liblttng-ust-cyg-profile.so` is verified.  
```bash
sudo LD_PRELOAD=/usr/lib/x86_64-linux-gnu/liblttng-ust-cyg-profile.so.1 ./app/dpdk-testpmd -l 0-1 -n 2   --vdev=net_tap0,iface=tap0   --vdev=net_tap1,iface=tap1   --   -i
```
Two **TAP interfaces** are created and pinned to two CPUs. Initially, the port status shows no traffic.

![TAP interfaces setup](images/Picture3.png)

---

## 4. RX/TX Queue Configuration

Ports are stopped, new **RX and TX queues** are added, and the ports are restarted.

![RX/TX queues](images/Picture4.png)

---

## 5. Forwarding Rules Configuration

The command below is used to verify the forwarding configuration:
show config fwd
```bash
flow create 0 ingress pattern eth / ipv4 / udp / end actions queue index 0 / end
```
A flow rule is created so that packets matching **Ethernet, IPv4, and UDP** headers are forwarded to **Queue 0**.

![Forwarding rule](images/Picture5.png)

---

## 6. Traffic Generation Using Tcpreplay

`tcpreplay` is installed and configured to simulate network traffic load.

![Tcpreplay setup](images/Picture6.png)

---

## 7. Packet Capture with Wireshark

Traffic is captured on the TAP interface using **Wireshark** and saved for later replay.

![Wireshark capture](images/Picture7.png)

---

## 8. Traffic Replay

The captured traffic is replayed, and the generated packet flow is observed on the interfaces.

![Traffic replay](images/Picture8.png)
```bash
tcpreplay -i tap0 --loop=1000 ./Capture.pcap 
```
![Interface traffic observation](images/Picture9.png)

---

## 9. LTTng Tracing Setup

An **LTTng tracing script** is written and executed. The results are analyzed using **Trace Compass**.
```bash
#!/bin/bash
lttng create libpcap
lttng enable-channel --userspace --num-subbuf=4 --subbuf-size=40M channel0
#lttng enable-channel --userspace channel0
lttng enable-event --channel channel0 --userspace --all
lttng add-context --channel channel0 --userspace --type=vpid --type=vtid --type=procname
lttng start
sleep 2
lttng stop
lttng destroy
```
![LTTng tracing script](images/Picture10.png)

---

## 10. CPU Consumption Analysis

A **2-second LTTng recording** (approximately 1 million events) shows that most of the additional CPU overhead observed after applying the software flow rule is consumed by the **TAP PMD RX hot path**.

As shown in the pie charts:
- ~50% of events correspond to `func_entry`
- ~50% correspond to `func_exit`

![CPU event distribution](images/Picture11.png)


---

## 11. Time Chart Analysis – Bimodal Oscillation

The **Time Chart** reveals severe **bimodal oscillatory behavior**, where execution time periodically spikes.

This square-pattern behavior is a classical indicator of **cache depletion and refill cycles**, confirming that the current **Mempool configuration cannot sustain the incoming traffic rate**.

![Function entry vs exit](images/Picture12.png)


---

## 12. Execution Pattern Overview
![Time chart](images/Picture13.png)

![Cache depletion](images/Picture14.png)

![Oscillatory execution](images/Picture15.png)

![Execution pattern](images/Picture16.png)

**Observed execution flow:**
Receive → Process → Transmit → Free Memory.

This represents the main packet-processing pipeline in DPDK.

---

## 13. Main Loop Analysis (`pkt_burst_io_forward`)

- **Total Duration:** ~1.5 seconds  
- **Number of Calls:** 48,728  
- **Average Duration:** 32 µs  
- **Maximum Duration:** **28.5 ms**

**Critical Insight:**  
The difference between the average and maximum execution time (~1000×) indicates severe intermittent stalls in the processing loop.

---

## 14. TX Path Analysis

- **Duration:** ~300 ms  
- **Label:** `common_fwd_stream_transmit`

**Functions involved:**
- `rte_eth_tx_burst`
- `pmd_tx_burst`
- `rte_pktmbuf_free`

**Observation:**  
The TX path is relatively efficient and does not represent a major performance concern.

---

## 15. RX Path Analysis – Main Bottleneck

- **RX Duration:** **~1.1 seconds**
- **CPU Utilization:** **>73%**
- **Maximum System Latency:** 28.5 ms
- **Maximum RX Latency:** **22.3 ms**

**Functions involved:**
- `rte_eth_rx_burst` — consumes nearly **⅔ of total CPU time**
- `pmd_rx_burst`
- `rte_pktmbuf_alloc`

RX processing is approximately **4× more expensive than TX**.

---

## 16. Layered Bottleneck Analysis

- **General Flow:** RX dominates with over **73%** of total CPU time  
- **Driver Level:** `pmd_rx_burst` causes critical **22 ms delays**  
- **Inefficiency:** **99% empty RX polls**, indicating inefficient kernel–user signaling

The system bottleneck is the **inefficient RX path of the TAP driver**, which relies heavily on kernel–user context switches.

---

## 17. Root Cause Analysis

Frequent **kernel ↔ user context switches**, combined with intensive memory allocation (`rte_pktmbuf_alloc`), impose significant overhead and prevent DPDK from reaching its expected performance.

---

## 18. Proposed Solutions

1. Analyze sender-side behavior  
2. Increase RX burst size  
3. Replace the TAP driver with a more efficient alternative  
4. Enable deeper kernel-level tracing  

---

## 19. Final Results and Conclusion

Because the TAP driver relies heavily on **system calls** and **CPU-based memory copying**, most processing occurs in the kernel and appears as a black box at the user level.

Trace Compass analysis confirms:
- `pmd_rx_burst` is the primary bottleneck  
- Memory allocation consumes nearly **half of the driver execution time**  
- Suboptimal **Mempool configuration** amplifies RX latency  

✅ Increasing the **Mempool cache size** can significantly reduce latency and improve overall RX performance.

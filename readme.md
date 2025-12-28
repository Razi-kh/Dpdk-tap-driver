# High-Speed Protocol Stack  
## DPDK TAP Driver Performance Analysis

---

## 1. DPDK Installation and Build Configuration

First, DPDK is downloaded and installed. Then, the build environment is configured using **Meson** with function instrumentation enabled.
-ffunction-sections -finstrument-functions


content_copy
text

This flag guarantees that function entry and exit points are marked during compilation; otherwise, **LTTng traces will be empty**.

![DPDK build configuration](images/picture1.png)

---

## 2. Huge Pages Configuration

Huge pages are configured to provide large contiguous memory regions required for high-speed packet processing in DPDK.

![Huge pages configuration](images/picture2.png)

---

## 3. Kernel Version Check and TAP Interface Setup

The kernel version corresponding to `liblttng-ust-cyg-profile.so` is verified.  
Two **TAP interfaces** are created and pinned to two CPUs. Initially, the port status shows no traffic.

![TAP interfaces setup](images/picture3.png)

---

## 4. RX/TX Queue Configuration

Ports are stopped, new **RX and TX queues** are added, and the ports are restarted.

![RX/TX queues](images/picture4.png)

---

## 5. Forwarding Rules Configuration

The command below is used to verify the forwarding configuration:
show config fwd


content_copy
text

A flow rule is created so that packets matching **Ethernet, IPv4, and UDP** headers are forwarded to **Queue 0**.

![Forwarding rule](images/picture5.png)

---

## 6. Traffic Generation Using Tcpreplay

`tcpreplay` is installed and configured to simulate network traffic load.

![Tcpreplay setup](images/picture6.png)

---

## 7. Packet Capture with Wireshark

Traffic is captured on the TAP interface using **Wireshark** and saved for later replay.

![Wireshark capture](images/picture7.png)

---

## 8. Traffic Replay

The captured traffic is replayed, and the generated packet flow is observed on the interfaces.

![Traffic replay](images/picture8.png)

![Interface traffic observation](images/picture9.png)

---

## 9. LTTng Tracing Setup

An **LTTng tracing script** is written and executed. The results are analyzed using **Trace Compass**.

![LTTng tracing script](images/picture10.png)

---

## 10. CPU Consumption Analysis

A **2-second LTTng recording** (approximately 1 million events) shows that most of the additional CPU overhead observed after applying the software flow rule is consumed by the **TAP PMD RX hot path**.

As shown in the pie charts:
- ~50% of events correspond to `func_entry`
- ~50% correspond to `func_exit`

![CPU event distribution](images/picture11.png)

![Function entry vs exit](images/picture12.png)

---

## 11. Time Chart Analysis – Bimodal Oscillation

The **Time Chart** reveals severe **bimodal oscillatory behavior**, where execution time periodically spikes.

This square-pattern behavior is a classical indicator of **cache depletion and refill cycles**, confirming that the current **Mempool configuration cannot sustain the incoming traffic rate**.

![Time chart](images/picture13.png)

![Cache depletion](images/picture14.png)

![Oscillatory execution](images/picture15.png)

---

## 12. Execution Pattern Overview

![Execution pattern](images/picture16.png)

**Observed execution flow:**
Receive → Process → Transmit → Free Memory


content_copy
text

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

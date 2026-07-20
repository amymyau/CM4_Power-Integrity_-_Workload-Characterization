# CM4 Hardware-in-the-Loop (HIL) Power Integrity & Workload Characterization

An embedded system-level validation framework built on the Raspberry Pi Compute Module 4 (CM4). This project correlates low-level asynchronous C++ software execution with physical-layer transient response ($V_{\text{droop}}$ / AC ripple) on the 3.3V PCIe power delivery network (PDN).

Using modern `libgpiod` v2.x and high-throughput asynchronous block I/O (`io_uring`), this harness generates deterministic, high $di/dt$ step-load bursts while toggling a hardware GPIO trigger for oscilloscope synchronization.

---

## 🛠 System Architecture

## 🛠 System Architecture

```text
+-----------------------------------------------------------------------------------+
| Raspberry Pi Compute Module 4 (CM4 IO Platform)                                   |
|                                                                                   |
|  +--------------------------------+   libgpiod v2.x   +------------------------+  |
|  | C++ Workload Generator         |------------------>| GPIO 17 (Sync Trigger) |  |
|  | - liburing (O_DIRECT, QD=32)   |                   +------------------------+  |
|  | - 4-Core ARM NEON SIMD Stress  |                               │               |
|  +--------------------------------+                               │ Rising Edge   |
|                  │                                                │ Trigger       |
|                  ▼ PCIe Direct I/O Burst                          │               |
|  +-------------------------------------------------------------+  │               |
|  | 3.3V PCIe Power Rail (MLCC Decoupling near NVMe Slot)        |  │               |
|  +-------------------------------------------------------------+  │               |
+----------------------------------│--------------------------------│---------------+
                                   │                                │
                                   ▼ AC-Coupled (20MHz BW Limit)    ▼
                   +--------------------------------------------------+
                   | Rigol Digital Oscilloscope                       |
                   | CH1 (Yellow): 3.3V PCIe Rail Noise (10mV/div)    |
                   | CH2 (Cyan):   GPIO 17 Software Trigger (500mV/div)|
                   +--------------------------------------------------+



## 📊 Physical Bench Results & Transient Capture

High-queue-depth unbuffered block reads (`O_DIRECT`, 1MB transfers) were issued over the PCIe x1 bus to an NVMe SSD simultaneously with a 4-core ARM NEON SIMD load step.



### Key Metrics & Observations
* **GPIO Trigger Pulse Width ($\text{CH2}$):** $135\text{ ms}$ software execution window.
* **Thread Setup Latency:** $\sim 50\text{ ms}$ delay between rising edge and bus saturation during `io_uring` ring buffer setup.
* **Active I/O Burst Window ($\Delta X$):** **$83.20\text{ ms}$** high-current transient window.
* **Peak-to-Peak Rail Swing Expansion:** 
  * **Idle Baseline Noise:** $\sim 20\text{--}25\text{ mV}_{\text{pk-pk}}$
  * **Active PCIe/NVMe Load Burst:** **$>60\text{--}80\text{ mV}_{\text{pk-pk}}$** ($>3\times$ expansion over idle baseline).

---

## 🔑 Key Engineering Highlights

* **Modern Linux Character Device GPIO (`libgpiod` v2.x):** Utilizes `gpiod::line_config`, `gpiod::line_settings`, and `gpiod::request_config` with RAII resource management for microsecond-accurate pin control.
* **Kernel Cache Bypass (`O_DIRECT` + `io_uring`):** Maximizes PCIe bus current step-loads ($di/dt$) by executing unbuffered asynchronous block I/O at a queue depth of 32 directly to `/dev/nvme0n1`.
* **Multi-Core SIMD Synchronization:** Coordinates 4 concurrent ARM NEON SIMD worker threads aligned via atomic spinlock barriers to hit peak compute current simultaneously.
* **Physical-Layer Correlation:** Establishes direct visual correlation between low-level C++ software execution timing and power integrity performance on target hardware.

---

## 🚀 Getting Started

### Prerequisites

Install the GCC C++ toolchain, `libgpiod` v2.x headers, and `liburing`:

```bash
sudo apt update
sudo apt install -y build-essential cmake libgpiod-dev gpiod liburing-dev
CompilationCompile the C++17 source code with ARM v8-A SIMD optimizations and link against libgpiodcxx and liburing:Bashg++ -O3 -march=armv8-a+simd -pthread -std=c++17 \
    nvme_pcie_stress.cpp -o nvme_pcie_stress \
    -lgpiodcxx -lgpiod -luring
ExecutionRun with sudo permissions (required for character device access to /dev/gpiochip4 and raw block device access to /dev/nvme0n1):Bashsudo ./nvme_pcie_stress
🔬 Oscilloscope Configuration GuideCH1 (Power Rail Probe): Connected across MLCC capacitor near PCIe connector.Coupling: AC CouplingScale: $10\text{ mV/div}$Bandwidth Limit: $20\text{ MHz}$ (enabled to filter high-frequency ambient RF clutter)CH2 (Sync Trigger): Connected to GPIO 17.Coupling: DC CouplingScale: $500\text{ mV/div}$Trigger Type: Single / Normal on Rising Edge at $1.5\text{ V}$


# LLM Runner by CPU BYTE

> **A fully local, offline GGUF inference engine and chat client for 64-bit Android devices.**

LLM Runner is a fully local, offline GGUF inference engine and chat client engineered specifically for **64-bit Android devices (arm64-v8a)**. It allows users to run local language models directly on smartphone hardware without root access, terminal environments, external runtimes, or network connectivity.

**LLM Runner is free to use for everyone.**

---

## Direct Download

Download the latest pre-compiled APK directly from the official Releases page:

<p align="center">
  <a href="https://github.com/cpubyteofficial/LLM-Runner-Android/releases/latest">
    <img src="https://img.shields.io/badge/Download%20Latest%20APK-2ea44f?style=for-the-badge&logo=android&logoColor=white" alt="Download LLM Runner APK">
  </a>
</p>

* **Target Architecture:** ARM64 (`arm64-v8a`)
* **Minimum Android Version:** Android 8.0 (API Level 26) or higher
* **Memory Scaling:** Runs small models fully in RAM, and streams larger multi-billion parameter models from storage using dedicated memory modes.

---

## Support Development

<p align="center">
  <img src="https://img.shields.io/badge/Free%20for%20Everyone-16a34a?style=for-the-badge" alt="Free for Everyone">
</p>

### Help Keep LLM Runner Free for Everyone

LLM Runner is **free for everyone to use**.

If you find LLM Runner useful and would like to support CPU BYTE, donations help fund:

* Development
* Performance optimization
* Maintenance
* Hardware testing
* Android compatibility work
* Large-model experimentation
* Future improvements

Your support helps us continue developing and maintaining **LLM Runner as a free-to-use local AI application**.

<p align="center">
  <a href="https://cpubyte-donate.blogspot.com/">
    <img src="https://img.shields.io/badge/%E2%9D%A4%EF%B8%8F%20DONATE-Support%20CPU%20BYTE-16a34a?style=for-the-badge&logoColor=white" alt="Donate / Support CPU BYTE">
  </a>
</p>

<p align="center">
  <sub>Donations are optional. LLM Runner remains free for everyone.</sub>
</p>

---

## How It Works on Android

Running multi-gigabyte neural networks on mobile platforms presents severe memory and process lifecycle constraints. LLM Runner solves these through an integrated native bridge, zero-copy file mapping, storage-backed memory mapping, and an active hardware monitoring framework.

### 1. Storage Access Framework and Zero-Copy Loading

Android sandboxes file access using the Storage Access Framework (SAF). Instead of copying multi-gigabyte `.gguf` files into internal application storage or loading them into JVM byte arrays, LLM Runner works directly with the user-selected model file:

* The application obtains a persistent file descriptor through `ContentResolver.openFileDescriptor`.
* The file descriptor is detached (`detachFd()`) and passed directly across the JNI bridge to the native C++ engine.
* The native engine uses `mmap` (memory-mapping) to map the file descriptor directly into the process virtual address space.
* Weight tensors are faulted into memory on demand by the Linux kernel directly from physical flash storage.
* File copying into application memory is avoided, significantly reducing unnecessary RAM usage.

This architecture allows model files that are much larger than the available physical RAM to be addressed through storage-backed memory mapping and specialized execution modes.

### 2. Zero Permissions Architecture

LLM Runner requests zero Android runtime permissions for inference:

* No storage permissions required (`READ_EXTERNAL_STORAGE` or `MANAGE_EXTERNAL_STORAGE` are never requested; user-selected files are accessed through individual system URI grants).
* No network access is required for local inference.
* Once a model is available on the device, inference can operate fully disconnected from the internet.

---

## Dynamic RAM Framework and Memory Safety

Android Low Memory Killer (LMK) aggressively terminates applications when system memory becomes critically constrained. LLM Runner implements a multi-tiered memory-management framework designed to keep inference within available hardware limits.

### Safe Inference Budget Calculation

Upon startup and model inspection, the profiler queries the operating system memory subsystem:

```text
Total Device RAM (ActivityManager.MemoryInfo.totalMem)
 - OS and System Reserves (maximum of 1.4 GB or 21% of total RAM)
 - App Runtime Footprint (UI, JVM heap, tokenizer allocations)
 - Safety Margin (8% of total RAM)
 = Safe Inference Budget (capped at 60% of physical RAM)
```

### Real-Time Memory Watchdog

A background polling watchdog interrogates system memory APIs every 1000 milliseconds:

* If remaining available system RAM falls below critical limits (under 300 MB or 6% of total capacity), the native engine aborts generation mid-decode.
* The KV cache and working buffers are safely preserved, reducing the risk of the Android kernel forcefully terminating the process.
* Memory usage is continuously monitored while the model is generating.

---

## Model Execution Modes

LLM Runner does not enforce a rigid parameter-count cap. Instead, it provides four operational modes that adapt to the relationship between model size, available RAM, storage performance, and the selected context length.

| Mode                       | Residency Strategy                            | Best Suited For                                                                                            |
| :------------------------- | :-------------------------------------------- | :--------------------------------------------------------------------------------------------------------- |
| **Standard**               | Full pre-population (`MAP_POPULATE`) into RAM | Models whose weights and context comfortably fit inside the safe memory budget. Maximum tokens per second. |
| **Low Memory**             | Demand-paged mapping (`CPUBYTE_MMAP_LAZY`)    | Models near the memory limit. The operating system evicts clean weight pages under memory pressure.        |
| **Layer-by-Layer Offload** | Synchronous layer streaming                   | Dense models larger than physical RAM, where model layers are streamed from storage as required.           |
| **Flash-MoE**              | Active expert streaming via demand paging     | Mixture-of-Experts models where only a subset of experts are active for each token.                        |

---

## Running Multi-Billion Parameter Models Beyond Device RAM

LLM Runner is designed around the principle that **model storage capacity and model RAM residency do not have to be the same thing**.

A GGUF model can occupy many gigabytes on internal storage while only the currently required working data is resident in RAM.

The achievable performance depends heavily on:

* Device RAM
* UFS storage generation and sustained read speed
* CPU architecture and clock speed
* Model quantization
* Context length
* Model architecture
* KV-cache size
* Number of layers
* Offloading configuration
* Android memory pressure

Very large models may therefore be technically executable on devices that cannot hold the complete model in RAM, but generation speed can become extremely low when storage bandwidth becomes the primary bottleneck.

---

## Extremely Large Models on Android

### 127B-Class Models

LLM Runner's storage-backed architecture is designed to make experimentation with extremely large GGUF models possible on Android hardware.

For example, a **127B-class model** can be stored on a device with sufficient available storage and accessed through memory mapping and offloading mechanisms rather than requiring the entire model file to be simultaneously resident in RAM.

The important distinction is:

```text
Model File Size
      ≠
Required Resident RAM
```

With demand paging and layer-by-layer execution, the application can work with a portion of the model at a time.

A 127B-class model is therefore an example of an **extreme large-model workload** rather than a claim that every Android phone can run such a model at practical conversational speed.

On devices with limited RAM, the expected bottleneck for extremely large dense models is generally storage I/O and CPU computation rather than simply the ability to open the model file.

### Example

```text
Android Phone
      │
      ├── Internal UFS Storage
      │       └── Large GGUF Model
      │              │
      │              ▼
      │       Memory-Mapped Weights
      │              │
      │              ▼
      ├── Available RAM
      │       ├── Active Model Layers
      │       ├── KV Cache
      │       └── Compute Buffers
      │
      ▼
   CPU Inference
```

This architecture makes it possible to explore models substantially larger than the phone's physical RAM, subject to storage capacity, Android memory limits, model architecture, quantization, and practical inference performance.

---

## Flash-MoE: How Mixture-of-Experts Streaming Works

Mixture-of-Experts (MoE) models can contain very high total parameter counts, such as 16B, 8x7B, 8x22B, or larger configurations, while activating only a subset of their experts for each generated token.

LLM Runner implements Flash-MoE streaming inspired by storage-backed "LLM in a Flash" methodologies:

1. **Structural Separation:** The base network, attention layers, shared components, router gates, and KV cache remain resident in memory where possible.
2. **On-Demand Expert Access:** Routed expert weights remain backed by fast internal flash storage.
3. **Kernel Page Cache Management:** As tokens are generated, the router selects the active experts and the required weight pages can be brought into memory.
4. **Inactive Weight Eviction:** Clean weight pages can be reclaimed by the operating system when memory pressure increases.
5. **Reduced Working Set:** Only the active working set needs to be resident at a particular point in execution.

This approach can make very large MoE models significantly more practical to experiment with on memory-constrained devices compared with attempting to load every parameter into RAM simultaneously.

---

## Layer-by-Layer Offload for Dense Models

Dense models are more challenging because essentially all model weights participate in every token calculation.

For dense architectures that exceed physical RAM, LLM Runner can use layer-by-layer storage-backed execution:

* A sequential pre-read pass streams weights through the evictable kernel page cache.
* Anonymous working memory such as the KV cache and compute buffers is dynamically constrained.
* Required weight pages are brought into memory as execution progresses.
* Previously used clean weight pages can be reclaimed.
* Generation can continue without requiring the complete model to remain resident in RAM.

The trade-off is performance.

When the model is significantly larger than available RAM, generation may become primarily limited by:

```text
Storage Read Bandwidth
        +
CPU Compute
        +
Memory Pressure
        +
Model Architecture
```

Consequently, extremely large dense models may run at very low tokens-per-second on mobile hardware even when the execution architecture allows them to operate.

---

## Supported Architectures

The native engine supports a comprehensive spectrum of quantized architectures in the GGUF specification, including:

### Dense Families

* Llama (1, 2, 3, 3.1, 3.2)
* Qwen
* Qwen2
* Qwen2.5
* Gemma (1, 2, 3)
* Phi (Phi-2, Phi-3, Phi-3.5)
* Mistral
* StarCoder
* Command-R
* Falcon
* InternLM
* MiniCPM
* And others

### MoE Families

* Mixtral (8x7B, 8x22B)
* Qwen2-MoE
* Qwen1.5-MoE
* DeepSeek-MoE
* DeepSeek-V2
* DeepSeek-V3
* AMD Instella-MoE
* Grok
* DBRX
* Granite-MoE

> Compatibility depends on the GGUF format, architecture implementation, quantization type, available device resources, and the capabilities of the native backend.

---

## Additional Features

* **Built-in Model Hub:** Search and download verified GGUF models directly from Hugging Face with automatic parameter sizing and compatibility scoring.
* **Reasoning / Thinking Mode Controls:** Native support for thinking tags (`<think>...</think>`) found in reasoning models such as DeepSeek-R1 and QwQ, including explicit hardware toggles to enable or suppress reasoning output to save tokens and inference time.
* **Hardware-Adaptive Context Fitting:** Context lengths automatically scale to fit available compute budgets, helping prevent excessive anonymous buffer allocation.
* **Memory-Aware Model Analysis:** Estimates model requirements and compares them with the available device memory before loading.
* **Multiple Execution Modes:** Choose between standard RAM execution, low-memory demand paging, layer-by-layer offload, and Flash-MoE execution.
* **Offline Inference:** Once the model is stored locally, inference does not require an internet connection.
* **Monochrome Interface:** Material 3 dark interface optimized for OLED displays and low-power usage.

---

## Installation and Quick Start

### 1. Download

Download the latest APK from the official Releases page:

<p align="center">
  <a href="https://github.com/cpubyteofficial/LLM-Runner-Android/releases/latest">
    <img src="https://img.shields.io/badge/Download%20APK-2ea44f?style=for-the-badge&logo=android&logoColor=white" alt="Download APK">
  </a>
</p>

### 2. Install

Install the APK on your Android device.

If Android displays a security prompt, enable installation from unknown sources for the application or file manager you are using.

### 3. Launch

Open **LLM Runner**.

### 4. Choose a Model

You can either:

* **Load your model:** Select any existing `.gguf` file stored on your phone.
* **Download model:** Use the built-in Hub to download recommended models such as Qwen2.5-0.5B, Qwen3-0.6B, or Llama-3.2-1B directly to your device.

### 5. Analyze

Review the **Model Analysis** screen to see your device memory budget, estimated working set, and recommended loading mode.

### 6. Select Execution Mode

Choose the appropriate execution mode based on your device and model.

### 7. Start Inference

Tap **Load Model** and begin local inference.

---

## Recommended Model Sizes

The best model size depends on the device's RAM, storage, CPU, quantization, and desired context length.

| Device RAM                     | Typical Starting Point | Larger Models / Advanced Usage                                                                                     |
| :----------------------------- | :--------------------- | :----------------------------------------------------------------------------------------------------------------- |
| **4 GB**                       | 0.5B–2B                | Small quantized models with low context                                                                            |
| **6 GB**                       | 1B–3B                  | 4B-class models with careful settings                                                                              |
| **8 GB**                       | 2B–7B                  | Larger models using low-memory modes                                                                               |
| **12 GB**                      | 3B–8B+                 | Larger models and selected MoE workloads                                                                           |
| **16 GB**                      | 7B–14B+                | Larger models with offloading                                                                                      |
| **24 GB+**                     | 14B–30B+               | Large quantized and MoE workloads                                                                                  |
| **Very Large Storage Systems** | —                      | Experimental workloads including 70B, 100B+ and 127B-class GGUF models, subject to extreme performance constraints |

> These are general examples rather than fixed hardware limits. Actual compatibility and performance can vary substantially between devices and GGUF quantizations.

---

## Performance Considerations

LLM Runner is designed to prioritize **local execution and memory efficiency**, not to make every large model run at desktop-class speed.

For the best performance:

* Use fast UFS storage.
* Keep sufficient free internal storage.
* Use an appropriate GGUF quantization.
* Keep context length within the available memory budget.
* Close unnecessary background applications.
* Use **Standard mode** when the model comfortably fits in RAM.
* Use **Low Memory mode** when the model is close to the memory limit.
* Use **Layer-by-Layer Offload** for oversized dense models.
* Use **Flash-MoE** for supported MoE architectures.
* Avoid extremely large models when practical generation speed is required.

---

## Privacy

LLM Runner is designed for local inference.

When using a locally stored GGUF model:

* Model inference occurs on the Android device.
* Prompts can remain on the device.
* Generated responses can remain on the device.
* No cloud inference service is required.
* Internet connectivity is not required for local model execution.

The built-in Model Hub requires network connectivity when searching for or downloading models. After a model has been downloaded, it can be used for offline inference.

---

## Free for Everyone

LLM Runner is provided **free for everyone to use**.

There is no requirement to donate in order to use the application.

If you enjoy the project and want to help CPU BYTE continue development, optimization, maintenance, and hardware testing, you can optionally support the project.

<p align="center">
  <a href="https://cpubyte-donate.blogspot.com/">
    <img src="https://img.shields.io/badge/%E2%9D%A4%EF%B8%8F%20DONATE-Help%20Support%20Development-16a34a?style=for-the-badge&logoColor=white" alt="Donate">
  </a>
</p>

<p align="center">
  <strong>Donations are completely optional.</strong><br>
  <sub>LLM Runner remains free for everyone.</sub>
</p>

---

## Project

# LLM Runner by CPU BYTE

**Built for local AI inference on Android.**

<p align="center">

`ARM64` • `GGUF` • `OFFLINE` • `STORAGE-AWARE` • `MEMORY-AWARE`

</p>

---

<p align="center">
  <strong>CPU BYTE</strong><br>
  Local AI. Private AI. Your Hardware.
</p>

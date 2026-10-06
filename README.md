# LLM Runner by CPU BYTE

LLM Runner is a fully local, offline GGUF chat application for Android (arm64). It allows users to run open-weight language models directly on their device with zero network traffic, no background servers, and zero Android permissions requested.

---

## Download

Download the latest version directly from the Releases tab:

[Download LLM Runner APK](../../releases/latest)

---

## Features

- 100% Offline: Everything runs directly on your device.
- Zero Permissions: Uses Android Storage Access Framework; does not request storage or network permissions for inference.
- Wide Model Support: Runs quantized GGUF models including Qwen, Llama, Gemma, Phi, and Mistral.
- Memory Efficient: Implements demand-paged memory mapping (mmap) and Flash-MoE streaming to handle large models without crashing.
- Built-in Model Hub: Browse and download public GGUF models directly within the app.
- Reasoning Support: Includes dedicated controls for thinking models (e.g., DeepSeek-R1, QwQ).

---

## System Requirements

- Architecture: arm64-v8a (64-bit ARM)
- Operating System: Android 8.0 (API Level 26) or higher
- Device RAM: 4 GB minimum recommended (depends on the size of the model used)

---

## Installation Instructions

1. Download the APK file from the Download link above.
2. Open the downloaded APK on your Android device.
3. If prompted by Android, enable "Allow installation from unknown sources" for your browser or file manager.
4. Open LLM Runner.
5. Select an existing GGUF model from your storage or use the built-in Hub to download one.

---

## Support and Donations

LLM Runner is provided free of charge. If you find this software helpful and want to support ongoing development, hardware testing, and maintenance, please consider donating:

[Support CPU BYTE - Official Donation Page](https://cpubyte-donate.blogspot.com/)

Your support helps keep the project active. Thank you!

**Running Qwen3-4B on a 6-Year-Old AMD APU? Yes, and It Works Surprisingly Well!**

I just successfully ran **unsloth/Qwen3-4B-Instruct-2507-UD-Q4_K_XL.gguf** on a modest home server with the following specs:

- **CPU**: AMD Ryzen 5 2400G (8) @ 3.600GHz  
- **RAM**: 16 GB (2 × 8 GiB DDR4-2133, unbuffered, unregistered)  
- **iGPU**: Radeon Vega 11 (with 2 GB of VRAM allocated in BIOS)

And the results?  
✅ **Prompt processing**: **25.9 tokens/sec** (24 tokens)  
✅ **Text generation**: **9.76 tokens/sec** (1,264 tokens)

This is honestly **unexpected**—but it turns out that the Vega 11 iGPU, often overlooked for AI workloads, can actually handle **lightweight LLM tasks** like news summarization or simple agent workflows quite effectively—even on hardware from 2018!

### Key Setup Details

- **BIOS**: 2 GB of system RAM allocated to integrated graphics  
- **Debian 12 with kernel (6.1.0-40-amd64) parameters**:  
  ```text
  GRUB_CMDLINE_LINUX_DEFAULT="amdgpu.gttsize=8192"
  ```
- **Runtime**: `llama.cpp` with **Vulkan backend**, running inside a Docker container:  
  [`ghcr.io/mostlygeek/llama-swap:vulkan`](https://github.com/mostlygeek/llama-swap)

### Docker Compose
```yaml
services:
  llama-swap:
    container_name: llama-swap
    image: ghcr.io/mostlygeek/llama-swap:vulkan
    devices:
      - /dev/kfd
      - /dev/dri
    group_add:
      - "video"
    security_opt:
      - seccomp=unconfined
    shm_size: 2g
    environment:
      - AMD_VISIBLE_DEVICES=all
    command: /app/llama-swap -config /app/config.yaml -watch-config
```

### llama-swap Config (`config.yaml`)
```yaml
macros:
  "llama-server-default": |
    /app/llama-server
    --port ${PORT}
    --flash-attn on
    --no-webui

models:
  "qwen3-4b-instruct-2507":
    name: "qwen3-4b-instruct-2507"
    cmd: |
      ${llama-server-default}
      --model /models/Qwen3-4B-Instruct-2507-UD-Q4_K_XL.gguf
      --ctx-size 4096
      --temp 0.7
      --top-k 20
      --top-p 0.8
      --min-p 0.0
      --repeat-penalty 1.05
      --cache-type-k q8_0
      --cache-type-v q8_0
      --jinja
    ttl: 60
```

### Takeaway
You **don’t need a high-end GPU** to experiment with modern 4B-parameter models. With the right optimizations (Vulkan + llama.cpp + proper iGPU tuning), even aging AMD APUs can serve as capable local LLM endpoints for everyday tasks.

If you’ve got an old Ryzen desktop lying around—give it a try! 🚀

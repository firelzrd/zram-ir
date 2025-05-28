# ZRAM Immediate Recompression (ZRAM-IR)

## Overview

ZRAM Immediate Recompression (ZRAM-IR) is a patch for the Linux kernel's ZRAM module that enhances its behavior when multiple compression algorithms (compressors) are available (`CONFIG_ZRAM_MULTI_COMP`). It introduces a logic to immediately try multiple higher-priority compressors during a page write operation and select the first one that achieves a sufficient compression ratio, aiming for a better balance between performance and memory efficiency.

## Why use it?

Modern Linux kernels with `CONFIG_ZRAM_MULTI_COMP` enabled allow ZRAM to potentially utilize multiple compression algorithms. Different compressors have different characteristics: some are very fast but offer lower compression ratios (e.g., LZ4), while others are slower but provide higher compression ratios (e.g., ZSTD).

The optimal compressor can vary depending on the data within each page. However, the standard ZRAM implementation typically relies primarily on the highest-priority compressor for initial writes. If this primary compressor fails to compress the page efficiently, the page might be stored as uncompressed data, wasting physical memory.

ZRAM-IR addresses this by introducing an "immediate recompression" (or rather, multi-trial compression) step during the write operation. Instead of giving up after the primary compressor, it can be configured to try the next available higher-priority compressors until one achieves a satisfactory compression ratio or the configured limit is reached. This allows ZRAM to make a more intelligent, per-page decision at the time of writing.

## Key Features

*   **Immediate Multi-Compressor Trial:** When a page is written, ZRAM-IR can try multiple configured compressors sequentially based on their priority.
*   **Intelligent Selection:** The first compressor that compresses the page below a specific size threshold (typically the size of an uncompressed page, `PAGE_SIZE`) is selected and used for storage.
*   **Configurable Trial Range:** A new sysctl interface (`/proc/sys/vm/zram_recomp_immediate`) allows controlling how many higher-priority compressors are immediately tried during a write.
*   **Improved Efficiency:** By trying multiple options, ZRAM-IR helps reduce the number of pages stored as uncompressed data, leading to a higher overall ZRAM compression ratio and reduced physical RAM usage compared to single-compressor setups where the primary compressor struggles.
*   **Balanced Performance:** Provides a better balance between write/read speed and memory efficiency compared to using only a very fast (lower ratio) or only a very slow (higher ratio) compressor.

## Benefits Comparison

Let's compare ZRAM-IR configured to try, for example, LZ4 (Priority 0) then ZSTD (Priority 1) with single-compressor setups:

*   **Compared to LZ4-only:**
    *   **Pros:** Higher overall compression ratio and lower physical RAM consumption. Pages that LZ4 cannot compress well but ZSTD can will be compressed by ZSTD instead of being stored uncompressed.
    *   **Cons:** Slightly increased write latency and CPU load for pages that LZ4 doesn't compress below the threshold, as the ZSTD trial adds overhead.

*   **Compared to ZSTD-only:**
    *   **Pros:** Significantly faster average write and read speeds. Most pages will be compressed by the faster LZ4. Lower write-time CPU load.
    *   **Cons:** The *initial* overall compression ratio might be slightly lower than ZSTD-only, as LZ4 is used for pages where ZSTD might have achieved a higher ratio.

*   **Overall with ZRAM-IR (e.g., LZ4 -> ZSTD):** Achieves a compromise that leverages the speed of LZ4 for easily compressible data while utilizing the better compression of ZSTD for data that challenges LZ4. This often provides a better overall system responsiveness and memory utilization than using either compressor exclusively.

## Complementary to Background Recompression

ZRAM-IR optimizes the *initial write* process. It complements the Linux kernel's existing background recompression feature. While ZRAM-IR selects the best compressor *at the time of write* within a configured immediate trial range, background recompression can be used later during idle times to migrate pages from lower-compression algorithms (like LZ4) to higher-compression ones (like ZSTD) if further optimization is desired, without impacting interactive performance.

## Requirements

*   Linux kernel with ZRAM support.
*   `CONFIG_ZRAM_MULTI_COMP` kernel configuration option enabled.
*   Multiple compression algorithms compiled into the kernel or as modules.

## Usage

1.  **Apply the Patch:** Apply the `0001-linux6.15.y-zram-ir-1.2.patch` (or a version compatible with your kernel) to your Linux kernel source code.
2.  **Compile and Install:** Compile and install the kernel and ZRAM module with the patch applied and `CONFIG_ZRAM_MULTI_COMP` enabled.
3.  **Load ZRAM Module:** Load the ZRAM module (`modprobe zram`). Ensure your desired multiple compressors are available.
4.  **Configure Compressors:** Configure the compressors for your ZRAM device(s). The specific method depends on your kernel version and ZRAM configuration tools, but typically involves setting the `comp_algorithm` sysfs attribute. Ensure compressors are assigned priorities as needed for ZRAM-IR's trial order (lower priority number = higher priority).
5.  **Adjust Immediate Trial Range:** Control the number of compressors ZRAM-IR immediately tries using the `/proc/sys/vm/zram_recomp_immediate` sysctl parameter.

[![ko-fi](https://ko-fi.com/img/githubbutton_sm.svg)](https://ko-fi.com/Y8Y5NHO2I)

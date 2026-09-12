---
title: Volatile unlock status
description: Evidence and limits for the public CMP100-210 runtime Tensor and PCIe Gen2 recovery path.
---

# Volatile unlock status

This page records a tested recovery path for **NVIDIA CMP100-210 (`10de:1d84`, GV100)**. The maintained implementation is the public [CmpUnlocker-100-210](https://github.com/Brazzo978/CmpUnlocker-100-210/tree/1664127f4a0222206fd7211c964b8dbed55b81eb) project, licensed separately under GPL-2.0. This documentation links to it; it does not copy its code, firmware inputs, private laboratory material, or privileged payloads.

The intervention is runtime-only. It does not flash a VBIOS or program eFuses, and a reset or power cycle returns the tested boards to their stock state. Use the public project's exact hardware, firmware, driver, ownership, and recovery-console checks before considering its one-shot procedures.

!!! warning "Evidence boundary"

    Results below are split into direct observations, controlled measurements, and external reports. A reported identity, a register readback, or an application speed number alone is not treated as proof that another board, driver, or topology behaves the same way.

## Directly observed state

On 2026-09-12, two `10de:1d84` cards both exposed **68 SMs**, or **544 Tensor Cores** by the Volta `68 x 8` mapping. Their active-GPC inventories were:

| GPU | GPC TPC counts | Observed SM total |
| --- | --- | ---: |
| GPU 0 | `[7, 7, 7, 7, 6]` across five GPCs | 68 |
| GPU 1 | `[6, 6, 6, 5, 6, 5]` across six GPCs | 68 |

`CTRLTPC[0x21838 + 4*gpc]` was zero for every GPC. `STATUS[0x21c38 + 4*gpc]` exclusions matched the reported `0x21760 + 4*gpc` values. These reads provide no evidence that extra SMs became available: this work restores compute throughput on the already visible 68 SMs; it does **not** claim an SM-count unlock.

## Validated run: 2026-09-12

This is a separate, dated after-unlock run on two 68-SM `sm_70` cards: PyTorch `2.6.0+cu124`, CUDA `12.4`, and NVIDIA driver `550.163.01`. The cards were at 877 MHz HBM and physical Gen2 x1. It does not replace the earlier public control measurements below.

| Measurement | GPU 0 | GPU 1 | Validation and limit |
| --- | ---: | ---: | --- |
| FP16 GEMM, `N=8192`, 8 warmups, 15 repeats | 73.8018 TFLOPS median | 75.6263 TFLOPS median | Finite-result check and a CPU double-precision full-K reference on an `8 x 8` output block passed. |
| FP64 GEMM, `N=4096`, 8 warmups, 15 repeats | 5.61980 TFLOPS median | 5.61886 TFLOPS median | Finite-result check and a CPU double-precision full-K reference on an `8 x 8` output block passed. This is **after-only** evidence; no new local controlled FP64 before/after claim is made. |
| Pinned 32 MiB host to device | 416.45 MB/s | 416.98 MB/s | Consistent with the independently checked Gen2 x1 link. |
| Pinned 32 MiB device to host | 418.68 MB/s | 418.86 MB/s | Consistent with the independently checked Gen2 x1 link. |

## Controlled runtime results

The public implementation uses a signed Nouveau ACR handoff and verifies a runtime transition at `0x409664` from the stock `0x999` value to `0x888` before returning the device to NVIDIA. The observed effect is restoration of the compute path. Calling that field a "compute-pipe unlock" is a useful working description, but its complete hardware semantics remain tentative.

| Area | Evidence | Scope and limit |
| --- | --- | --- |
| FP16 Tensor | Two cards measured median **74.179** and **75.040 TFLOPS** in an `8192 x 8192` FP16 GEMM with PyTorch 2.6 / CUDA 12.4. | Focused GEMM evidence, not an application benchmark or persistence claim. [Parameters and captured result](https://github.com/Brazzo978/CmpUnlocker-100-210/blob/1664127f4a0222206fd7211c964b8dbed55b81eb/docs/TENSOR-RESULTS.md). |
| PCIe Gen2 | Two physical x1 links reached Gen2; the public control/result was about **207/209 MB/s** stock and **417/419 MB/s** after the Gen2 change for 32 MiB pinned transfers. | Root port and endpoint state were checked. This does **not** establish Gen3 or wider links. [PCIe evidence](https://github.com/Brazzo978/CmpUnlocker-100-210/blob/1664127f4a0222206fd7211c964b8dbed55b81eb/docs/PCIE-RESULTS.md). |
| PCIe Gen3 and width | Neither Gen3 nor x16 has been achieved on these cards. | A Gen2 x1 result must not be read as a Gen3 or x16 result. |

No instruction here suggests a speculative zero write or a write intended to expose additional SMs. The published implementation has its own fail-closed device, firmware, hash, module, driver, and readback checks; its [disclosure boundary](https://github.com/Brazzo978/CmpUnlocker-100-210/blob/1664127f4a0222206fd7211c964b8dbed55b81eb/DISCLOSURE.md) deliberately keeps broader experimentation private.

## Integration findings

Three reported defects produced local fixes validated during the 2026-09-12 run. They are recorded here as test findings; they do not assert that every public revision of the linked runtime already contains each fix.

- The base `nvidia` module must remain loaded. `modprobe -r` on an unused child such as `nvidia_uvm`, `nvidia_drm`, or `nvidia_modeset` can recursively unload its parent `nvidia`; the local correction uses `rmmod` only on those explicitly named child modules. The dedicated hook was already removed separately with `rmmod`.
- Some distributions block Nouveau through an alias policy. In the affected local test, `modprobe -C /dev/null nouveau` supplied a configuration-free fallback before direct per-device binding. It changes no persistent module policy and is not evidence that every distribution is supported.
- After an unbind, BAR0/MMIO reads can return `FFFFFFFF` until PCI memory decoding is restored. The observed recovery was `setpci ... COMMAND=0002:0002`, which enables the memory-decode bit before the BAR0 readback. This is a recovery observation for the affected test state, not a general configuration recipe.

## Reports kept outside the controlled table

An independent Ubuntu tester reported nine stock `1d84` cards plus one hardware-modified card under driver `575.57.08` and kernel `6.8.0-139`. The tester said the public runtime mechanism worked without modification; reported FP64 throughput was **4.43 TFLOPS per card** (about 40 TFLOPS across nine cards) versus a **0.443 TFLOPS** reference that was not a controlled same-card baseline. FP32 was reported unchanged. These results are useful leads, not a replacement for an A/B capture on the same board.

The same tester reported a single-card llama.cpp prefill change of **358 to 2,109 tok/s** (5.89x). In a distinct tuned six-GPU workload, the reported Gen2 gain was **+32.5%**; the compute unlock was reported as **+4.9%** on the same workload. Model, binary, prompt, device split, and measurement controls differ from this site's accepted benchmarks, so none of these figures are generalized here. A separate `0.443 -> 6.876` FP64 figure attributed to [duggasco](https://github.com/duggasco) has not been independently reproduced by this project.

## Rejected strap and VBIOS inference

A newer strap-plus-VBIOS compatibility claim was retracted. The stated eligible mappings are `1df4 -> 1db4` for V100 and `1dc1 -> 1d81` for Titan V; **`1d84` is not listed as eligible**. This correction neither invalidates the volatile software result above nor proves that every additional SM is irrecoverable. It does rule out presenting a `1d84` strap/VBIOS conversion as an established path.

## Attribution and licence boundary

- [Josephur/llama.cmp-v](https://github.com/Josephur/llama.cmp-v) supplies the original site, documentation structure, and comparison methodology preserved by this fork.
- [Brazzo978/CmpUnlocker-100-210](https://github.com/Brazzo978/CmpUnlocker-100-210) supplies the public, GPL-2.0 runtime implementation and its published evidence; its source and licence remain separate from the MIT-licensed llama.cpp fork.
- [amoghmunikote/cmpunlocker](https://github.com/amoghmunikote/cmpunlocker) concerns the GA100-based CMP 170HX. It is credited here only for that public work and is not evidence for the GV100/CMP100-210 claims on this page.
- The duggasco figure above remains explicitly unverified here; attribution records its reported provenance, not an endorsement of the result.

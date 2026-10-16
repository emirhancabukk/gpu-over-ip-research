# Phase 0 — Cloud Provider Priority Changed: Azure (2026-08-27)

## Decision
On AWS, the GPU instance quota (`Running On-Demand G and VT instances`) was still awaiting
approval and blocking progress. It turned out the user had $200 of free credit on Azure:
not real money, and usable immediately without waiting for a quota. The primary cloud
target was switched to **Azure**. AWS is kept waiting in parallel in the background (if the
quota is approved, it will be used for comparison/verification, with a $30 cap).

## Equivalent instance
AWS g5.xlarge (1x NVIDIA A10G, full GPU) ≈ Azure **NV36ads_A10_v5** (1x NVIDIA A10, full
GPU — Azure partitions the GPU in this series: NV6ads/NV12ads/NV18ads are partial GPU
slices, NV36ads gives the full GPU). Note: NV36ads comes with far more vCPU/RAM (36 vCPU);
in our architecture (the remote side only runs the host render process) this is
unnecessary, but it does not matter because the credit is free.

## Region
The NVadsA10 v5 series is available in many EU regions, including Germany West Central,
West Europe (Netherlands) and Poland Central. **Germany West Central** is recommended:
geographically the same as / very close to our AWS Frankfurt measurement, so the two can be
compared.

## Spending cap
The autonomous loop may spend up to a total of **$150** on Azure without asking the user
(the user approved; more relaxed than the $30 cap on AWS because the credit is free). The
instance will be stopped at the end of every experiment and not left idle.

## Next step
The user will create the VM from the Azure Portal themselves (as we did on AWS) and give me
the public IP + SSH access (the path to the private key file).

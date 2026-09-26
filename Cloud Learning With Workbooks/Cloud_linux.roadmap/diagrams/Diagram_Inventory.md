# Linux — Diagram Inventory

> 19 diagrams. One row per diagram; update this file when a diagram is added or moved (Diagram Standard §16.1).
> Sources are in [`svg/`](svg), the images the workbooks link to are in [`png/`](png). Turkish and English workbooks share the same image.

| File | Figure | Section | Type | Height |
|---|---|---|---|---|
| [`lx-0-01-os-layers`](png/lx-0-01-os-layers.png) | 0.1 | 0.1.2 Why can't programs access the hardware directly? | Layers | 900 |
| [`lx-0-02-syscall-path`](png/lx-0-02-syscall-path.png) | 0.2 | 0.2.2 Syscall — the only door between the two worlds | Flow / cycle | 900 |
| [`lx-0-03-everything-is-a-file`](png/lx-0-03-everything-is-a-file.png) | 0.3 | 0.3.3 /sys — the device and driver tree | Block | 900 |
| [`lx-1-01-command-lookup`](png/lx-1-01-command-lookup.png) | 1.1 | 1.1.2 A command = a program call: how does the shell find a command? | Flow / cycle | 900 |
| [`lx-1-02-fhs-map`](png/lx-1-02-fhs-map.png) | 1.2 | 1.2.1 One tree, deliberate directories | Tree | 900 |
| [`lx-1-03-pipe-fds`](png/lx-1-03-pipe-fds.png) | 1.3 | 1.4.2 Redirection and pipes | Block | 900 |
| [`lx-2-01-mode-bits`](png/lx-2-01-mode-bits.png) | 2.1 | 2.2.1 rwx across owner, group, other | Structure | 900 |
| [`lx-2-02-permission-check`](png/lx-2-02-permission-check.png) | 2.2 | 2.2.1 Which triple does the kernel use | Decision tree | 1200 |
| [`lx-2-03-ssh-key-journey`](png/lx-2-03-ssh-key-journey.png) | 2.3 | 2.5.1 EC2 default users and the SSH key journey | Flow / cycle | 900 |
| [`lx-3-01-container-anatomy`](png/lx-3-01-container-anatomy.png) | 3.1 | 3.6.4 Container = process + namespace + cgroup | Comparison | 900 |
| [`lx-4-01-free-memory-anatomy`](png/lx-4-01-free-memory-anatomy.png) | 4.1 | 4.2 Page cache — the "free RAM" fallacy | Comparison | 900 |
| [`lx-5-01-boot-chain`](png/lx-5-01-boot-chain.png) | 5.1 | 5.1 Boot chain — from power to PID 1 | Flow | 900 |
| [`lx-6-01-storage-stack`](png/lx-6-01-storage-stack.png) | 6.1 | 6.6 Cloud storage flow — EBS to a persistent mount | Flow | 900 |
| [`lx-7-01-connection-gates`](png/lx-7-01-connection-gates.png) | 7.1 | 7.4 Connection gates — the path a request must pass | Flow | 900 |
| [`lx-8-01-script-to-service`](png/lx-8-01-script-to-service.png) | 8.1 | 8.2 From a script to a managed service | Comparison | 900 |
| [`lx-9-01-defense-in-depth`](png/lx-9-01-defense-in-depth.png) | 9.1 | 9.1 Defense in depth | Layered model (nested) | 900 |
| [`lx-10-01-idempotency`](png/lx-10-01-idempotency.png) | 10.1 | 10.4 Idempotency | Comparison | 900 |
| [`lx-11-01-debugging-layers`](png/lx-11-01-debugging-layers.png) | 11.1 | 11.2 Layer-by-layer debugging methodology | Flow (decision / layered narrowing) | 900 |
| [`lx-12-01-ami-to-production`](png/lx-12-01-ami-to-production.png) | 12.1 | 12 — the empty-AMI-to-production journey | Flow (pipeline with foundation mapping) | 900 |

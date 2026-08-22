# Cloud Engineer — Foundation Roadmaps (EN)

This repository contains three **foundational** learning roadmaps for someone who
wants to become a **Cloud Engineer**, built on **Socratic (constructive Q&A)
mentorship**. The goal is not to memorize commands; it is to see the system
**beneath the cloud** (hardware, operating system, network) deeply enough to
justify an architecture decision and hunt down failures.

> This trio is the **foundation layer** of the journey — roughly the first third
> of the whole path. Once finished, continue with **application-layer** roadmaps
> such as AWS core services, IaC (Terraform), container orchestration, CI/CD, and
> architecture patterns (see "Next step").

---

## Files

| File | Topic | Phases | What it gives you |
|---|---|---|---|
| `Cloud_hardware_roadmap_en.md` | The physical layer of computing (CPU, memory, storage, buses, network hardware, virtualization) | 0–7 | "Justify instance selection and bottlenecks **with physics**" |
| `Cloud_linux_roadmap_en.md` | Linux / operating system (shell, permissions, processes, memory, boot/systemd, storage, networking-OS, hardening, automation, troubleshooting) | 0–12 | "**Read** a running server and **hunt down** the failure" |
| `Cloud_network_fundamentals_roadmap_en.md` | Network fundamentals (OSI/encapsulation, addressing, subnet/CIDR, L2/L3, transport, DNS, NAT, HTTP/TLS, firewall) | 0–11 | "How does data flow, **why did the packet drop**" |

The three files bound each other cleanly (overlap is minimal): hardware takes the
network *hardware*, the network map takes the network *protocol*; Linux defers
protocol theory to the network map.

---

## How to use them

These files are **not self-read articles**; they are **instruction files handed
to an AI mentor**. Each file opens with a **"Mentor Startup Instructions — for
the AI reading this file"** section; the mentor adopts these rules, and you (the
learner) set the direction.

### 1. Give the file to an AI mentor
Upload/paste an entire file into an assistant like Claude and say, "this is my
roadmap, adopt the mentorship rules." Work with **one** map at a time.

### 2. You pin the position
You — not the mentor — manage the progress:
- **"We're on Phase X"** → start from the beginning of that phase
- **"We're on Phase X.Y"** → continue uninterrupted from that sub-item

### 3. Follow the Socratic rhythm
For each concept: the mentor **asks a guiding question** → you **guess** → if
correct, the mentor **places the term and defines it** → if wrong, it **corrects
directly** → you build on top of it together. You don't move to the next concept
before the current one is complete.

### 4. Read the depth labels
The label next to each heading tells you **how much** you need to learn a topic:

| Label | Meaning |
|---|---|
| `[concept]` | Being able to explain intuitively / give the definition is enough |
| `[mechanism]` | You must be able to explain step by step how it works |
| `[application]` | Run it by hand on your own machine & see it / relate it to an architecture decision |
| `[skip]` | Just know the name, don't go deep |

### 5. Use the Labs and checkpoints (optional)
- **Lab / command suggestions**: after `[application]` topics, the mentor
  suggests the relevant commands. If you run them and bring back the output, you
  read it together. If you don't want to, you're not forced — the decision is
  yours.
- **✅ Checkpoint quizzes / phase closings**: at checkpoints, the mentor
  automatically suggests a quiz. If you want to defer, it's deferred.
- ⚠️ Some Labs permanently change the system (mount, fstab, services). On
  destructive steps, the mentor warns you first.

### 6. Useful cues
- **"just tell me directly"** → drop the Socratic question, explain outright.
- The **"When it breaks"** angle is in every phase: "how does this show up in the
  system when it fails?" — build your failure instinct from here.

---

## Suggested order

The maps are independent, but for the Cloud Engineer goal the suggested flow is:

1. **Network Phases 0–2** (subnet/CIDR) + **Linux Phases 0–4** → start in
   parallel; these are daily job skills.
2. Interleave **hardware as the "deep why" track**. Hardware Phase 0 (logic
   gates, flip-flops) can be passed quickly at the `[concept]` level.
3. Advance the remaining phases and settle the transition to AWS by doing all
   three maps' **final phase (Bridge to the Cloud)** together.

---

## What these maps cover / don't cover

**Cover (deeply):** hardware/compute physics, Linux/OS, network fundamentals
(L1–L7), Bash scripting, troubleshooting methodology, virtualization/container
*concepts*.

**Don't cover (deliberately — the next layer):**
- AWS core services at operational depth (EC2/VPC/S3/IAM/RDS…)
- IaC — Terraform / CloudFormation / CDK
- Container orchestration — Docker in depth, Kubernetes/ECS/EKS
- CI/CD, Git, Python (boto3) automation
- Databases, observability at scale, architecture patterns, cost/FinOps

---

## Next step (suggestion)

Mapping the missing application layer in the same Socratic + "When it breaks" +
cloud-bridge format:

```
4. AWS Core Services (SAA-focused)     ← most urgent
5. IaC / Terraform
6. Docker → Kubernetes / ECS-EKS
7. CI/CD + Git + Python automation
8. Architecture patterns + observability + FinOps  ← capstone
```

---

## Notes

- These roadmaps are the English editions of the Turkish originals (`*_temel_tr.md`).
  The one deliberate adaptation: the Turkish files gave each English/technical
  term a phonetic pronunciation for Turkish speakers on first use; the English
  editions replace those with plain-English definitions, keeping the
  "define on first use" pedagogy intact.
- Cloud examples are **AWS-centric**; terminology differs on GCP/Azure.
- The files are not personal; the learner is treated generically, so anyone can
  work through them.

---

*Prepared by: Denis Ergöçmen, 2026 — English edition*

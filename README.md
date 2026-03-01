# GPUForeman

**A multi-GPU pipeline scheduler for generative AI workloads.**

> ⚠️ **Prior Art Disclosure — March 1, 2026**
> This repository exists to establish a public timestamp for the architectural concepts described below. No implementation is provided at this time.

---

## The Problem

Current multi-GPU generative AI tooling (ComfyUI, Forge, A1111) has no pipeline awareness. GPUs:

- Repeatedly load and unload model weights between tasks
- Sit idle waiting for sequential steps to complete
- Can't dynamically reassign to bottleneck stages
- Have no cross-card coordination

---

## The Concept

One GPU acts as **Controller**. The rest are **Workers**.

```
Controller GPU
├── Maintains a Register Table (Job ID → indexed output chunks)
├── Tracks Worker states: ACTIVE / IDLE / WAIT
├── Caches stage outputs until downstream Worker is ready
└── Dynamically reassigns IDLE Workers to backlogged stages

Worker GPUs
├── Assigned a pipeline role (T2I, I2I, Mesh, Texture...)
├── Run their full batch to completion — never interrupted mid-inference
├── Signal COMPLETE to Controller
└── Enter IDLE state, await reassignment
```

### Example Pipeline

```
Worker 1: Text-to-Image   →  completes batch  →  reassigned to Mesh (bottleneck)
Worker 2: Image-to-Image  →  completes batch  →  awaits next job
Worker 3: Image-to-Mesh   →  running          →  now has 2 workers helping
Worker 4: Mesh Texturing  →  running
Controller: watching queue depth, dispatching indexed outputs, rebalancing
```

Workers are **never interrupted mid-batch**. A Worker runs to completion, signals the Controller, and is then reassigned. This eliminates VRAM thrashing and mid-inference checkpointing complexity.

---

## Register Table

The Controller maintains a simple indexed job register:

```python
register = {
  "job_1435": {
    "stage":        "img2img",
    "status":       "READY",
    "output_index": [1435, 1436, 1444, 1445],
    "consumer":     "mesh_worker",
    "created_at":   "2026-03-01T00:00:00Z"
  }
}
```

When a Worker completes, outputs are cached and indexed. The Controller dispatches the relevant index entries to the next waiting Worker.

---

## Dynamic Reallocation

If mesh generation is the bottleneck:

1. Mesh queue depth exceeds threshold
2. T2I Worker finishes its batch → signals IDLE
3. Controller reassigns it to load the Mesh model
4. Two Workers now process mesh jobs in parallel
5. Backlog clears → Controller rebalances again

No user intervention required.

---

## Implementation Primitives (Not Yet Built)

The building blocks exist:

| Component | Technology |
|---|---|
| Inter-GPU memory sharing | CUDA IPC |
| Job queue & register | Redis or SQLite |
| Worker orchestration | Python asyncio |
| ComfyUI integration | ComfyUI API mode |
| Tensor passing between cards | cudaMemcpy / PyTorch device map |

---

## Prior Art Statement

This repository is published as a good-faith prior art disclosure. The architectural concepts described here are intended to remain freely available to prevent monopolistic patenting by third parties.

The author retains all rights. No license is granted at this time.

**Disclosure date: March 1, 2026**

---

## Full Technical Document

See [`GPU_Pipeline_Scheduler_PriorArt_2026-03-01.docx`](./GPU_Pipeline_Scheduler_PriorArt_2026-03-01.docx) for the full technical design document.

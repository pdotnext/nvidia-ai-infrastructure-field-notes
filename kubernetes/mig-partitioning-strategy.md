## What MIG Is

MIG stands for Multi Instance GPU. MIG provides option to slice the GPU
which is at silicon level isolation. This level isolation is the only isolation where memory error do not cross over
or impact others. MIG is higly recommended for Production environment with strict SLA

## The Two Building Blocks — GI and CI
Every GPU comes with GPU Instance and under GI comes Compute Instance.
When you slice an GPU, it is basically is referred as GI.
1 GPU can be sliced into 7 slices within 80 GB RAM
7g.80gb = 1 GI with 80gb this is full GPU, no MIG -- equivalent to MIG disavled.

CI stands for Compute Instance and it is only subset of GI. So when you create GI, under it there is CI.
This CI instances shared the memory from GI.


## H100 80GB Profile Table

1g.10gb - Slice with 10gb
2g.20gb - Slice with 20gb
3g.40gb - Slice with 40gb
4g.40gb - Slice with 40gb
7g.80gb - Slice with 80gb

## The Two-Constraint Rule
There are two rule which applies to creating slice

- max 80 GB available
- max 7 slices

3g.40gb + 1g.10gb + 2g.20gb = 6 slices, 70gb


## MIG and NVLink — Why Training Jobs Must Not Use MIG

NVLink manages communication between GPUs during allreduce operation.
During this operation GPUs communication with eachother at 900 GB/s
Any reduction in the speed of this link, leads to significant performance loss of AI Training.
And MIG unfortunately not suitable for AI training job.
Because MIG, NVLink is disabled. 
This means, GPU needs to communicate with eachother using PCIe 64 GB/s, which is 12 times less.


## When to Use MIG vs When Not To

Do not use MIG when running multi-GPU AI training. Training jobs require
GPUs to communicate via NVLink during allreduce. MIG disables NVLink,
forcing fallback to PCIe at 64 GB/s — significantly degrading throughput.

Use MIG when:
- Running inference workloads — models are independent, no cross-GPU
  communication needed. Slice to match model memory requirements.
- Running single-GPU fine-tuning — lean GPU requirements, no NVLink needed.
- Multi-tenant production environments with strict SLA — silicon-level
  isolation guarantees one tenant cannot impact another.

Use time-slicing instead of MIG when:
- Dev/test environments with shared access and no SLA requirements —
  simpler configuration, maximises GPU utilisation without partitioning.

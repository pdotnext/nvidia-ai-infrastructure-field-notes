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

- 1g.10gb - Slice with 10gb
- 2g.20gb - Slice with 20gb
- 3g.40gb - Slice with 40gb
- 4g.40gb - Slice with 40gb
- 7g.80gb - Slice with 80gb

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

## Frankfurt Financial Services — Reference Design

### Scenario
We have a budget for two DGX H100 nodes — 16 GPUs total. Three teams need access:
- Team A has 4 data scientists doing model development and fine-tuning during business hours, small models, each fits in 20GB.
- Team B runs fraud detection inference in production — 8 models, 24/7, each model needs 10GB, cannot tolerate any downtime or interference from other teams.
- Team C is our research team — they run large LLM pre-training jobs overnight, 4-GPU jobs, they book the cluster in advance.

### Design Decisions
Let me summarize what we have at hand and then justify my solution

- First we have 16 GPUs at hand and total 16 * 80 = 1280 gb GPU memory at hand.

#### Team A

Data scientist are training fine tuned model which has 20GB requirement. In other words, this requirement can be satisfied with a single GPU when properly sliced
Normally we do not use MIG when multiple GPUs needs to communicate using NVLink.
Here it is not the case. Hence we allocate 2 GPUs and spreading the model across the GPUs.
All 4 cannot fit into single GPU as 7 slice per GPU constraint is violated.

Total requirement = 4 model each with 20gb
4 x 2g.20gb violates the constrain of 7 slices.
Hence
3 x 2g.20gb - GPU0 = 60gb
1 x 2g.20gb - GPU1 = 20gb

This means out 16 GPU - 4 GPU will be reserved for them.
We have now 12 GPU.

#### Team B
Team -B is has inference workload, model memory and latency requirement is deciding factor for MIG.
10 GB model can easily fit when slicing GPU.

Two GPU and allocation summary below following the constraint rule of maximum 80gb or max 7 slices
1g.10gb x 7 = 7 model
1g.10gb x 1 = 1 model,
Total = 8 models spread across 2 GPUs

### Team C
Is pre-training large LLM job, which is reprocessing of the data, however GPUs need to communicate with each other
Since Team A is consuming GPU only during the business hours and during night hours, these GPU could be used.
But we must disable MIG before starting the training. This is significant overhead and tradeoff is not worth esp when there sufficient GPUs available

Hence dedicated 4 GPUs will be allocated.

### GPU Allocation Table

| GPUs | Team | Mode |Allocation | Hours|
|--- |--- |--- | --- | --- |
| GPU0 - 1 | Team -A | MIG | 4 x 2g.20gb across 2 GPUs | Business Hours |
| GPU2 - 5 | Team -C | MIG Disable | 4 dedicated GPUs, NVLink Active | Non-Business Hours |
| GPU6 - 7 | Team -B | MIG Enabled | 1g.10, total 8 slices, across 2 GPUs | 24 x 7 |
| GPU8 -15 | Reserved | --- | Future Growth | ---|

**Total 8 GPUs are available for future growth**

### Operational Consideration — MIG Toggle Overhead
There is another option which can be used. But there is an operational overhead.
In short the steps looks like
- enable MIG
- stop AI training
- drain the nodes
- disable MIG
- start Team-C job
- stop the job
- enable MIG

This is significant overhead for a production environment.
Hence this solution is not scalable and not recommended.

### Customer Summary
Out of 16 GPUs, 8 GPUs are used optimally and in case of Team-B, they have additional slices available for scalability.
Team-A as well get 2 GPUs with 50% more capacity to expand for future.
Team-C gets dedicated 4 GPU. So this solution offers not only provide head rooms for Team-A and Team-B, but also have
additional 8 GPUs for future expansion, thereby reducing any need to buy additional hardware for next 12 months.

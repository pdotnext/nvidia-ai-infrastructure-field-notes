
## Timeslicing vS MIG -- Knowing When to Use Each

In MIG, I learnt that it is silicon level isolation.
It is like planning how to partition harddisk and how many partition.
Because once formatted, we cannot do much without vacating data and restarting again.
So in short, when it comes to MIG, 📝 planing is must.

## What Is Time-Slicing
Timeslicing is more like telling kubernetes nodes, assume you have 4 GPUs
This is done using replicas.
In Timesclicing, GPU is scheduling process based on timeslot.
e.g. GPU schedules process1, then process2, in short in rotation fashion.
This ensure GPU remains busy all the time.
🞋 Goal: Maximum utilization of GPU and hence Business benefits in terms of cost


## What Time-Slicing Cannot Do
But timeslicing cannot protect against
- fault or memory error which will impact all the process running on the GPU.

**Use cases:**
- Best fit for dev and test environment because they do not run on tight SLA
- inference but only where latency tolerance exist.
But real production inference workload with SLA, MIG is must.



## How to Configure Time-Slicing in Kubernetes

create a configMap with metadata name, namespace
then create a config where we disable MIG i.e. migStrategy: None
then under sharing, use timeSlicing, providing the exact replicas we need.

Below is the sample yaml file

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: time-slicing-config
  namespace: gpu-operator
data:
  any: |-
    migStrategy: none # disable MIG
    sharing:
      timeSlicing:
        reources:
          - name: nvidia.com/gpu
            replicas: 4
```
Next create configmap and update clusterpolicy

```bash
kubectl apply timeslicingconfig.yaml
kubectl patch clusterpolicy gpu-cluster-policy \
-n gpu-operator \
--type merge -p '{"spec" : {"devicePlugin" : {"config": {"name": time-slicing-config }}}}'


```

```json
{
  "spec": {
    "devicePlugin": {
      "config": {
        "name": "time-slicing-config"
      }
    }
  }
}
```

## Replica Sizing — The Design Responsibility
Replica selection is a design choice e.g. you have 80 GB GPU,
you have the responsibility to ensure it does not end up on Out of Memory errors (OOM).
e.g. 4 replicas using 20 GB memory, will sure end up in OOM
instead 4 replicas using 15 GB memory, is safe approach

## What Is MPS (Multi-Process Service)
Now lets talk about another GPU Sharing mechanims which is MPS
MPS Stands for Multi process Service

Heavily used in HPC. Because in MPS, one shared CUDA context is created where processes run their job in parallel

## MPS vs Time-Slicing — The Key Distinction
One distinction between MPS (schedules job in parallel) and Timeslicing (schedules job in rotation) .However MPS is like single trusted application creating multiple processes to efficiently use GPU.Like in HPC, single job creates multi process and these process help each other rather than compete.
e.g. Weather forecast simulation start 8 MPI process, so it looks like

- Rank0 takes care on one part of the data
- Rank1 another and so on and so forth
- all ranks sync with each other.

## When MPS Is the Right Choice

Hence MPS is suitable for
- one team with
- single trusted application
- with processes helping each other

In the end, MPS is telling CUDA, let multiple process share the GPU context efficiently

## SA Decision Framework — Three Questions

| Scenario | Recommendation | Why |
|---|---|---|
| Multi-GPU LLM training | Full GPU Allocation | AI workload communicate across GPUs |
| Production inference with SLA | MIG | GPU Memory required for the Inference can be allocated using MIG|
| Dev/test shared cluster | Timeslicing| For Dev, Test workload do not need memory/fault isolation |
| Batch inference, latency-tolerant | Time-slicing | No strict SLA, maximise GPU utilization, lower cost |
| HPC, single team, cooperative ranks | MPS | Single Application, multiple process |

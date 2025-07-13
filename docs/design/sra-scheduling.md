# Scarce Resource Avoidance (SRA)

## Introduction
In a cluster where GPU nodes and CPU nodes are deployed together, there is a risk that a task submitted to the cluster, which does not require GPU resources, may be scheduled to a GPU node. This can lead to inefficient resource utilization, as a job requiring both CPU and GPU resources might be pending due to insufficient CPU resources on the GPU node. This scenario is not ideal, as it can result in underutilization of resources and potential delays in job execution. Therefore, sra is proposed to avoid nodes with critical resources (such as GPUs) for certain CPU jobs and to improve overall resource utilization.

## Policy
At present, the following two policies are supported:
- `retention`: Give each scare resource a weight. Higher weight means more scare. Jobs that don’t need the scare resource will try to stay off nodes that have it, so the scare resource can be retained.
- `proportional`: By specifying 'primary' scare resources (e.g. GPU in deep learning) and preserve the amount of associated 'secondary' resources by a pre-set proportion.

## Solution
### retention
1. In retention policy, arguments `sra.resources` is provided to configure important resources in the cluster.
2. Based on the significance of different resources, `sra.retention.[ResourceName]` can assign varying weights for resource allocation. A higher weight signifies greater importance, and tasks that do not require this resource will, to the extent possible, be scheduled away from nodes with such resources
3. For all tasks, user can set `sra.resources` and `sra.retention.[ResourceName]` field in `resource-strategy-fit` arguments via `volcano-scheduler-configmap` in following format:
      ```yaml
       actions: "enqueue, reclaim, allocate, backfill, preempt"
       tiers:
       - plugins:
           - name: resource-strategy-fit
             arguments:
               sra.policy: retention
               sra.resources: nvidia.com/t4, nvidia.com/a10
               sra.retention.weight: 2
               sra.retention.nvidia.com/t4: 1
               sra.retention.nvidia.com/a10: 1
      ```

4. `retention` policy will affect the score result of node order:
   1. Now, we have tasks:

      | Task Name  | Task request resource                                        |
      |------------|--------------------------------------------------------------|
      | cpu-task-0 | `{cpu: 2, memory: 4Gi}`                                      |
      | gpu-task-0 | `{cpu: 2, memory: 4Gi, nvidia.com/t4: 2}`                    |
      | gpu-task-1 | `{cpu: 2, memory: 4Gi, nvidia.com/t4: 1, nvidia.com/a10: 2}` | 
   
   2. Suppose there are 3 nodes available in cluster:

      | Node Name | Resource capacity on node                                       |
      |-----------|-----------------------------------------------------------------|
      | node1     | `{cpu: 32, memory: 64Gi}`                                       | 
      | node2     | `{cpu: 16, memory: 32Gi, nvidia.com/t4: 10}`                    | 
      | node3     | `{cpu: 16, memory: 32Gi, nvidia.com/t4: 5, nvidia.com/a10: 10}` |

    Through the retention policy we will get the following results:
   
      | Task       | Node  | Score result (retention) | Reason                                                       |
      |------------|-------|--------------------------|--------------------------------------------------------------|
      | cpu-task-0 | node1 | 200                      | node resources meet the task request and no scarce resources |
      | cpu-task-0 | node2 | 100                      | node resources meet the task request and have t4             |
      | cpu-task-0 | node3 | 0                        | node resources meet the task request and have t4, a10        |
      | gpu-task-0 | node1 | 0                        | node resources don't meet the task requirements              |
      | gpu-task-0 | node2 | 100                      | node resources meet the task request and have t4             |
      | gpu-task-0 | node3 | 0                        | node resources meet the task request and have t4, a10        |
      | gpu-task-1 | node1 | 0                        | node resources don't meet the task requirements              |
      | gpu-task-1 | node2 | 0                        | node resources don't meet the task requirements              |
      | gpu-task-1 | node3 | 0                        | node resources meet the task request and have t4, a10        |

### proportional
For more details, please refer to: [proportional design](./proportional.md)
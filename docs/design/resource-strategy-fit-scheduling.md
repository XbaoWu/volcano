# ResourceStrategyFit Plugin

## Summary

The native k8s ResourceStrategyFit plug-in can only adopt one type of strategy for all resources, such as MostRequestedPriority and LeastRequestedPriority. However, in industrial practice, this design is not applicable in some scenarios. For example: in AI scenarios, we usually disperse CPU tasks in CPU machine groups to reduce hot spots. GPU tasks are gathered in GPU machine groups to reduce GPU fragmentation. Therefore, we need to expand a scheduling strategy to meet the needs of this scenario.

## Motivation

- Different resource types can be configured with different aggregation or dispersion strategies, and weights can be used to distinguish priorities

### Goals

- Different types of resources can be configured with different strategies to prioritize them in the form of weights

### Non-Goals

- None.

## Proposal

Extend one plug-in to meet the above needs

- ResourceStrategyFit

## User Story

### Story1
- Users expect different resource allocation strategies to be applied based on resource types. For example, in PyTorch jobs, the master pod (which uses CPU) should be distributed to avoid node hotspots, while worker pods (which use GPU) should be aggregated to minimize resource fragmentation.

## Design Details

### ResourceStrategyFit

config：
```yaml
  actions: "enqueue, allocate, backfill, reclaim, preempt"
  tiers:
  - plugins:
    - name: resource-strategy-fit
      arguments:
        resourceStrategyFitWeight: 10
        resources:
          nvidia.com/gpu:
            type: MostAllocated
            weight: 2
          cpu:
            type: LeastAllocated
            weight: 1
```
config description：

<table>
	<tr>
	    <th>strategy</th>
	    <th>calculation method</th>
	    <th>effect</th>  
	</tr>
	<tr>
	    <td>MostAllocated</td>
	    <td>requested/allocable</td>
	    <td>Aggregated</td>
	</tr>
	<tr>
	    <td>LeastAllocated</td>
	    <td>(allocable-requested)/allocable</td>
	    <td>Dispersed</td>
	</tr>
</table>

node score:
```
finalScoreNode = [(weight1 * resource1) + (weight2 * resource2) + … + (weightN* resourceN)] /(weight1+weight2+ … +weightN)
```

## Alternatives

### Binpack VS ResourceStrategyFit
If you want to use the clustering strategy for all resource types, you can choose the Binpack plugin. If you need to configure different clustering or scattering strategies for different resource types, you can choose the ResourceStrategyFit plugin. ResourceStrategyFit can also achieve the same results as Binpack by adjusting configuration parameters.

## Best Practices
### AI scenario
In some AI scenarios, CPU tasks are usually dispersed into CPU machine groups to reduce hot spots. GPU tasks are gathered in the GPU machine group to reduce GPU fragmentation. At the same time, it is also necessary to avoid the situation that CPU tasks are assigned to GPU nodes, resulting in long-term waiting of GPU tasks due to insufficient CPU or MEM resources of nodes. In this scenario, we can combine **resourceStrategyFit** and **sra policy** to deal with this scenario. The corresponding example configuration is as follows:

```yaml
  actions: "enqueue, allocate, backfill, reclaim, preempt"
  tiers:
    - plugins:
      - name: resource-strategy-fit
        arguments:
          resourceStrategyFitWeight: 10
          resources:
            nvidia.com/gpu:
              type: MostAllocated
              weight: 2
            cpu:
              type: LeastAllocated
              weight: 1
          sra.policy: retention
          sra.resources: nvidia.com/gpu
          sra.retention.weight: 10
          sra.retention.nvidia.com/gpu: 1
```
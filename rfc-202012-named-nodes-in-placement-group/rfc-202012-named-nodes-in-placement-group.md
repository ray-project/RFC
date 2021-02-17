# RFC-202012-named-nodes-in-placement-group

| Status        | Proposed      |
:-------------- |:---------------------------------------------------- |
| **Author(s)** | Hao Zhang, Dacheng Li  |
| **Sponsor**   | Ion Stoica               |
| **Reviewer**  | |
| **Updated**   | YYYY-MM-DD                                           |


## Objective
In many distributed ML applications, for better system performance, we need to precisely place a computation-heavy task on a specific type of node of cluster suitable for computation, whereas a communication-heavy task/actor on a high-bandwidth node.


Ray currently does not **explicitly** support this type of placement semantic. This RFC proposes to use existing Ray functions (e.g., `PlacementGroup`) to produce a "placement" feature, so we can place an actor/task on a designated node that meets the resource condition (e.g. a parameter server on a CPU-only high-bandwidth node), as long as we know the cluster configurations in advance.  

### Non-goals


## User Benefit
With this feature, the users can have finer control when specifying resource bundles. Compared to existing `PACK`, `SPREAD` strategies, the user now can specify resource bundles on designated nodes of the cluster, so to place their actors/tasks on desired nodes with cluster awareness.

## Design Proposal


### Architecture

We propose to add support to the existing placement_group(bundles, strategy, name) so that the user can precisely specify
which node to use for a task in the cluster. In current ray usage, a user can specify a placement_group using:

```python
pg = placement_group([{"CPU": 2}, {"GPU": 2}])
ray.get(pg.ready())
```
After the placement group is ready, the user can specify which bundle to place for a new task using bundle index:

```python
@ray.remote(num_gpus=1)
class GPUActor:
    def __init__(self):
        pass

gpu_actors = [GPUActor.options(
        placement_group=pg,
        placement_group_bundle_index=1) # Index of gpu_bundle is 1.
    .remote() for _ in range(2)]
```

Here the user has no control whether these bundles are placed. An example using our new signature will be:

```python
pg = placement_group([{"CPU": 2}, {"GPU": 2}], nodes = [node_ip_1, node_ip_2])
ray.get(pg.ready())
gpu_actors = [GPUActor.options(
        placement_group=pg,
        placement_group_bundle_index=1) # Index of gpu_bundle is 1.
    .remote() for _ in range(2)]
```

Now the gpu_actors are guaranteed to be scheduled on node_ip_2. This gives the user finer control if they know
for example, which node has higher bandwidth. 

A possible implementation is to utilize custom resources. When the placement_group() is called for the first time,
we iterate through the existing nodes, and give each node a unique resource related to their name. Then, we can
construct custom resources using these unique names to constraint scheduler to only place desired bundles on these 
nodes. A demo code with 1 node is attached here:

```python
nodes = ray.nodes()
id = nodes[0]["NodeID"]
resource_name = id+"_gpu:0"
resource_capacity = 1.0

@ray.remote
def set_resource(name, capacity, node_id):
    node_id_obj = ray.NodeID(ray.utils.hex_to_binary(node_id))
    return ray.worker.global_worker.core_worker.set_resource(
                  resource_name, capacity, node_id_obj)

ray.get(set_resource.remote(resource_name, resource_capacity, id))
print(ray.nodes())
```

We are aware that ray is planning to remove the ray.experimental.set_resource API from the user, so we directly use 
the cpp code in the backend. Upon finish time, this node will be added a resource with name id+"_gpu:0". And we can
use this resource to place the bundle on this node.

### APIs

We modify the current API to accept an additional argument: placement_group(bundle, strategy, nodes), where bundle and 
nodes are both lists, and each element in the bundle will be placed in the corresponding node in nodes. 



### Unsolved Problems


#### Place on a device level
As far as we understand, current Racy cannot understand general GPU types. Thus, we are only able to specify bundles against physical cluster nodes, but not at the granularity of devices, such as requesting a GPU as "the P100 GPU on node A".
This finer grain control might be ideal for distributed ML optimization, e.g., put heavy ML computation on a more powerful 3080 GPU where lighter computation on a 1080 GPU.

#### Working with the supported accelerator types in Ray
The current Ray supports and can recognize several accelerator types, such as "V100", "P100", etc. -- essentially most of the AWS-supported GPU devices. See [ray/util/accelerator/accelerator.py](https://github.com/ray-project/ray/blob/master/python/ray/util/accelerators/accelerators.py). As long as Ray can recognize these devices as supported resources, We might extend this feature to understand these device types and support finer placement over a designated GPU device. We are currently figuring out more details on this.


#### Cons



#### Pros


### Other Considerations


### Performance Implications
None.

### Dependencies

### Best Practices

### Tutorials and Examples


### Compatibility

### User Impact

## Implementation Plan

## Questions and Discussion Topics

# RFC-202011-collective-in-ray

| Status        | Proposed      |
:-------------- |:---------------------------------------------------- |
| **Author(s)** | Hao Zhang, Dacheng Li  |
| **Sponsor**   | Ion Stoica               |
| **Reviewer**  | |
| **Updated**   | YYYY-MM-DD                                           |


## Objective



### Non-goals



## Motivation


## User Benefit



## Design Proposal

### Architecture


### APIs


### Unsolved Problems


### Alternative Design
#### Pros
#### Cons


### Alternative #2: Using a new in-memory Python Object Store
        Instead of using the current ray object store logic, we can also build a new object
store on top of Python.This new object store is always stored in memory, and only to
store inputs for collective operations. An example would be:

```
@ray.remote(num_gpus=1)
class Actor:
    def __init__(self):
        pass
    
    @collective
    def prepare(self):
        buffer = 1
        return buffer

    def compute(self, buffer):
        return buffer * 2

actors = [actor.options(collective={
            "group_name": "177",
            "group_rank": [0,1],
            "world_size": 2,
            "backend": "nccl"}).remote() for i in range(2)]

refs = [a.prepare.remote() for a in actors]
results = ray.collective.allreduce_refs([a.compute.remote(ref) 
                                       for (a, ref) in zip(actors, refs)])
print(ray.get(results))

```

The way that the current codebase interact with the object store at Python level is that
 it calls the submit_actor_task() function during the remote actor method call, which goes 
into ray's system layer logic, and puts this future object into th object store. Instead we 
can require the user to mark a function that stores the inputs of collective operations.
In the above example, we require the user to use the decorator "collective" to identify 
that these variables are going to be allreduced accross processes, so that the backend knows
to store them in an in-memory store. At runtime, when the system discovers that the functions 
are marked, it processes with our new in-memory object store instead of invoking the ray's one. 
Also, to better support these refs usage, we propose an allreduce_refs function in the collective 
module, where the user can directly call allreduce at the driver program. Note that before this 
new function, the user will need to call allreduce at every worker. For example:

```
@ray.remote(num_gpus=1)
class Actor:
    def __init__(self):
        self.buffer
    
    def compute(self):
        return ray.collective.allreduce(self.buffer)

actors = [actor.options(collective={
            "group_name": "177",
            "group_rank": [0,1],
            "world_size": 2,
            "backend": "nccl"}).remote() for i in range(2)]

results = [a.compute.remote() for a in actors]
print(ray.get(results))

By using this new allreduce_ref function, the user can have more flexibility of whether to use
collective operations in the driver programs or not. A possible implementation to this is that: The
backend first validates all the inputs to this allreduce_ref have been annotated as using our python
in-memory object store, then it get the values from our object store and do the compuation, instead of
querying the current ray object store.


```

#### Cons
        (1) Since we submit the task and store the results ourselves, these collective operations are
invisible to ray. Thus we are lack of several ray's features, including fault tolerance at fails,
and garbage collection. 
        (2) It is unclear how ray's scheduler will react to these invisible tasks. I.e., it may think 
these nodes are degenerated and tend to schedule less jobs to these nodes, while the resources at 
these nodes are simply acquired by our collective operations.

#### Pros
    (1) The current in-memory object store in ray uses serialization, which is slow compared to
typical collective operations. In our experience, serialization requires time in ms, while collective
operations also requires time in ms. By providing a new object store that recovers smiliar functionality
without serialization, we can finish collective operations more efficiently.
    (2) It is likely that we need far less development in current c++ code, which shortens development
time.

### Other Considerations


### Performance Implications


### Dependencies


### Engineering Impact


### Platforms and Environments


### Best Practices

### Tutorials and Examples


### Compatibility

### User Impact

## Implementation Plan
We plan to prioritize the implementations of NCCL backends so to unblock some ongoing development in NumS and RayML.

## Detailed Design
### Collective Groups
### Communicators
### Implementations of Collective Primitives



## Questions and Discussion Topics

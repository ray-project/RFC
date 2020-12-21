# RFC-202011-collective-in-ray

| Status        | Proposed      |
:-------------- |:---------------------------------------------------- |
| **Author(s)** | Hao Zhang, Dacheng Li  |
| **Sponsor**   | Ion Stoica               |
| **Reviewer**  | |
| **Updated**   | YYYY-MM-DD                                           |


## Objective

Based on [RFC-20201119-collective-in-ray](https://github.com/ray-project/ray/pull/12174), we have brought CCL libraries such as NCCL into Ray; see the current set of APIs under [python/ray/util/collective](https://github.com/ray-project/ray/tree/master/python/ray/util/collective).

As of now, this set of collective APIs are supposed to work inside the code of an actor/task (and NOT in the driver code) in an imperative way, interacting directly with the contents/arrays to be communicated, similar to most APIs in traditional CCLs (e.g., [torch.distributed](https://pytorch.org/docs/stable/distributed.html)). In this RFC, we expand this set of APIs to work with Ray ObjectRefs. Specifically, we will introduce a set of **more declarative** collective APIs that make use of ObjectRefs as inputs, such as `allreduce(ObjectRefs: List[ObjectRef])`, so users can flexibility use collective in driver code, or use it with Ray futures.


### Non-goals


## User Benefit

The RFC enables users to 
- use the CCL-based collective APIs over Ray ObjectRefs;
- specify collective communication in a more Ray-ish (hence more declarative) way;
- Hence slightly reduce user code complexity and elimiate some boilerplate code when performing collective communication.


## Design Proposal

### Architecture

For Alternative #2, we potentially will need to modify the following part of the current codebase:
- When calling a remote actor method, we now need to check whether the method is decorated with
@collective, if so, we only proceed with the new python in-memory object store (see Alternative #2
for details), instead of using ray's object store.
- Add new functions to support collective operations in the driver program (for example,
allreduce_refs, see below for details). These functions will query our object store to get the
values, instead of the ray one.


### APIs

For Altenative #2, we introduce the following new APIs:
- @collective: The user can decorate a function with @collective to put the return value
of this function in an in-memory object store for faster collective operations. Typically,
user will decorate a function that prepares some data as collective, and uses these in-memory
objects to do following collective operations.

- allreduce_refs: The user can now use allreduce in the driver program, while before the user
needs to write collective operations in actor classes. This function is proposed to compat better
with our in-memory object-refs usage, and to give user more flexibility. See the below examples
for details.

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

- The current in-memory object store in ray uses serialization, which is slow compared to
typical collective operations. In our experience, serialization requires time in ms, while collective
operations also requires time in ms. By providing a new object store that recovers smiliar functionality
without serialization, we can finish collective operations more efficiently.
- It is likely that we need far less development in current c++ code, which shortens development
time.

### Other Considerations


### Performance Implications
- The collective communication performance in unaffected and is only determined by the CCL library.
- Since the POS is local to the process and objects are not serialized, comparing to the in-memory store and plasma, querying the objects in POS using ObjectRef has the least overhead. 


### Dependencies
Currently we don't see any outstanding dependencies, but later depending on how we implement the POS we might introduce some Python-level dependencies.

### Engineering Impact


### Platforms and Environments


### Best Practices

### Tutorials and Examples


### Compatibility

### User Impact

## Implementation Plan

## Questions and Discussion Topics

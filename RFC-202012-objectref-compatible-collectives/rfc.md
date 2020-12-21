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


### APIs


### Unsolved Problems


### Alternative Design
#### Pros
#### Cons


### Alternative #2: Using a new in-memory Python Object Store

Instead of using the current ray object store logic, we can also build a new object
store on top of Python. This new object store is always stored in memory, and only to
store inputs for collective operations. The way that the current codebase interact with the 
object store at Python level is that it calls the submit_actor_task() function during the 
remote actor method call, which goes into ray's system layer logic, and puts this future object
into th object store. Instead we can require the user to mark a function that stores the inputs
of collective operations (Possibly using a decorator). At runtime, when the system discovers that
the functions are marked, it processes with our new in-memory object store instead of invoking the 
ray's one. 

#### Cons

- Since we submit the task and store the results ourselves, these collective operations are
invisible to ray. Thus we are lack of several ray's features, including fault tolerance at fails,
and garbage collection. 
- It is unclear how ray's scheduler will react to these invisible tasks. I.e., it may think 
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

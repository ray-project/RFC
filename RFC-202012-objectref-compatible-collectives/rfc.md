# RFC-202011-collective-in-ray

| Status        | Proposed      |
:-------------- |:---------------------------------------------------- |
| **Author(s)** | Hao Zhang, Dacheng Li  |
| **Sponsor**   | Ion Stoica               |
| **Reviewer**  | |
| **Updated**   | YYYY-MM-DD                                           |


## Objective

Based on [RFC-20201119-collective-in-ray](https://github.com/ray-project/ray/pull/12174), we have brought CCL libraries such as NCCL into Ray; see the current set of APIs under [python/ray/util/collective](https://github.com/ray-project/ray/tree/master/python/ray/util/collective).

As of now, this set of collective APIs are supposed to work inside the code of an actor/task (but **NOT** in the driver code) in an imperative way; and they can only interact with the contents/arrays to be communicated, similar to most APIs in traditional CCLs (e.g., [torch.distributed](https://pytorch.org/docs/stable/distributed.html)). In this RFC, we expand this set of APIs to work with Ray ObjectRefs. Specifically, we will introduce a set of **more declarative** collective APIs that make use of ObjectRefs as inputs, such as `allreduce(ObjectRefs: List[ObjectRef])`, so users can flexibility use collective in driver code, or use it with Ray futures.


### Non-goals


## User Benefit

The RFC enables users to 
- use the CCL-based collective APIs over Ray ObjectRefs;
- specify collective communication in a more Ray-ish (hence more declarative) way;
- Hence slightly reduce user code complexity and elimiate some boilerplate code when performing collective communication.


## Design Proposal

We first expose a few design requirements for this feature. 

The main goal of introducing CCL into Ray is to improve performance for collective communication patterns e.g. `allreduce`, `allgather`, or P2P communication between GPUs, e.g. GPU `send/recv`. Hence, when expanding these APIs to work with ObjectRefs, performance is still our first concern.

First, in the current Ray, retrieving an Object via ObjectRef involves serialization and de-serialization, or sometimes IPC or RPC if the Object is not local, regardless of where the object is held (in-memory or plasma). The serialization/de-serialization overhead is sometimes at a similar scale to a collective operation (10^-5, e.g. NCCL allreduce) itself, letting alone the IPC or RPC overhead.

Second, all collective or send/recv communications exhibit a *symmetric* pattern: each process in the collective group owns a message to be communicated; each process exchanges its own version of message with all other processes. For collective APIs to work with ObjectRefs, each actor/task only needs to access a local storage (that stores the local message) to establish the communication -- no IPC/RPC required.

These two considerations help flesh out two major parts of the design: 
- A serialization-free, in-memory local object store to store collective messages;
- ObjectRef-compatible collective APIs, and specialized routes for object resolution bjects when calling collective functions

### Architecture: Using a New In-memory Python Object Store

Instead of using the current ray object store, we propose to build a light-weight object store on top of Python, called Python object store (POS). POS stores objects in the process memory, at Python level. Its major usage as of now is just to store inputs for collective operations. 

An example usage using an decorator `@pos` to tell the actor to store the returned object in POS is below:

```python
@ray.remote(num_gpus=1)
class Actor:
    def __init__(self):
        pass
    
    @pos
    def prepare(self):
        buffer = 1
        return buffer

    def compute(self, buffer):
        return buffer * 2
```

A second possible example to use a `put_to_pos(...)` API to put the object into POS would be:
```python
@ray.remote(num_gpus=1)
class Actor:
    def __init__(self):
        pass
    
    def prepare(self):
        # or similarly: ray.put(cupy_array, destination=pos)
        return ray.put_to_pos(cupy_array)

    def compute(self, buffer):
        return buffer * 2
```

With this, we can then introduce the ObjectRef-compatible APIs, such as `allreduce_refs(...)` to do the following:

```python
refs = [a.prepare.remote() for a in actors]
results = ray.collective.allreduce_refs([a.compute.remote(ref) 
                                       for (a, ref) in zip(actors, refs)])
```

To contrast, the way that the current Ray Python APIs interact with the object store at Python level is that it calls the `submit_actor_task()` function during the remote actor method call, which goes into Ray's `core_worker` logic (in CPP), and puts this future object into th object store. 

Instead we can require the user to annotate a function that stores the inputs of collective operations.
In the above example, we require the user to use the decorator `pos` to identify that these variables are going to be allreduced across processes, so that the backend knows to store them in POS. At runtime, when the system discovers that the functions are marked, it proceeds with POS APIs instead of invoking the Ray low-layer APIs.

Also, to better support these objectrefs usage, we propose to add an additional set of `allreduce_refs(List[ObjectRefs])` function in the `ray.util.collective` module, where the user can directly call allreduce at the driver program, or in a collective process. 

### APIs

we introduce the following new APIs:
- @collective: The user can decorate a function with @collective to put the return value
of this function in an in-memory object store for faster collective operations. Typically,
user will decorate a function that prepares some data as collective, and uses these in-memory
objects to do following collective operations.

- allreduce_refs: The user can now use allreduce in the driver program, while before the user
needs to write collective operations in actor classes. This function is proposed to compat better
with our in-memory object-refs usage, and to give user more flexibility. See the below examples
for details.

We next briefly discuss what implementations are needed to realize the above architecture.

#### APIs and relevant modification to existing Ray 
For Alternative #2, we potentially will need to modify the following part of the current codebase:
- When calling a remote actor method, we now need to check whether the method is decorated with
@collective, if so, we only proceed with the new python in-memory object store (see Alternative #2
for details), instead of using ray's object store.
- Add new functions to support collective operations in the driver program (for example,
allreduce_refs, see below for details). These functions will query our object store to get the
values, instead of the ray one.

### Python Object Store (POS)
The Python Object Store will be created with the actor/task process. 

Note that, unlike the normal object stores in Ray, we will:
- restrict users to only read/write to the POS inside its corresponding actor/task processes.
- prohibit `ray.get([ref], ...)` outside the actor for objects in POS.
- Any subsequent remote functions over a POS ref will result in a POS ref, unless the user explicitly call `ray.put` to promote it to other object stores
- A POS ref can only be accessed by the actor (and its methods) which owns the POS, unless it is a recv method from another actor in the same collective group.

The basic APIs of the POS would mostly resemble that of the in-memory store, exposing APIs like `put`, `get` APIs; see [memory_store.cc](https://github.com/ray-project/ray/tree/master/src/ray/core_worker/store_provider/memory_store) implementations


### Unsolved Problems
- How to put this POS inside the actor process? I am currently pretty fuzzy on this.



#### Cons

- Since we submit the task and store the results in Python, these collective operations are invisible to Ray. Thus this POS is incompatible with several of Ray's core features, including fault tolerance at failures, and garbage collection, etc.
<!-- 
- It is unclear how Ray's scheduler will react to these invisible tasks, e.g., Ray may think these nodes taskls degenerated and tend to schedule less jobs to these nodes, while the resources at these nodes are simply acquired by our collective operations. -->

#### Pros

- The current in-memory object store in Ray uses serialization, which is slow compared to typical collective operations. In our experience, serialization requires time in ms, while collective operations also requires time in ms. By providing a new object store that recovers similar functionality without serialization, we can finish collective operations more efficiently.
- 
- It is likely that we need far less development in current c++ code, which shortens development time.


### Alternative Design
#### Pros
#### Cons


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

Note that before this new function, the user will need to call allreduce at every worker. For example:

```python
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
```
By using this new `allreduce_ref` APIs, the user can have more flexibility of whether to use collective operations in the driver programs or not, and whether to perform allreduce over ObjectRefs (instead of the message array, which might be on GPUs). 

## Implementation Plan

## Questions and Discussion Topics

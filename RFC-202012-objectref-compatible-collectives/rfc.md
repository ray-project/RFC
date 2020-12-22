# RFC-202012-ObjectRef-compatible-collectives

| Status        | Proposed      |
:-------------- |:---------------------------------------------------- |
| **Author(s)** | Hao Zhang, Dacheng Li |
| **Sponsor**   | Ion Stoica, Stephanie Wang |
| **Reviewer**  | |
| **Updated**   | YYYY-MM-DD                                           |


## Objective

Based on [RFC-20201119-collective-in-ray](https://github.com/ray-project/RFC/blob/main/rfc-20201119-collective-in-ray/20201119-collective-in-ray.md), we have brought CCL libraries such as NCCL into Ray; see the current set of APIs under [python/ray/util/collective](https://github.com/ray-project/ray/tree/master/python/ray/util/collective).

As of now, this set of collective APIs are supposed to work inside the code of an actor/task (but **NOT** in the driver code), in an *imperative* way; and they can only interact with the contents/arrays to be communicated, similar to most APIs in traditional CCLs (e.g., [torch.distributed](https://pytorch.org/docs/stable/distributed.html)). In this RFC, we expand this set of APIs to work with Ray ObjectRefs. Specifically, we will introduce a set of **more declarative** collective APIs that make use of ObjectRefs as inputs, such as `allreduce(ObjectRefs: List[ObjectRef])`, so users can flexibility use collective in driver code, or use it with Ray futures.


## User Benefit

The RFC enables users to 
- use the CCL-based collective APIs over Ray ObjectRefs;
- specify collective communication in a more Ray-ish (declarative) way; this sometimes might avoid situations where the user code cause deadlocks, such as trigger send/recv operations between two actors (see more details in [RFC-20201119-collective-in-ray](https://github.com/ray-project/RFC/blob/main/rfc-20201119-collective-in-ray/20201119-collective-in-ray.md))
- In some scenario, it might reduce user code complexity and elimiate some boilerplate code when performing collective communication.

### An example use case
Support a user is trying to specify GPU-to-GPU send/recv operations using the current set of collective APIs, among two actors in a collective group. Normally a user would do:
```python
import ray.util.collective as col

@ray.remote
class CupyWorker:
    def __init__(self):
        self.buffer = cupy.ones((10,), dtype=cupy.float32)
	
    def get_buffer(self)
        return self.buffer
    
    def do_send(self, target_rank=0):
        # this call is blocking
        col.send(target_rank)
    
    def do_recv(self, src_rank=0):
        # this call is blocking
        col.recv(src_rank)
        
    def do_allreduce(self):
        # this call is blocking as well
        col.allreduce(self.buffer)
        return self.buffer
        
A = CupyWorker.remote()
B = CupyWorker.remote()

# Put A and B in a collective group
col.declare_collective_group([A, B], options={rank=[0, 1], ...})

# let A to send a message to B
# the only way to trigger and complete this send/recv is by:
ray.get([a.do_send.remote(target_rank=1), b.do_recv.remote(src_rank=0)])

# allreduce a message across two workers in the group
ray.get([a.do_allreduce.remote(), b.do_allreduce.remote()])

# The following code will hang, because it does instantiate the recv side call
ray.get([a.do_send.remote(target_rank=1)])
# This will also hang, cuz allreduce is sysmetric -- all processes need to make the call.
ray.get([a.do_allreduce.remote()])

# Following this way, the users needs to write the following code that does a round trip between A and B using send/recv:
# it might hang as well, but we have resolved it by providing a better communicator management 
ray.get([A.send.remote(target_rank=1), B.recv.remote(src_rank=0), A.recv.remote(src_rank=1), B.send.remote(target_rank=0)])
```

Essentially, to perform collective communiaction between actors using the current set of collective APIs, we add substantial coding complexity (boilerplate code for triggering collectives on every participating process) to user code. This is the case for using any CCL library in Ray, such as `torch.distributed`, or horovod.

In fact, when not using Ray, normally this "triggering" step is done by launching each collective process separatly (i.e., launching `python my_collectve_program.py` or `mpiexec my_collective_program.py` on each node of the cluster). When using Ray, the user has to use `ray.get` or `ray.wait` to appropriately trigger collective calls in each actor/task. 
For this step, the users have to acquire all the actor handles for collective workers, and make the trigger calls. This might be error prone, as shown in the examples above:
- sometimes the user might not be able to acquire all the actor handles
- the user might get confused on managing all different collective calls ane making mistakes, e.g. forgeting to trigger the collective call on one actor and then the collective operaion hangs
- [Imagenary case] In some programs, if a recv call trigge has to be delayed until another blocking call is finished, while this blocking call depends on another call that replies on the recv call, this causes a deadlock

To summarize, the current collective APIs exhibits the same semantics with mostly existing CCL APIs. They allow users to partially specificy a collective communication on a subset of participants, and will block and wait until all participant processes have triggered the collective call, which might be error-prone and sometimes cause deadlocks.

With Ray, we can actually improve the way collective calls are specified as follows:
```python
col.declare_collective_group([A, B], options={rank=[0, 1], ...})

# Specify a collective allreduce "completely" instead of "partially" on each actor
ray.util.collective.allreduce(A.get_buffer.remote(), B.get_buffer.remote())

# Specificy a send/recv safely and "completely" instead of "partially" on each actor
A.recv.remote(B.get_buffer.remote())
```
In the above code, we can use ObjectRef-compatible collective APIs to specify a collective communication **declaratively** and **completely** following Ray's design philosophy, instead of sysmetrically as normally done in CCLs (we might understand this by connecting this pattern with the `in-graph` and `between-graph` parallelization in TensorFlow, see [this stackoverflow thread](https://stackoverflow.com/questions/41600321/distributed-tensorflow-the-difference-between-in-graph-replication-and-between#:~:text=Between%2Dgraph%20replication.,train.) and Derek Murray's explanations)

This design is supposed to overcome the aforementioned disadvantages (deadlocks, etc.), and can potentially reduce user code complexity. As an example, users use one line to specify a group communicaiton, instead of multiple triggers in each process of the collective group.

A design to realize the above feature is presented below.

## Design Proposal

We first expose a few design requirements for this feature. 

The main goal of introducing CCL into Ray is to improve performance for collective communication patterns e.g. `allreduce`, `allgather`, or P2P communication between GPUs, e.g. GPU `send/recv`. Hence, when expanding these APIs to work with ObjectRefs, performance is still our first concern.

In the current Ray, retrieving an Object via ObjectRef involves serialization and de-serialization, or sometimes IPC or RPC if the Object is not local, regardless of where the object is held (in-memory or plasma store). The serialization/de-serialization overhead is at a similar scale to a collective operation (e.g., 10^-5 for an NCCL `allreduce`), letting alone the IPC or RPC overhead.

On the other hand, all collective or send/recv communications exhibit a *symmetric* pattern: each process in the collective group owns a memory buffer to be communicated; each process exchanges its owned version of message with all other processes. For collective APIs to work with ObjectRefs, each actor/task only needs to access local process memory of the actor/task to establish the communication -- no IPC/RPC required.

These two considerations help flesh out two major parts of the design: 
- A serialization-free, in-memory local object store to store collective buffers (inputs);
- ObjectRef-compatible collective APIs, and specialized routes for Ray to resolve such objects.

### Architecture: Using an In-memory Python Object Store

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

With this, we can then introduce the ObjectRef-compatible APIs under `ray.util.collective`, such as `allreduce_refs(...)`, to do the following:

```python
refs = [a.prepare.remote() for a in actors]
results = ray.collective.allreduce_refs([a.compute.remote(ref) 
                                       for (a, ref) in zip(actors, refs)])
```

To contrast, the way that the current Ray Python APIs interact with the object store at Python level is that it calls the `submit_actor_task()` function during the remote actor method call, which goes into Ray's `core_worker` logic (in CPP), and puts the return future object into the object store. 

In the above example, we require the user to use the decorator `pos` to identify that these returns are going to be allreduced across processes, so that the backend knows to store them in POS. At runtime, when the system discovers that the functions are marked, it proceeds with POS APIs instead of invoking the Ray low-layer APIs.

Also, to promote ObjectRefs colectives, we propose to add a set of `allreduce_refs(List[ObjectRefs])` function in the `ray.util.collective` module, where the user can directly declare allreduce operations over ObjectRefs, at the driver program, or in a collective process.

### APIs

we introduce the following new APIs:
- `@pos`: The user can decorate a function with @pos to put the return value of this function in an in-memory object store for faster collective operations. Typically, user will decorate a function that prepares some data as collective, and uses these in-memory objects to do following collective operations.

- `{collective}_refs`: The user can now use allreduce in the driver program, while before the user
needs to write collective operations in actor classes. This function is expected give user more flexibility. See the  examples for details.

We next briefly discuss what implementations are needed to realize the above architecture.


### Implementation Proposal #1: modify existing `ray/actor.py` and `ray/worker.py` code.

For this proposal, we potentially will need to modify the following part of the current codebase:
- When calling a remote actor method, we now need to check whether the method is decorated with
@pos, if so, we only proceed with the new python in-memory object store (see Alternative #2
for details), instead of using ray's object store.
- Add new functions to support collective operations in the driver program (for example,
allreduce_refs, see below for details). These functions will query our object store to get the
values, instead of the ray one.

#### Python Object Store (POS)
The Python Object Store will be created with the actor/task process. 

Note that, unlike the normal object stores in Ray, we will:
- restrict users to only read/write to the POS inside its corresponding actor/task processes.
- prohibit `ray.get([ref], ...)` outside the actor for objects in POS.
- Any subsequent remote functions over a POS ref will result in a POS ref, unless the user explicitly call `ray.put` to promote it to other object stores
- A POS ref can only be accessed by the actor (and its methods) which owns the POS, unless it is a recv method from another actor in the same collective group.

The basic APIs of the POS would mostly resemble that of the in-memory store, exposing APIs like `put`, `get` APIs; see [memory_store.cc](https://github.com/ray-project/ray/tree/master/src/ray/core_worker/store_provider/memory_store) implementations.

 A proof-of-concept version of the object store could just be a Python `pos: dict` as an attributes to any created actors: `actor._pos = dict()`. The key of `pos` is a generated ObjectID (as ObjectRef), while the the value corresponding to it is a pointer to the object.

#### Cons

- Since we submit the task and store the results in Python, these collective operations are invisible to Ray. Thus this POS is incompatible with several of Ray's core features, including fault tolerance at failures, and garbage collection, etc.

#### Pros

- The current in-memory object store in Ray uses serialization, which is slow compared to typical collective operations. In our experience, serialization requires time in ms, while collective operations also requires time in ms. By providing a new object store that recovers similar functionality without serialization, we can finish collective operations more efficiently.
- 
- It is likely that we need far less development in current c++ code, which shortens development time.


### Implementation Proposal #2: using a CollectiveActor class
Upon the discussion with Stephanie, we might start with implementation a CollectiveActor with a few wrappers as follows:

```python

@ray.remote
class MyActor:
    def __init__(self):
        self._pos = dict()
        self.buffer = 1
        self.idx = 0

    # get_buffer was a user function that returns self.buffer.
    @pos
    def get_buffer(self):
        buffer = self.buffer
        return buffer

def CollectiveActor(cls, ...):
    class modify_class(...):
        # Implementations to be figured out, but similar to ray.remote()
        pass
    return modify_class

```

The user can convert their actor via:
```python
actor = MyActor.remote()
actor = CollectiveActor(actor)
actor.get_buffer.remote()
```

Note here the `get_buffer` will be modified to behave as follows:
```python
def get_buffer(self):
    buffer = self.buffer
    object_ref = self._generated_ref(actor_id + self.idx)
    self._pos[object_ref] = buffer
    return object_ref   
```


In this implementation, we introduce at least two APIs: 
- a class decorator `@CollectiveActor` to convert a normal Ray actor into a CollectiveActor, which owns a POS.
- a class method decorator `@pos` that annotates that a method should put its return values in the POS and return a generated POS ObjectRef.

We require users to use these two APIs: first decorate an actor as a collective actor, then annotate those methods that stores return values in POS; or otherwise, we could by default store the return values of all class methods of a collective actor in POS.

Compared to the previous implementation:

#### Pros
- We don't have to modify any existing Ray core code, which is safe; instead, we can put this wrapper class under `ray.util.collective`.
- This appears straightforward to implement.

#### Cons
- Compared to a single `@pos` interface, we introduce an addition API `@collective_actor` -- hence more API complexity.


### Other Considerations
For this proposed design and the two possible implementations, since they use a out-of-band POS, they cannot benefit from any features the Ray in-memory or plasma store provides, such as fault tolerance, etc.

After some discussion with Stephanie, we might favor the second way of implementation because it it least disruptive to the existing Ray code. We might need more practical use cases to validate that the set of ObjectRef-compatible collectives are useful. Then we can promote it into the `ray.actor` implementation following the first proposal, which can further simplify the interface.


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

Note that before this feature, using the current collective library will need to call allreduce at every worker. For example:

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
This is error-proned, and sometimes might result in deadlocks.

By using this new `allreduce_ref` APIs, the user can have more flexibility of whether to use collective operations in the driver programs or not, and whether to perform allreduce over ObjectRefs (instead of the message array, which might be on GPUs). 

## Implementation Plan

## Questions and Discussion Topics

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
store on top of Python. This new object store is always stored in memory, and only to
store inputs for collective operations. The way that the current codebase interact with the 
object store at Python level is that it calls the submit_actor_task() function during the 
remote actor method call, which goes into ray's system layer logic, and puts this future object
into th object store. Instead we can require the user to mark a function that stores the inputs
of collective operations (Possibly using a decorator). At runtime, when the system discovers that
the functions are marked, it processes with our new in-memory object store instead of invoking the 
ray's one. 

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

+++
title = "design : alchemiscale strategist"
date = "2025-06-05T21:15:00-07:00"
author = "david@vitalxi"
cover = ""
tags = ["project-alchemiscale", "design"]
keywords = ["design", "distributed-compute"]
description = "a design document for the alchemiscale Strategist service"
showFullContent = false
readingTime = true
Toc = true
+++


This design document proposes our design for the [**alchemiscale**]({{< ref "/projects#alchemiscale" >}}) `Strategist` service.
For prior work on this subject, see [Ian Kenney's development log](https://ianmkenney.github.io/devlog/strategiest_implementation/).


# the problem

Users of **alchemiscale** currently [submit an `AlchemicalNetwork`](https://docs.alchemiscale.org/en/latest/user_guide.html#submitting-your-alchemicalnetwork-to-alchemiscale) they wish to compute via the `AlchemiscaleClient`, then [create and action `Task`s](https://docs.alchemiscale.org/en/latest/user_guide.html#creating-and-actioning-tasks) for each `Transformation` in the `AlchemicalNetwork`.
This works just fine, and gives users ultimate control over how to allocate compute across their `Transformation`s, but it also means that users either:

1. waste compute on `Transformation`s that already have converged results by indiscriminantly performing more `Task`s than are needed across the `Transformation`s in an `AlchemicalNetwork`.

2. have to micromanage the compute they allocate by making decisions on which `Transformation`s to perform more `Task`s for.


# our solution

We aim to provide users with another option: define and submit a `Strategy` alongside their `AlchemicalNetwork`, and let this `Strategy` efficiently allocate compute effort across the `Transformation`s until it is satisfied.
This will allow users to largely "fire-and-forget" their `AlchemicalNetwork`s to **alchemiscale**, only concerning themselves with retrieving results as `Task`s complete.

## user interaction

Users can create a parameterized `Strategy` from [`stratocaster`](https://github.com/OpenFreeEnergy/stratocaster), then set this as the `Strategy` on their `AlchemicalNetwork`:


```python
from alchemiscale import AlchemiscaleClient, Scope, ScopedKey
from stratocaster import ConnectivityStrategy

# instantiate client for alchemiscale instance
asc = AlchemiscaleClient('https://api.alchemiscale.localdomain')

# read in a pre-built network; choose a scope to submit to
network = AlchemicalNetwork.from_json('network.json')
scope = Scope('org', 'campaign', 'project')

# submit the network
network_sk: ScopedKey = asc.create_network(network, scope)

# create an instance of a strategy, using default settings
strategy = ConnectivityStrategy(ConnectivityStrategy.default_settings())

# set the strategy for this network
asc.set_network_strategy(network_sk, strategy)

...

# later, retrieve results for the network
results = asc.get_network_results(an_sk)

```

The `Strategy` will automatically be applied by **alchemiscale** to the `AlchemicalNetwork`.
`Task`s will be periodically created and actioned on the `AlchemicalNetwork` as needed based on the `Strategy`'s proposal for how much additional effort to allocate to each `Transformation` given the results accumulated so far.


### additional options

The `AlchemiscaleClient.set_network_strategy()` method also features the following keyword arguments that adjust how the `Strategy` is performed by **alchemiscale**, independent of the `Strategy`'s own settings:

- `max_tasks_per_transformation` : the maximum number of actioned `Task`s allowed on a `Transformation` at once; default 3

- `max_tasks_per_network` : the max number of actioned `Task`s allowed on the `AlchemicalNetwork` at once; default `None`

- `task_scaling` : modulates how to translate weights into `Task` counts; `"linear"` scales this count directly by weight, while `"exponential"` operates more conservatively by requiring higher weights to yield higher counts

- `sleep_interval` : wait time between iterations of the `Strategy`; the `Strategist` service (see [below]({{< ref "#the-strategist-service" >}})) will have also have a minimum `sleep_interval`, and the larger of the two will take effect

A user can replace the `Strategy` on the `AlchemicalNetwork` using `set_network_strategy()` as above, or can drop the `Strategy` entirely by calling:

```python
# drop strategy from the network
asc.set_network_strategy(network_sk, None)

```

### mode, status, and introspection

A `Strategy` assigned to an `AlchemicalNetwork` can be in one of the following `mode`s:

1. **full** : the `Strategy` can create, action, and cancel `Task`s on the `AlchemicalNetwork` based on its proposed `Transformation` weights

2. **partial** : the `Strategy` can only create and action `Task`s on the `AlchemicalNetwork` based on its proposed `Transformation` weights; it cannot ever cancel `Task`s

3. **disabled** : the `Strategy` is switched off, and won't be performed by the server

Independent of `mode`, an assigned `Strategy` also has a `status`.
`status` may be one of the following:

1. **awake** : the `Strategy` has not hit any stop conditions, and has not errored; it will performed according to its `mode`

2. **dormant** : the `Strategy` has reached stop conditions, and will no longer be performed until new results appear

3. **error** : the `Strategy` has encountered an `Exception`, and will no longer be performed


The `mode` is set by the user, and changes the way the `Strategist` service handles the `Strategy`.
The `status` of the `Strategy` is changed by the `Strategist` over time based on execution conditions, and can be manually set back to **awake** by a user.
See [the `Strategist` service]({{< ref "#the-strategist-service" >}}) section for details.


Users can interrogate `Strategy` state with:

```python
> asc.get_network_strategy_state(an_sk)
StrategyState(mode: 'partial', status: 'awake', iterations: 4, sleep_interval: 3600,
              last_iteration: '2025-05-30T18:24:30.540413+00:00',
              last_iteration_result_count: 1213,
              max_tasks_per_transformation: 5,
              max_tasks_per_network: None,
              task_scaling: 'exponential',
             )

```

Or specifically ask for its `status`:

```python
> asc.get_network_strategy_status(an_sk)
'partial'

```

A `Strategy` with `status` **error** will also feature the ability to get `exception` and `traceback` information from `StrategyState`:

```python
> state = asc.get_network_strategy_state(an_sk)
> state.status
'error'

> state.exception
("KeyError", "No such key 'foo'")

> state.traceback
Traceback (most recent call last):
  File "/opt/conda/lib/python3.10/site-packages/stratocaster/base/strategy.py", line 127, in propose
    return self._propose(alchemical_network, protocol_results)
    ...

```

Users can also retrieve the `Strategy` itself with:

```python
> asc.get_network_strategy(an_sk)
<ConnectivityStrategy-d78edfd56996deef89deb19e231d0e79>

```

This is useful if a user wants to make a new `Strategy` based on the settings of the current one, but with some modifications.

If the `Strategy` has gone **dormant** or entered **error** status, users can kick it **awake** with:

```python
asc.set_network_strategy_awake(an_sk)
```


## the Strategist service

All `Strategy`s submitted by users are performed server-side by the `Strategist` service.
This service directly interfaces with the state and object stores in the same way as the API services do, minimizing latency and complexity.

```mermaid
---
config:
  theme: neutral
---
graph LR;

    subgraph user client
    client[[user client]]
    end

    subgraph alchemiscale server
    api[[user api]] & strategist[[strategist]] & computeapi[[compute api]]--> statestore[(state store)] & objectstore[(object store)]
    end

    subgraph HPC
    computehpc[[compute service]]
    end

    subgraph k8s
    computek8s[[compute service]]
    end

    subgraph Folding at Home
    computefah[[compute service]]
    end

    client-->api
    computehpc-->computeapi
    computek8s-->computeapi
    computefah-->computeapi

```


The service performs the following sequence as cycles in an infinite loop:

1. Query the state store for all non-**disabled** and non-**error** `Strategy`s.

2. For each `Strategy` due for an iteration based on `sleep_interval`, dispatch the following to `ProcessPoolExecutor`:

    - If the `Strategy` is **dormant**, check the number of successful `ProtocolDAGResult`s against last recorded count.

        - If different, switch `status` to `awake` and proceed; if not, remain `dormant` and skip.

    - Pull the `AlchemicalNetwork` and all successful `ProtocolDAGResult`s for each `Transformation`, and gather into `ProtocolResult`s.

    - Feed these to `Strategy.propose()` to yield a `StrategyResult`, and acquire normalized `Transformation` weights with `StrategyResult.resolve()`.

        - If weights are all `None`, set `Strategy` `status` to **dormant** and skip. Additionally, if `Strategy` `mode` is **full**, cancel all actioned `Task`s on the `AlchemicalNetwork`.

        - For `Transformation`s with `error`ed `Task`s, set weights to `None`.

    - Convert these weights into `Task` counts for each `Transformation` (see [below]({{< ref "#proposal-weights-to-task-counts" >}}) for how we do this).

    - Set the count of actioned `Task`s for each `Transformation`, creating new `Task`s as necessary.

        - If `Strategy` `mode` is **full**, cancel `Task`s as necessary on `Transformation`s that have too many `Task`s actioned.

    - Update the `Strategy` `iteration` count, number of successful `ProtocolDAGResult`s encountered, `last_iteration` datetime.

    - If any of the above steps failed, set `Strategy` `status` to **error**.


3. Sleep for configured `Strategist` `sleep_interval`.


### proposal weights to Task counts

We require a mechanism for translating normalized `Strategy` proposal weights (continuous values from 0 to 1) into discrete `Task` counts for the `Transformation`s in an `AlchemicalNetwork`.
There are likely many reasonable ways to do this, but we propose the following per `Transformation`:

```python
w: float                               # proposed Transformation weight
max_tasks_per_transformation: int
task_scaling: str                      # 'linear' or 'exponential'

if w is None or w == 0:
    tasks = 0
elif w == 1:
    tasks = max_tasks_per_transformation
else:
    if task_scaling == 'linear':
        tasks = int(1 + w * max_tasks_per_transformation)
    elif task_scaling == 'exponential':
        tasks = int((1 + max_tasks_per_transformation)**w)

```

Which gives the following qualitative relationship between weight and `Task` counts, assuming `max_tasks_per_transformation = 6`:

```

max_tasks_per_transformation = 6

# linear

tasks        1           2           3          4           5          6
        |----------|-----------|----------|-----------|----------|-----------|
weight  0                                                                    1


# exponential

tasks                   1                         2            3      4   5 6
        |--------------------------------|----------------|--------|----|--|-|
weight  0                                                                    1


```

Following this, we (potentially) scale down the `Task` counts based on `max_tasks_per_network`:

```python
import numpy as np
task_counts: list[int]       # proposed task counts for each Transformation

total_task_counts = sum(task_counts)

if (max_tasks_per_network is not None) and
   (total_task_counts > max_tasks_per_network):
    task_counts_scaled = (np.array(task_counts) *
                          max_tasks_per_network/total_task_counts)
    task_counts_scaled = list(task_counts_scaled.astype(int))
else:
    task_counts_scaled = task_counts

```

Importantly, this rescaling retains the determinism in the `Task` counts emitted by `Strategist`.


### Strategy status transitions

A newly-set `Strategy` begins in the **awake** `status`, but can transition to **dormant** or **error** (and back again):

```mermaid
stateDiagram-v2
    direction LR
    dormant --> awake : count(ok PDRs) != last_iteration_result_count
    dormant --> awake : user reset
    awake --> dormant : Strategy stop condition

    error --> awake : user reset
    awake --> error : Strategy raise exception

```


### Strategy database schema

A `Strategy` submitted to **alchemiscale** for a given `AlchemicalNetwork` is represented in the state store (Neo4j) as:

```mermaid
graph LR;

   Strategy-- PROGRESSES -->AlchemicalNetwork

```

Where the `PROGRESSES` relationship features the (Neo4j-friendly versions of) attributes of `StrategyState`:

```python
class StrategyState:
    mode: StrategyModeEnum
    status: StrategyStatusEnum
    iterations: int
    sleep_interval: int
    last_iteration: datetime
    last_iteration_result_count: int
    max_tasks_per_transformation: int
    max_tasks_per_network: int | None
    task_scaling: StrategyTaskScalingEnum
```

Since `Strategy` is a `GufeTokenizable`, this allows the same `Strategy` object to serve multipe `AlchemicalNetwork`s in the same `Scope` if already present, while the `StrategyState` specific to each `AlchemicalNetwork` is encoded in the relationship to that `AlchemicalNetwork`.
An alternative to this approach would be to make `StrategyState` a separate node with relationships to both the `Strategy` and `AlchemicalNetwork`, but this is likely unnecessarily complex.


## miscellanea

Additional notes:

1. The `Strategist` should not create and action any new `Task`s on a `Transformation` featuring `Task`s with `status = error`.
    - users should deal with these `error`ed `Task`s in order to clear the `Transformation` for handling by the `Strategy`

2. The `Strategist` must feature an LRU cache of substantial max size for `ProtocolDAGResult`s, reducing the need to pull them from the object store each time it performs a `Strategy` iteration.
    - instead of `ProtocolDAGResult`s, this cache could retain `ProtocolResult`s along with the count of `ProtocolDAGResult` used to create them; cache invalidation would compare this count with the count present in the state store; this approach could substantially reduce cache size while still offering sufficient performance

3. When a `Strategy` is in **full** `mode`, it should prioritize canceling unclaimed `Task`s when possible to avoid wasting compute.

4. We may later want to include a mechanism for making `Strategy`s perform extensions from existing `Task`s on a `Transformation`. One idea is to include an `extends_preference` keyword argument to `AlchemiscaleClient.set_network_strategy()` with the following behavior:
    - if `0`, perform no extensions at all
    - if `1`, always perform extension when possible
    - if between `0` and `1`, extend or not extend in proportion to this value (e.g. `0.7` would mean around 70% of `Task`s created would extend from another `Task`)

5. Must be careful that setting `Strategy` to `None` doesn't delete `Strategy` object if other `PROGRESSES` relationship(s) present.

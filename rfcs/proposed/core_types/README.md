# Advanced Core Type Selection

## Introduction

Modern CPUs, especially those with hybrid architectures, present new challenges for thread affinity,
concurrency, and resource management. These systems can feature multiple core types - such as performance and
efficiency cores - and may have cores with differing cache hierarchies (e.g., lacking L3 cache).

Currently, oneTBB provides mechanisms to constrain execution to a single core type. To better support
heterogeneous and hybrid processor architectures, we propose to introduce support for multiple core-type
constraints, improving the library's ability to identify and utilize available hardware resources accurately.
This enhancement is motivated by:

- The need for improved user experience when tuning for increasingly diverse processor designs.
- The opportunity for performance improvements through more precise control over thread placement.
- The desire to future-proof the library as hardware evolves.

This proposal builds on ongoing efforts to improve hardware topology awareness and affinity support in
oneTBB, as seen in RFCs addressing NUMA and task scheduling.

## Proposal

We propose the following key changes:

- **Expansion of constraints API:**
  The `constraints` structure would support specifying multiple core types, with new methods such as:
    - `set_core_types(const std::vector<core_type_id>& ids)`
    - `get_core_types() const`
    - `single_core_type() const`
- **Improved concurrency calculation:**
  The logic for default concurrency would sum across all specified core types, ensuring better utilization on
  CPUs featuring diverse core types.
- **Enhanced hardware topology parsing:**
  The detection logic would distinguish additional core types, including those without L3 cache, providing
  more accurate affinity masks.

### Usage Example

```cpp
tbb::task_arena::constraints c;
c.set_core_types({core_type_id::performance, core_type_id::efficiency});
arena.initialize(c, 0);
```
This example would allow an arena to support both performance and efficiency cores.

### Impact and Rationale

#### New Use Cases Supported

- More granular thread placement on hybrid CPUs
- Accurate support for systems with cores lacking L3 cache
- Enables new high-level features and optimizations for scheduling and affinity

#### Performance Implications

- Potential for improved throughput and resource utilization, especially in workloads sensitive to core-type
  heterogeneity.

#### Compatibility

- The API would be backward compatible. Existing code specifying a single core type would remain valid.

- No ABI breakage; semantic versioning maintained.

#### Build System and Dependencies

- No changes to CMake or build configuration are anticipated.
- No new external dependencies introduced.

### Detailed Changes in TBBBind

The proposal includes refining the hardware topology parsing logic in TBBBind to better detect and
differentiate core types on hybrid CPUs, especially in cases where cores lack certain cache levels (such as
L3).

#### Key Enhancements

- **Core Type Detection:**
  The updated parsing algorithm would identify additional core types beyond the existing performance and
  efficiency categories. This includes distinguishing cores without L3 cache - important for hybrid
  architectures where core characteristics may vary significantly.

- **Affinity Mask Construction:**
  Affinity masks would be built with stricter separation of core types. Cores without L3 cache could be
  classified into a new core type or grouped appropriately, improving the accuracy of affinity settings and
  resource allocation.

- **Topology Robustness:**
  These changes would enhance hardware topology detection, ensuring accurate mapping of logical processors to
  their corresponding core types. This supports more granular constraints and better thread placement across
  diverse systems.

#### Example Scenario

On a hybrid CPU with both performance and efficiency cores - and with some efficiency cores lacking L3
cache - the new logic could create distinct masks for each combination, such as:
- performance cores with L3 cache
- efficiency cores with L3 cache
- efficiency cores without L3 cache

This would enable users to precisely target specific hardware resources when configuring arenas or
constraints.

#### Impact

- **Improved default concurrency calculation** for heterogeneous core systems.
- **Better support for hybrid and emerging CPU designs**, laying groundwork for future affinity and
  scheduling improvements.
- **Enhanced portability and reliability** of oneTBB applications across a wider range of hardware.

#### Future Directions

Further refinement may be required as new hardware topologies emerge, such as CPUs with more than two core
types or complex cache hierarchies.

### Alternatives Considered

- **Single Core-Type Constraint:**
  Simpler API, but increasingly inadequate for modern hardware.
- **Automatic core type grouping:**
  Might hide necessary details from users and reduce control.

#### Pros and Cons

| Approach                       | Pros                    | Cons                               |
|--------------------------------|---------------------------------------|----------------------|
| Multiple core-type constraints | Flexible, future-proof  | Slightly more complex API          |
| Single core-type constraint    | Simpler, minimal change | Inadequate for hybrid systems      |
| Automatic grouping             | Ease of use             | Reduced transparency, less control |

### Testing

- Should be validated on a wide range of hardware, including hybrid CPUs and those with mixed cache
  hierarchies.
- Regression tests should confirm backward compatibility.

## Open Questions

1. How should user-specified constraints interact with automatic topology detection?
2. Are there edge cases in hardware topology that require further handling?
3. Should the API expose additional performance metrics for multi-core-type configurations?
4. Would users benefit from explicit APIs for cores with missing cache levels?
5. Is further abstraction needed for more complex hardware partitioning scenarios?

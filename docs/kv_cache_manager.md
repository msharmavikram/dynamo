# Dynamo KV Block Manager
The Dynamo KV Block Manager reduces the cost of LLM inference by offloading rarely accessed KV blocks from GPU memory to more affordable storage tiers such as CPU DRAM, local SSDs, or remote object stores. It efficiently reuses historical KV data to alleviate GPU memory pressure and sustain SLA targets under fixed budgets. As a scalable runtime component, it handles memory allocation, lifecycle management, and remote sharing of KV blocks across heterogeneous and distributed environments. It serves as a unified memory backend for frameworks like vLLM, SGLang, and TRT-LLM, enabling petabyte-scale cache management with low overhead.

It offers: 

* A **unified memory API** that spans GPU memory, pinned host memory, remote RDMA-accessible memory, local or distributed pool of SSDs and remote file/object/cloud storage systems.  
* Support for evolving **block lifecycles** (allocate → register → match) with event-based state transitions that storage can subscribe to.  
* Integration with **NIXL**, a dynamic memory exchange layer used for remote registration, sharing, and access of memory blocks over RDMA/NVLink.

The Dynamo KV Block Manager serves as a reference implementation that emphasizes modularity and extensibility. Its pluggable design enables developers to customize components and optimize for specific performance, memory, and deployment needs.

## Why build KVBM from scratch?

Large language models (LLMs) and other AI workloads increasingly rely on KV caches that extend beyond GPU and local CPU memory into remote storage tiers. However, efficiently managing the lifecycle of KV blocks in remote storage presents challenges:

* Tailored for GenAI use cases  
* Lack of visibility into real-time block usage patterns.  
* Need for lightweight ownership driven memory management over complex object stores with unneeded overheads    
* Modular and need extremely simplified UX and be memory safe.    
* Inability to differentiate between hot (frequently accessed) and cold (infrequently accessed) blocks across the stack without intrusive application-level changes.  
* Difficulty in optimizing storage placement across heterogeneous storage tiers (e.g., SSDs, object storage, cloud storage).

Conventional systems either lack dynamic feedback mechanisms or require deep integration into core storage paths, which increases complexity and reduces portability.

## Dynamo KV Block Manager Architecture 

The Dynamo KV Block Manager (KVBM) serves as a critical infrastructure component for scaling LLM inference workloads efficiently. By cleanly separating runtime logic from memory management, and by enabling distributed block sharing, KVBM lays the foundation for high-throughput, multi-node, and memory-disaggregated AI systems.

![][image1]  
Figure 1: High level layered architecture view of Dynamo KV Block manager and how it interfaces different components of LLM inference ecosystem.


The KVBM has three primary logical layers as shown in Figure 1 \- at the top layer is the LLM inference runtimes (TRTLLM, vLLM and SGLang) that integrate through a dedicated connector module to the Dynamo KVBM module. These connectors act as translation layers, mapping runtime-specific operations and events into KVBM’s block-oriented memory interface. This decouples memory management from the inference runtime, enabling backend portability and providing memory tiering. 

The middle layer, the KVBM layer, encapsulates the core logic of the KV block manager and serves as the runtime substrate for managing block memory. The KVBM adapter layer normalizes the representations and data layout for the incoming requests across runtimes and forwards them to the core memory manager. The KVBM and the core modules implement the required internal functionality like table lookups, memory allocation, block layout management, lifecycle and state transitions and block reuse or eviction was on policies. The KVBM layer also has required abstractions for external components to integrate to override or augment its behavior. 

The last layer, the NIXL provides a unified support for enabling all data and storage transactions. NIXL enables P2P GPU transfers, enables RDMA and NVLINK remote memory sharing, dynamic block registration and metadata exchange and provides a plugin interface for storage backends. 

NIXL integrates with multiple backends: 

- Block memory (Eg. GPU HBM, Host DRAM, Remote DRAM, Local SSD when exposed as block device)   
- Local file system (e.g. POSIX)   
- Remote file system (e.g. NFS)   
- Object stores (e.g. S3-compatible)   
- Cloud storage (e.g. blob storage APIs)

NIXL abstracts away the registration and integration complexity for each backends via custom optimizable plugin architecture and enables memory blocks to be published, serialized, and accessed remotely — allowing compute and memory to be disaggregated across nodes.  Combined with the Dynamo KV Block Manager (KVBM), storage providers no longer need to retrofit or optimize individual LLM inference engines. Instead, they can focus on tuning their own stack, providing optimized endpoints, knowing that integration is smooth, standardized, and efficient. And for those who *do* wish to go further—Dynamo KVBM offers a clean separation of concerns, making custom optimization not only possible, but remarkably simple. 

## For KVBM geeks

### Key components in KVBM

The design of the KVBM is inspired from vLLM and SGLang KV block managers but with a twist from historical memory tiering design aspired in general GPU programming \[1,2, 3\]. Figure 2 shows the internal architecture of KVBM and how it works across workers using NIXL. 

![][image2]

Figure 2: Internal architecture and key modules in the Dynamo KVBM. 

#### KvBlockManager as Orchestration Layer

The \`KvBlockManager\<H, D\>\` acts as a coordinator across memory tiers—host (CPU), device (GPU), and remote—by managing per-backend block pools and exposing consistent block lifecycle APIs. It tracks KV block locations across device memory (G1), CPU memory within and across nodes (G2), local/pooled SSDs (G3), and remote storage (G4). G1-G4 are key tiers enabled by KVBM. Critical to note that KVBM treats G4 storage as an opaque blob store, unaware of internal layout optimizations.  

\`KvBlockManager\<H, D\>\` owns:

* A device-side \`BlockPool\<Device\>\`  
* A host-side \`BlockPool\<Host\>\`  
* A remote NIXL agent that allows communication and memory sharing across nodes  
* A block set registry for remote lookup and import/export of block metadata

Implementation-wise, \`KvBlockManagerState\` holds actual logic: it is initialized via \`KvBlockManagerConfig\`, which merges runtime, model, and layout configs. Remote awareness is injected via \`NixlOptions\`.

#### Block Layout and Memory Mapping

Each block is a 2D array \`\[num\_layers\]\[page\_size × inner\_dim\]\`. The memory layout is abstracted by the \`BlockLayouttrait\`. The default implementation is \`FullyContiguous\`, which stores all layers for all blocks in one region with alignment-aware stride computation:

````
```
block_stride_in_bytes = align_up(num_layers × layer_stride, alignment);
```
````

This memory layout is shared by both CPU and GPU pools but uses storage-specific backends:

* \`DeviceStorage\` → CUDA device buffer  
* \`PinnedStorage\` → page-locked host memory  
* \`SystemStorage\` → CPU heap memory (fallback/test)  
* \`NixlStorage\` → remote memory via NIXL RDMA handles (includes storage)  

Each layout is constructed using a \`LayoutConfig\`, and storage is either passed directly or allocated via a StorageAllocator.

#### BlockPool and Memory Pools (Active \+ Inactive)

Each \`BlockPool\<T\>\` (where \`T\` is \`DeviceStorage\`, \`PinnedStorage\`, etc.) tracks two sub-pools:

* \`ActiveBlockPool\`: Contains blocks currently in use, typically indexed by SequenceHash. These blocks are actively being filled or recently committed and registered for reuse. 
* \`InactiveBlockPool\`: Recycled blocks ready for allocation. Owns and recycles blocks no longer in use. Think of this as a prioritized free list with internal structure to enable efficient reuse.

When a token block is requested (e.g., via `get_mutable_block()`), the allocator retrieves a block from the `InactiveBlockPool`, updates its state accordingly, and returns a writable handle to the caller. Once the associated sequence is either committed or evicted, the block is reset and placed back into the inactive pool for future reuse.

The `InactiveBlockPool` manages block reuse using several internal data structures:
* `VecDeque:` Acts as a FIFO queue for freshly allocated or reset blocks, allowing fast access to uninitialized memory.
* `HashMap<SequenceHash, Block>`: Provides constant-time lookup for deduplication by mapping sequence hashes to their corresponding committed blocks.
* `BTreeSet<PriorityKey>`: Organizes blocks by reuse priority (e.g., LRU or custom heuristics), supporting efficient and deterministic selection for reclamation.

Each block implements the `BlockMetadata` trait, which captures key metadata fields such as:
* `priority`: A score representing reuse desirability, either user-defined or based on time/age.
* `acquired_tick` and `returned_tick`: Logical timestamps tracking block usage and lifecycle phases.

The `PriorityKey<M>` type wraps this metadata, enabling consistent ordering within the BTree and powering fair and efficient block reuse across the system.


#### Block Lifecycle and State Transitions

The block lifecycle transitions are tracked via a state machine (\`BlockState\`), which includes:

| State | Description | Ownership | Valid Actions / Transitions |
| ----- | ----- | ----- | ----- |
| Reset | Block is uninitialized or has been reset. No sequence is associated. | Held in InactiveBlockPool, reusable | init\_sequence(salt\_hash) → Partial |
| Partial | Block is being filled with tokens for a new sequence. In-progress. | Owned by the sequence creator | add\_token() / add\_tokens() (accumulate)- commit() → Complete- reset() → Reset |
| Complete | Block is fully filled with token data but not yet visible to others. | Still owned by creator thread | register() → Registered- reset() → Reset |
| Registered | Block is finalized and visible for reuse. Available in the deduplication cache. Can use block for lookups | Shared ownership (global registry) | Auto drop() → triggers Remove event and transitions to Reset |

The below tables shows the valid transition table of KBVM block manager. 

| From → To | Trigger | Validation |
| ----- | ----- | ----- |
| Reset → Partial | init\_sequence(salt\_hash) | Must not be in use |
| Partial → Complete | commit() | Must be full |
| Complete → Registered | register() | Must be finalized |
| Registered → Reset | Drop of RegistrationHandle | Automatic |
| Partial → Reset | Aborted sequence | Explicit or drop |
| Complete → Reset | Invalidated | Explicit or drop |

An example lifecycle of a block in the KVBM block manager can be thought as below: 

Let’s say a sequence requests a new KV block:

1. Allocator pops from InactiveBlockPool → Block is in Reset  
2. init\_sequence() → Transitions to Partial  
3. Tokens are appended → State remains Partial  
4. On full → commit() → State becomes Complete  
5. Register() → Block is hashed and moved to Registered. Blocks can now be used to lookup.   
6. On eviction or end-of-life → drop() of RAII handle returns block to Reset







#### Lifecycle Management via RAII and Event Plane

The system uses RAII for memory lifecycle management. Every block holds metadata and registration state, and registration is coupled with an \`EventManager\`. On registration and drop:

* \`PublishHandle\` triggers Register events  
* Dropping it triggers Remove events

This pattern ensures consistency for shared memory tracking across workers without requiring explicit deallocation logic. The events are propagated in the Dynamo Events plane and any Dynamo component subscribed to the events plane can listen to these changes. It is critical to understand that even the storage provider can subscribe to the events plane and create an internal prefix tree representation tailored and optimized for their platform. 

#### Remote Memory Integration via NIXL

The NIXL agent exposes remote memory buffers using \`NixlBlockSet\`, \`RemoteBlocks\`, and layout descriptors. Key operations include:

* \`nixl\_register()\`: Registers memory region with NIXL runtime  
* \`serialize() / deserialize()\`: Converts layout and memory into transferable descriptors  
* \`import\_remote\_blockset()\`: Loads remote node’s block layouts into the manager  
* \`get\_remote\_blocks\_mutable()\`: Fetches transferable memory views from another node

\`RemoteBlocks\` is a lightweight abstraction over shared memory for cross-node block usage (via UCX or other backends). 

The left side of the Figure 2 illustrates a bidirectional remote memory registration and layout synchronization protocol between workers (e.g., Worker 1 and Worker 2\) using NIXL. 

1. ###### *Agent Creation & Memory Registration:* 

   Each worker independently sets up a NixlAgent:   
* Registers its memory regions (e.g., device memory) via nixl\_register().   
* These regions correspond to blocks managed in the local BlockPool.  
  Once memory is registered, NIXL creates remote-accessible descriptors, which are bound to the memory layout.

2. ###### *Metadata exchange:*

   After memory registration, workers exchange serialized layout metadata, encapsulated in a \`SerializedNixlBlockLayout\`.  
   Why is this step critical?  
* LLM inference workloads often differ in *tensor parallel (TP)* configurations.  
  * Worker 1 might have TP=4, while Worker 2 has TP=8.  
  * Hence, even if both systems use similar \`FullyContiguous\` layouts, their internal slicing and alignment assumptions differ.  
* The metadata exchange bridges this semantic mismatch by sharing:  
  * LayoutConfig (num\_layers, page\_size, inner\_dim, dtype)  
  * BlockSetID  
  * Base address \+ stride information (including alignment)  
  * Device ID \+ memory type (host/device)  
* Once shared, each worker can reconstruct the layout on its side using deserialize().  
  This enables NIXL to:  
* Understand where each layer/block lives  
* Perform correct gather-scatter operations during RDMA-like transfers  
  Without this step, remote fetches would result in data corruption or misaligned tokens.


3. ###### *Serialization & Deserialization: Making Layouts Portable*

   In the serialization stage, KVBM exports, \`FullyContiguous::serialize()\` encodes:  
* FullyContiguousConfig  
* base\_offset  
* Physical memory descriptors (NixlStorage) including:  
  * Memory type (VRAM, DRAM)  
  * Address & size  
  * Device ID

  This is sent over using NIXL transfer and then injected into a KVBM scheduler state. In the deserialization stage, \`SerializedNixlBlockLayout::deserialize()\` rehydrates this into:

* A fully reconstructed memory layout view  
* Local representation of a remote memory slice with correct offsets and size semantics  
* Enables direct access to remote memory with consistent logical semantics  
  This guarantees that even across different system configurations (hardware or LLM shape), both parties agree on the memory view for each KV block.

4. ###### *Ownership handles and lifetime tracking* 

   Memory ownership in NIXL is tightly coupled with RAII-based handles:  
* When a block is registered, it returns a \`PublishHandle\` which wraps a \`RegistrationHandle\`.  
* On drop of this handle, an automatic Remove event is published, which:  
  * Deregisters the block from the NIXL layer  
  * Removes it from the remote block registry  
* This ensures that once the block is evicted from the cache or no longer used in inference, all references are invalidated cleanly across nodes.  
  This mechanism avoids:  
* Stale memory access  
* Dangling pointers on GPU or host  
* Manual deregistration bugs  
  The system can batch and publish registration events via a Publisher, optimizing performance under high concurrency.


#### Storage backends and pluggability

Integrating KVBM with storage backend is extremely trivial by extending or wrapping \`NixlEnabledStorage\` to support cross-node RDMA registration. All layouts and block pools are generic over these backends, allowing for fine-grained control over memory tiers.  We are deferring detailed integration guidance as we are actively collaborating with storage partners to simplify and standardize these integration paths. In a future release, we will share concrete examples and pluggable backend templates for file systems, NVMe devices, and RDMA-aware object stores. 

```
An example system architecture
                        +------------------------------+
                        |Distributed Inference engine  |
                        +------------------------------+
                                  |
                                  v
                        +------------------------------+
                        |  Dynamo KV Block Manager      |
                        +------------------------------+
                                  |
                 +----------------+----------------+
                 |                                 |
                 v                                 v
   +------------------------------+    +----------------------------+
   |        NIXL Storage Agent     |    |        Event Plane          |
   |  - Volume registration        |    |  - NATS-based Pub/Sub       |
   |  - get()/put() abstraction    |    |  - StoreEvent / RemoveEvent |
   +------------------------------+    +----------------------------+
                 |                                 |
                 v                                 v
     +-----------------------------+   +-----------------------------+
     |   G4 Storage Infrastructure  |   | Storage Provider Subscriber |
     |  (SSD, Object store, etc.)   |   |  - Parse Events             |
     |  - Store KV blocks           |   |  - Build fast tree/index    |
     +-----------------------------+    |  - Optimize G4 tiering      |
                                        +-----------------------------+
```

For now, the following breakdown provides a high-level understanding of how KVBM interacts with external storage using the NIXL storage interface and the Dynamo Event Plane:

##### NIXL Storage Interface (for Backend Integration)

The NIXL interface abstracts volume interaction and decouples it from mounting, metadata tracking, or direct system I/O. It provides:

* registerVolume(descriptor): Register a logical volume for KV cache data.  
* unregisterVolume(): Cleanly deregister and release volume mappings.  
* get() / put(): Block-level APIs used by KVBM to fetch and store token blocks.

These abstractions allow backends to be integrated without tying into the host’s file system stack, enabling safe interaction with block devices, local filesystems, and RDMA-capable volumes. Please note that these APIs are still being finalized. 

##### Dynamo Event Plane (Pub/Sub Coordination Layer)

To support external storage optimizations without modifying KVBM logic, we provide an **event plane** built on NATS.io that emits lifecycle events for all block operations. Particularly there are two events emitted.  

* StoreEvent: Emitted when a KV block is registered.  
* RemoveEvent: Emitted when a KV block is released or evicted.

Each KVEvent (\~100 bytes) contains:

* sequence\_hash: Unique identifier of the KV block  
* prefix\_hash: Prefix grouping for query-level aggregation  
* block\_size: Size in bytes  
* storage\_location: Logical volume identifier  
* event\_type: Store or Remove  
* extra\_metadata: Reserved fields for partner-specific optimization

These events are batched and published periodically (e.g., every \~10s or dynamically based on system load) for scalability.

##### A conceptual design of a storage advisor

This section provides an overview for the storage provider who is interested in integrating as a custom backend to KVBM and providing optimized performance. ***Please note, this is optional and not required for KVBM to integrate with a backend.*** 

External storage systems are not tightly coupled with Dynamo’s execution pipeline. Instead, they passively observe KV block lifecycle events through a subscription model:

* Storage volumes are pre-provisioned and mounted by the storage provider.  
* These volumes are then registered with Dynamo via the NIXL Storage Agent using registerVolume() APIs. Dynamo itself does not manage mounts or provisioning.  
* The Dynamo KV Block Manager interacts only with logical block-level APIs (i.e., get() and put()).  
* In parallel, the Event Plane asynchronously broadcasts KV lifecycle events using a NATS-based pub/sub channel.  
* Storage vendors implement a lightweight subscriber process that listens to these events without interfering with the KV Manager’s runtime behavior.  
* This decoupling ensures that external storage systems can optimize block placement and lifecycle tracking without modifying or instrumenting the core Dynamo codebase.

Now, to enable fast lookup and dynamic tiering, storage vendors may build internal data structures using the received event stream. Here is a high level conceptual design:

* On receiving a StoreEvent, the storage system:  
  * Inserts a record into an internal prefix tree, hash map, or LRU index.  
  * This record includes the prefix\_hash and sequence\_hash, which logically identify the token block and its grouping.  
  * Associated metadata (e.g., block\_size, storage\_location) is also captured.  
* On receiving a RemoveEvent, the system:  
  * Deletes or prunes the corresponding record from its index.  
  * Optionally triggers cleanup or tier migration workflows.

This event-driven indexing allows the storage system to track which KV blocks are live and where they belong—enabling low-latency lookup, efficient space reclamation, and multi-tier coordination. With real-time visibility into KV block usage patterns, the storage system can implement smart tiering policies, such as:

* Hot block promotion: Frequently accessed KV blocks can be migrated to fast SSD volumes.  
* Cold block demotion: Infrequently used blocks can be demoted to slower storage (e.g., HDDs, cloud object storage).  
* Proactive compaction: If block sizes or prefix patterns indicate fragmentation, the storage backend can coalesce or rewrite blocks.

These optimizations are performed entirely outside of Dynamo, with the assumption that storage providers will adhere to SLA guarantees and volume availability.

Critically, this entire system is designed to be non-intrusive:

* The Dynamo KV Block Manager remains agnostic to how data is stored or optimized.  
* The Event Plane does not block or intercept any critical path of inference.  
* Storage vendors are given the freedom to innovate and optimize without requiring changes to the inference runtime.

This design ensures that performance, resilience, and extensibility scale independently across the KV layer and the storage backend layer.

## References

1. vLLM : [https://docs.vllm.ai/en/latest/design/automatic\_prefix\_caching.html](https://docs.vllm.ai/en/latest/design/automatic_prefix_caching.html)   
2. SGLang : [https://github.com/sgl-project/sglang/pull/2693](https://github.com/sgl-project/sglang/pull/2693)   
3. EMOGI: [https://arxiv.org/abs/2006.06890](https://arxiv.org/abs/2006.06890) 

[image1]: ./images/kvbm_logical_view.png
[image2]: ./images/kvbm_arch.png 

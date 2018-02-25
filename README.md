# Oracle Coherence

## Cache Design

### Cache Types 
  - Replicated Cache
    - Replicated cache service
  - Partitioned Cache (Distributed)
    - Partitioned/Distributed cache service
  - Near cache
    - Partitioned/Distributed cache service
  - CQC (Continuous Query Cache)

### Backing Map
  - Local cache
    - Backing map for replicated and partitioned caches
	- front cache for near and continuous query caches
	- It stores all the data on the heap
	- Eviction policy: 
	  - LRU (Least Recently Used)
	  - LFU (Least Frequently Used)
	  - HYBRID, which is the default and uses a combination of LRU and LFU
  - External Backing Maps
    - NIO Memory Manager 
	  - off-heap NIO buffer
	  - not affected by the garbage collection
	- NIO File Manager
	  - NIO memory-mapped files
	  - performance can vary widely depending on the OS and JVM 

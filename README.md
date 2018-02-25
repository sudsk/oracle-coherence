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
    - Berkeley DB Store Manager
      - implement on-disk storage of cache items 
  - Paged external backing map
    - instead of storing cache items in a single large file, a paged backing map breaks it up into a series of pages. 
    - Each page is a separate store, created by the specified store manager. 
    - The page that was last created is considered current and all write operations are performed against that page until a new one is created.
  - Overflow backing map
    - composite backing map 
    - two tiers: a fast, size-limited, in-memory front tier, and a slower, but potentially much larger back tier on a disk.
  - Read-write backing map
    - composite backing map
    - single internal cache (usually a local cache) and either a cache loader, which allows it to load data from the external data source on cache misses, or a cache store, which also provides the ability to update data in the external data store on cache puts.
    - key enabler of the read-through/ write-through architecture that places Coherence as an intermediate layer between the application and the data store
    - provides several cache loader and cache store implementations for relational database access out of the box, such as JPA, Oracle TopLink, and Hibernate.
   - Partitioned backing map
     - contains one backing map instance for each cache partition, which allows you to scale the cache size by simply increasing the number of partitions for a given cache. 
     
## Cache configuration

### Data access pattern
  - Read-only or Read-mostly
    - replicated cache 
    - partitioned cache fronted by a near or continuous query cache
  - Transactional data
    - partitioned caches to manage those entities
    - possibly fronted with a near cache depending on the ratio of reads to writes.
    
### Caching Schemes
  - Distributed cache scheme
    ```
    <distributed-scheme>
	<scheme-name>example-distributed</scheme-name>
	<service-name>DistributedCache</service-name>
	<backing-map-scheme>
	<local-scheme>
	<scheme-ref>example-binary-backing-map</scheme-ref>
	</local-scheme>
	</backing-map-scheme>
	<autostart>true</autostart>
	</distributed-scheme>
    ```

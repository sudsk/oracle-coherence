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
    - distributed-scheme, tells us that any cache that uses this scheme will use a partitioned topology
    - scheme-name element allows us to specify a name for the caching scheme, which we can later use within cache mappings and when referencing a caching scheme from another caching scheme
    - service-name element is used to specify the name of the cache service that all caches using this particular scheme will belong to
    - backing-map-scheme, defines the type of the backing map we want all caches mapped to this caching scheme to use
  - Local cache scheme
    ```
    <local-scheme>
	<scheme-name>example-binary-backing-map</scheme-name>
	<eviction-policy>HYBRID</eviction-policy>
	<high-units>{back-size-limit 0}</high-units>
	<unit-calculator>BINARY</unit-calculator>
	<expiry-delay>{back-expiry 1h}</expiry-delay>
	<flush-delay>1m</flush-delay>
	<cachestore-scheme></cachestore-scheme>
	</local-scheme>
    ```
    - local-scheme allows you to configure various options for the local cache, such as eviction policy, the maximum number of units to keep within the cache, as well as expiry and flush delay.
    - The high-units and unit-calculator elements are used together to limit the size of the cache, as the meaning of the former is defined by the value of the latter. Coherence uses unit calculator to determine the "size" of cache entries. There are two built-in unit calculators: fixed and binary.
    - binary use is constrained by the fact that the entries need to be in a serialized binary format, which means that it can only be used to limit the size of a partitioned cache
  - Near cache scheme
    ```
    <near-scheme>
	<scheme-name>example-near</scheme-name>
	<front-scheme>
	<local-scheme>
	<scheme-ref>example-binary-backing-map</scheme-ref>
	</local-scheme>
	</front-scheme>
	<back-scheme>
	<distributed-scheme>
	<scheme-ref>example-distributed</scheme-ref>
	</distributed-scheme>
	</back-scheme>
	<invalidation-strategy>present</invalidation-strategy>
	<autostart>true</autostart>
	</near-scheme>
    ```
    - Unfortunately, the example-binary-backing-map won't quite work as a front cache in the preceding definition;it uses the binary unit calculator, which cannot be used in the front tier of a near cache. 
  - READ-WRITE BACKING MAP SCHEME
    - Using local cache as a backing map is very convenient during development and testing, but more likely than not you will want your data to be persisted as well. If that's the case, you can configure a read-write backing map as a backing map for your distributed cache
    ```
    <distributed-scheme>
	<scheme-name>example-distributed</scheme-name>
	<service-name>DistributedCache</service-name>
	<backing-map-scheme>
	<read-write-backing-map-scheme>
	<internal-cache-scheme>
	<local-scheme/>
	</internal-cache-scheme>
	<cachestore-scheme>
	<class-scheme>
	<class-name>
	com.tangosol.coherence.jpa.JpaCacheStore
	</class-name>
	<init-params>
	<init-param>
	<param-type>java.lang.String</param-type>
	<param-value>{cache-name}</param-value>
	</init-param>
	<init-param>
	<param-type>java.lang.String</param-type>
	<param-value>{class-name}</param-value>
	</init-param>
	<init-param>
	<param-type>java.lang.String</param-type>
	<param-value>PersistenceUnit</param-value>
	</init-param>
	</init-params>
	</class-scheme>
	</cachestore-scheme>
	</read-write-backing-map-scheme>
	</backing-map-scheme>
	<autostart>true</autostart>
	</distributed-scheme>
    ```
    - The read-write backing map defined previously uses unlimited local cache to store the data, and a JPA-compliant cache store implementation that will be used to persist the data on cache puts, and to retrieve it from the database on cache misses.
  - PARTITIONED BACKING MAP
    - the partitioned backing map is your best option for very large caches. The following example demonstrates how you could configure a partitioned backing map that will allow you to store 1 TB of data in a 50-node cluster
    ```
    <distributed-scheme>
	<scheme-name>large-scheme</scheme-name>
	<service-name>LargeCacheService</service-name>
	<partition-count>20981</partition-count>
	<backing-map-scheme>
	<partitioned>true</partitioned>
	<external-scheme>
	<high-units>20</high-units>
	<unit-calculator>BINARY</unit-calculator>
	<unit-factor>1073741824</unit-factor>
	<nio-memory-manager>
	<initial-size>1MB</initial-size>
	<maximum-size>50MB</maximum-size>
	</nio-memory-manager>
	</external-scheme>
	</backing-map-scheme>
	<backup-storage>
	<type>off-heap</type>
	<initial-size>1MB</initial-size>
	<maximum-size>50MB</maximum-size>
	</backup-storage>
	<autostart>true</autostart>
	</distributed-scheme>
	```
	- We have configured partition-count to 20,981, which will allow us to store 1 TB of data in the cache while keeping the partition size down to 50 MB.
	- We have then used the partitioned element within the backing map scheme definition to let Coherence know that it should use the partitioned backing map implementation instead of the default one.
	- The external-scheme element is used to configure the maximum size of the backing map as a whole, as well as the storage for each partition. Each partition uses an NIO buffer with the initial size of 1 MB and a maximum size of 50 MB.
	- The backing map as a whole is limited to 20 GB using a combination of high-units, unit-calculator, and unit-factor values. Because we are storing serialized objects off-heap, we can use binary calculator to limit cache size in bytes. However, the high-units setting is internally represented by a 32-bit integer, so the highest value we could specify for it would be 2 GB.
	- In order allow for larger cache sizes while preserving backwards compatibility, Coherence engineers decided not to widen high-units to 64 bits. Instead, they introduced the unit-factor setting, which is nothing more than a multiplier for the high-units value. In the preceding example, the unit-factor is set to 1 GB, which in combination with the high-units setting of 20 limits cache size per node to 20 GB.
	- Finally, when using a partitioned backing map to support very large caches off-heap, we cannot use the default, on-heap backup storage. The backup storage is always managed per partition, so we had to configure it to use off-heap buffers of the same size as primary storage buffers.
  - PARTITIONED READ-WRITE BACKING MAP
    - we can use a partitioned read-write backing map to support automatic persistence for very large caches. The following example is really just a combination of the previous two examples, so I will not discuss the details. 
    ```
    <distributed-scheme>
	<scheme-name>large-persistent-scheme</scheme-name>
	<service-name>LargePersistentCacheService</service-name>
	<partition-count>20981</partition-count>
	<backing-map-scheme>
	<partitioned>true</partitioned>
	<read-write-backing-map-scheme>
	<internal-cache-scheme>
	<external-scheme>
	<high-units>20</high-units>
	<unit-calculator>BINARY</unit-calculator>
	<unit-factor>1073741824</unit-factor>
	<nio-memory-manager>
	<initial-size>1MB</initial-size>
	<maximum-size>50MB</maximum-size>
	</nio-memory-manager>
	</external-scheme>
	</internal-cache-scheme>
	<cachestore-scheme>
	<class-scheme>
	<class-name>
	com.tangosol.coherence.jpa.JpaCacheStore
	</class-name>
	<init-params>
	<init-param>
	<param-type>java.lang.String</param-type>
	<param-value>{cache-name}</param-value>
	</init-param>
	<init-param>
	<param-type>java.lang.String</param-type>
	<param-value>{class-name}</param-value>
	</init-param>
	<init-param>
	<param-type>java.lang.String</param-type>
	<param-value>SigfePOC</param-value>
	</init-param>
	</init-params>
	</class-scheme>
	</cachestore-scheme>
	</read-write-backing-map-scheme>
	</backing-map-scheme>
	<backup-storage>
	<type>off-heap</type>
	<initial-size>1MB</initial-size>
	<maximum-size>50MB</maximum-size>
	</backup-storage>
	<autostart>true</autostart>
	</distributed-scheme>
    ```

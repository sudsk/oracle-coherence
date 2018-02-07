# Production Checklist
This chapter provides a checklist of areas that should be planned for and considered when moving from a development or test environment to a production environment. Solutions and best practices are provided and should be implemented as required. Additional recommendations when using Coherence*Extend can be found in Developing Remote Clients for Oracle Coherence.
The following sections are included in this chapter:

- Network Performance Test and Multicast Recommendations
- Network Recommendations
- Cache Size Calculation Recommendations
- Hardware Recommendations
- Operating System Recommendations
- JVM Recommendations
- Oracle Exalogic Elastic Cloud Recommendations
- Security Recommendations
- Application Instrumentation Recommendations
- Coherence Modes and Editions
- Coherence Operational Configuration Recommendations
- Coherence Cache Configuration Recommendations
- Large Cluster Configuration Recommendations
- Death Detection Recommendations

## Network Performance Test and Multicast Recommendations

###Test TCP Network Performance

Run the message bus test utility to test the actual network speed and determine its capability for pushing large amounts TCP messages. Any production deployment should be preceded by a successful run of the message bus test. See "Running the Message Bus Test Utility," for details. A TCP stack is typically already configured for a network and requires no additional configuration for Coherence. If TCP performance is unsatisfactory, consider changing TCP settings. For common TCP settings that can be tuned, see "TCP Considerations."

###Test Datagram Network Performance

Run the datagram test utility to test the actual network speed and determine its capability for pushing datagram messages. Any production deployment should be preceded by a successful run of both tests. See Performing a Network Performance Test, for details. Furthermore, the datagram test utility must be run with an increasing ratio of publishers to consumers, since a network that appears fine with a single publisher and a single consumer may completely fall apart as the number of publishers increases.

###Consider the Use of Multicast

The term multicast refers to the ability to send a packet of information from one server and to have that packet delivered in parallel by the network to many servers. Coherence supports both multicast and multicast-free clustering. The use of multicast can be used to ease cluster configuration. However, the use of multicast may not always be possible for several reasons:

Some organizations disallow the use of multicast.

Multicast cannot operate over certain types of network equipment; for example, many WAN routers disallow or do not support multicast traffic.

Multicast is occasionally unavailable for technical reasons; for example, some switches do not support multicast traffic.

Run the multicast test to verify that multicast is working and to determine the correct (the minimum) TTL value for the production environment. Any production deployment should be preceded by a successful run of the multicast test. See Performing a Multicast Connectivity Test for more information.

Applications that cannot use multicast for deployment must use unicast and the well known addresses feature. See Developing Applications with Oracle Coherence for details.

Configure Network Devices

Network devices may require configuration even if all network performance tests and the multicast test pass without incident and the results are perfect. Review the suggestions in "Network Tuning".

Changing the Default Cluster Port

The default cluster port is 7574 and for most use cases does not need to be changed. This port number, or any other selected port number, must not be within the operating system ephemeral port range. Ephemeral ports can be randomly assigned to other processes and can result in Coherence not being able to bind to the port during startup. On most operating systems, the ephemeral port range typically starts at 32,768 or higher. Some versions of Linux, such as Red Hat, have a much lower ephemeral port range and additional precautions must be taken to avoid random bind failures.

On Linux the ephemeral port range can be queried as follows:

sysctl net.ipv4.ip_local_port_range

sysctl net.ipv4.ip_local_reserved_ports
The first command shows the range as two space separated values indicating the start and end of the range. The second command shows exclusions from the range as a comma separated list of reserved ports, or reserved port ranges (for example, (1,2,10-20,40-50, and so on).

If the desired port is in the ephemeral range and not reserved, you can modify the reserved set and optionally narrow the ephemeral port range. This can be done as root be editing /etc/sysctl.conf. For example:

net.ipv4.ip_local_port_range = 9000 65000
net.ipv4.ip_local_reserved_ports = 7574
After editing the file you can then trigger a reload of the settings by running:

sysctl -p
5.2 Network Recommendations
Ensure a Consistent IP Protocol

It is suggested that cluster members share the same setting for the java.net.preferIPv4Stack property. In general, this property does not need to be set. If there are multiple clusters running on the same machine and they share a cluster port, then the clusters must also share the same value for this setting. In rare circumstances, such as running multicast over the loopback address, this setting may be required.

Test in a Clustered Environment

After the POC or prototype stage is complete, and until load testing begins, it is not out of the ordinary for an application to be developed and tested by engineers in a non-clustered form. Testing primarily in the non-clustered configuration can hide problems with the application architecture and implementation that appear later in staging or even production.

Make sure that the application has been tested in a clustered configuration before moving to production. There are several ways for clustered testing to be a natural part of the development process; for example:

Developers can test with a locally clustered configuration (at least two instances running on their own computer). This works well with the TTL=0 setting, since clustering on a single computer works with the TTL=0 setting.

Unit and regression tests can be introduced that run in a test environment that is clustered. This may help automate certain types of clustered testing that an individual developer would not always remember (or have the time) to do.

Evaluate the Production Network's Speed for both UDP and TCP

Most production networks are based on 10 Gigabit Ethernet (10GbE), with some still built on Gigabit Ethernet (GbE) and 100Mb Ethernet. For Coherence, GbE and 10GbE are suggested and 10GbE is recommended. Most servers support 10GbE, and switches are economical, highly available, and widely deployed.

It is important to understand the topology of the production network, and what the devices are used to connect all of the servers that run Coherence. For example, if there are ten different switches being used to connect the servers, are they all the same type (make and model) of switch? Are they all the same speed? Do the servers support the network speeds that are available?

In general, all servers should share a reliable, fully switched network. This generally implies sharing a single switch (ideally, two parallel switches and two network cards per server for availability). There are two primary reasons for this. The first is that using multiple switches almost always results in a reduction in effective network capacity. The second is that multi-switch environments are more likely to have network partitioning events where a partial network failure results in two or more disconnected sets of servers. While partitioning events are rare, Coherence cache servers ideally should share a common switch.

To demonstrate the impact of multiple switches on bandwidth, consider several servers plugged into a single switch. As additional servers are added, each server receives dedicated bandwidth from the switch backplane. For example, on a fully switched gigabit backplane, each server receives a gigabit of inbound bandwidth and a gigabit of outbound bandwidth for a total of 2Gbps full duplex bandwidth. Four servers would have an aggregate of 8Gbps bandwidth. Eight servers would have an aggregate of 16Gbps. And so on up to the limit of the switch (in practice, usually in the range of 160-192Gbps for a gigabit switch). However, consider the case of two switches connected by a 4Gbps (8Gbps full duplex) link. In this case, as servers are added to each switch, they have full mesh bandwidth up to a limit of four servers on each switch (that is, all four servers on one switch can communicate at full speed with the four servers on the other switch). However, adding additional servers potentially create a bottleneck on the inter-switch link. For example, if five servers on one switch send data to five servers on the other switch at 1Gbps per server, then the combined 5Gbps is restricted by the 4Gbps link. Note that the actual limit may be much higher depending on the traffic-per-server and also the portion of traffic that must actually move across the link. Also note that other factors such as network protocol overhead and uneven traffic patterns may make the usable limit much lower from an application perspective.

Avoid mixing and matching network speeds: make sure that all servers connect to the network at the same speed and that all of the switches and routers between those servers run at that same speed or faster.

Plan for Sustained Network Outages

The Coherence cluster protocol can of detect and handle a wide variety of connectivity failures. The clustered services are able to identify the connectivity issue, and force the offending cluster node to leave and re-join the cluster. In this way the cluster ensures a consistent shared state among its members.

See "Death Detection Recommendations" for more details. See also "Deploying to Cisco Switches"

Plan for Firewall Port Configuration

Coherence clusters members that are located outside of a firewall must be able to communicate with cluster members that are located within the firewall. Configure the firewall to allow Coherence communication as required. The following list shows common default ports and additional areas where ports are configured.

cluster port: The default multicast port is 7574. For details, see Developing Applications with Oracle Coherence.

unicast ports: UDP requires two ports and TMB uses one port. The default unicast ports are automatically assigned from the operating system's available ephemeral port range. For clusters that communicate across a firewall, a range of ports can be specified for coherence to operate within. Using a range rather then a specific port allows multiple cluster members to reside on the same machine and use a common configuration. For details, see Developing Applications with Oracle Coherence.

Note:

In general, using a firewall within a cluster (even between TCMP clients and TCMP servers) is an anti-pattern as it is very easy to mis-configure and prone to reliability issues that can be hard to troubleshoot in a production environment. By definition, any member within a cluster should be considered trusted. Untrusted members should not be allowed into the cluster and should connect as Coherence extend clients or using a services layer (HTTP, SOA, and so on).

port 7: The default port of the IpMonitor component that is used for detecting hardware failure of cluster members. Coherence doesn't bind to this port, it only tries to connect to it as a means of pinging remote machines. The port needs to be open in order for Coherence to do health monitoring checks.

Proxy service ports: The proxy by default listens on an ephemeral port. For firewall based configurations, this can be restricted to a range of ports which can then be opened in the firewall. Using a range of ports allows multiple cluster members to be run on the same machine and share a single common configuration. For details, see Developing Remote Clients for Oracle Coherence.

Coherence REST ports: Any number of ports that are used to allow remote connections from Coherence REST clients. For details, see Developing Remote Clients for Oracle Coherence.

5.3 Cache Size Calculation Recommendations
The recommendations in this section are used to calculate the approximate size of a cache. Understanding what size cache is required can help determine how many JVMs, how much physical memory, and how many CPUs and servers are required. Hardware and JVM recommendations are provided later in this chapter. The recommendations are only guidelines: an accurate view of size can only be validated through specific tests that take into account an application's load and use cases that simulate expected users volumes, transactions profiles, processing operations, and so on.

As a starting point, allocate at least 3x the physical heap size as the data set size, assuming that you are going to keep 1 backup copy of primary data. To make a more accurate calculation, the size of a cache can be calculated as follows and also assumes 1 backup copy of primary data:

Cache Capacity = Number of entries * 2 * Entry Size

Where:

Entry Size = Serialized form of the key + Serialized form of the Value + 150 bytes

For example, consider a cache that contains 5 million objects, where the value and key serialized are 100 bytes and 2kb, respectively.

Calculate the entry size:

100 bytes + 2048 bytes + 150 bytes = 2298 bytes

Then, calculate the cache capacity:

5000000 * 2 * 2298 bytes = 21,915 MB

If indexing is used, the index size must also be taken into account. Un-ordered cache indexes consist of the serialized attribute value and the key. Ordered indexes include additional forward and backward navigation information.

Indexes are stored in memory. Each node will require 2 additional maps (instances of java.util.HashMap) for an index: one for a reverse index and one for a forward index. The reverse index size is a cardinal number for the value (size of the value domain, that is, the number of distinct values). The forward index size is of the key set size. The extra memory cost for the HashMap is about 30 bytes. Extra cost for each extracted indexed value is 12 bytes (the object reference size) plus the size for the value itself.

For example, the extra size for a Long value is 20 bytes (12 bytes + 8 bytes) and for a String is 12 bytes + the string length. There is also an additional reference (12 bytes) cost for indexes with a large cardinal number and a small additional cost (about 4 bytes) for sorted indexes. Therefore, calculate an approximate index cost as:

Index size = forward index map + backward index map + reference + value size

For an indexed Long value of large cardinal, it's going to be approximately:

30 bytes + 30 bytes + 12 bytes + 8 bytes = 80 bytes

For an indexed String of an average length of 20 chars it's going to be approximately:

30 bytes + 30 bytes + 12 bytes + (20 bytes * 2) = 112 bytes

The index cost is relatively high for small objects, but it's constant and becomes less and less expensive for larger objects.

Sizing a cache is not an exact science. Assumptions on the size and maximum number of objects have to be made. A complete example follows:

Estimated average entry size = 1k

Estimated maximum number of cache objects = 100k

String indexes of 20 chars = 5

Calculate the index size:

5 * 112 bytes * 100k = 56MB

Then, calculate the cache capacity:

100k * 2 * 1k + 56MB = ~312MB

Each JVM stores on-heap data itself and requires some free space to process data. With a 1GB heap this will be approximately 300MB or more. The JVM process address space for the JVM – outside of the heap is also approximately 200MB. Therefore, to store 312MB of data requires the following memory for each node in a 2 node JVM cluster:

312MB (for data) + 300MB (working JVM heap) + 200MB (JVM executable) = 812MB (of physical memory)

Note that this is the minimum heap space that is required. It is prudent to add additional space, to take account of any inaccuracies in your estimates, about 10%, and for growth (if this is anticipated). Also, adjust for M+N redundancy. For example, with a 12 member cluster that needs to be able to tolerate a loss of two servers, the aggregate cache capacity should be based on 10 servers and not 12.

With the addition of JVM memory requirements, the complete formula for calculating memory requirements for a cache can be written as follows:

Cache Memory Requirement = ((Size of cache entries + Size of indexes) * 2 (for primary and backup)) + JVM working memory (~30% of 1GB JVM)

5.4 Hardware Recommendations
Plan Realistic Load Tests

Development typically occurs on relatively fast workstations. Moreover, test cases are usually non-clustered and tend to represent single-user access (that is, only the developer). In such environments the application may seem extraordinarily responsive.

Before moving to production, ensure that realistic load tests have been routinely run in a cluster configuration with simulated concurrent user load.

Develop on Adequate Hardware Before Production

Coherence is compatible with all common workstation hardware. Most developers use PC or Apple hardware, including notebooks, desktops and workstations.

Developer systems should have a significant amount of RAM to run a modern IDE, debugger, application server, database and at least two cluster instances. Memory utilization varies widely, but to ensure productivity, the suggested minimum memory configuration for developer systems is 2GB.

Select a Server Hardware Platform

Oracle works to support the hardware that the customer has standardized on or otherwise selected for production deployment.

Oracle has customers running on virtually all major server hardware platforms. The majority of customers use "commodity x86" servers, with a significant number deploying Oracle SPARC and IBM Power servers.

Oracle continually tests Coherence on "commodity x86" servers, both Intel and AMD.

Intel, Apple and IBM provide hardware, tuning assistance and testing support to Oracle.

If the server hardware purchase is still in the future, the following are suggested for Coherence:

It is strongly recommended that servers be configured with a minimum of 32GB of RAM. For applications that plan to store massive amounts of data in memory (tens or hundreds of gigabytes, or more), evaluate the cost-effectiveness of 128GB or even 256GB of RAM per server. Also, note that a server with a very large amount of RAM likely must run more Coherence nodes (JVMs) per server to use that much memory, so having a larger number of CPU cores helps. Applications that are data-heavy require a higher ratio of RAM to CPU, while applications that are processing-heavy require a lower ratio.

A minimum of 1000Mbps for networking (for example, Gigabit Ethernet or better) is strongly recommended. NICs should be on a high bandwidth bus such as PCI-X or PCIe, and not on standard PCI.

Plan the Number of Servers

Coherence is primarily a scale-out technology. The natural mode of operation is to span many servers (for example, 2-socket or 4-socket commodity servers). However, Coherence can also effectively scale-up on a small number of large servers by using multiple JVMs per server. Failover and failback are more efficient the more servers that are present in the cluster and the impact of a server failure is lessened. A cluster should contain a minimum of four physical servers to minimize the possibility of data loss during a failure. In most WAN configurations, each data center has independent clusters (usually interconnected by Extend-TCP). This increases the total number of discrete servers (four servers per data center, multiplied by the number of data centers).

Coherence is often deployed on smaller clusters (one, two or three physical servers) but this practice has increased risk if a server failure occurs under heavy load. As discussed in "Evaluate the Production Network's Speed for both UDP and TCP", Coherence clusters are ideally confined to a single switch (for example, fewer than 96 physical servers). In some use cases, applications that are compute-bound or memory-bound applications (as opposed to network-bound) may run acceptably on larger clusters.

Also, given the choice between a few large JVMs and a lot of small JVMs, the latter may be the better option. There are several production environments of Coherence that span hundreds of JVMs. Some care is required to properly prepare for clusters of this size, but smaller clusters of dozens of JVMs are readily achieved. Please note that disabling multicast (by using WKA) or running on slower networks (for example, 100Mbps Ethernet) reduces network efficiency and makes scaling more difficult.

Decide How Many Servers are Required Based on JVMs Used

The following rules should be followed in determining how many servers are required for reliable high availability configuration and how to configure the number of storage-enabled JVMs.

There must be more than two servers. A grid with only two servers stops being machine-safe as soon as several JVMs on one server are different than the number of JVMs on the other server; so, even when starting with two servers with equal number of JVMs, losing one JVM forces the grid out of machine-safe state. If the number of JVMs becomes unequal it may be difficult for Coherence to assign partitions in a way that ensures both equal per-member utilization as well as the placement of primary and backup copies on different machines. As a result, the recommended best practice is to use more than two physical servers.

For a server that has the largest number of JVMs in the cluster, that number of JVMs must not exceed the total number of JVMs on all the other servers in the cluster.

A server with the smallest number of JVMs should run at least half the number of JVMs as a server with the largest number of JVMs; this rule is particularly important for smaller clusters.

The margin of safety improves as the number of JVMs tends toward equality on all computers in the cluster; this is more of a general practice than the preceding rules.

5.5 Operating System Recommendations
Selecting an Operating System

Oracle tests on and supports the following operating systems:

Various Linux distributions

Sun Solaris

IBM AIX

Windows

Mac

OS/400

z/OS

HP-UX

Various BSD UNIX distributions

For commodity x86 servers, Linux distributions (Linux 2.6 kernel or higher) are recommended. While it is expected that most Linux distributions provide a good environment for running Coherence, the following are recommended by Oracle: Oracle Linux (including Oracle Linux with the Unbreakable Enterprise Kernel), Red Hat Enterprise Linux (version 4 or later), and Suse Linux Enterprise (version 10 or later).

Review and follow the instructions in Platform-Specific Deployment Considerations for the operating system on which Coherence is deployed.

Note:

The development and production operating systems may be different. Make sure to regularly test the target production operating system.

Avoid using virtual memory (paging to disk).

In a Coherence-based application, primary data management responsibilities (for example, Dedicated Cache Servers) are hosted by Java-based processes. Modern Java distributions do not work well with virtual memory. In particular, garbage collection (GC) operations may slow down by several orders of magnitude if memory is paged to disk. A properly tuned JVM can perform full GCs in less than a second. However, this may grow to many minutes if the JVM is partially resident on disk. During garbage collection, the node appears unresponsive for an extended period and the choice for the rest of the cluster is to either wait for the node (blocking a portion of application activity for a corresponding amount of time) or to consider the unresponsive node as failed and perform failover processing. Neither of these outcomes are a good option, and it is important to avoid excessive pauses due to garbage collection. JVMs should be configured with a set heap size to ensure that the heap does not deplete the available RAM memory. Also, periodic processes (such as daily backup programs) should be monitored to ensure that memory usage spikes do not cause Coherence JVMs to be paged to disk.

See also: "Swapping".

Increase Socket Buffer Sizes

The operating system socket buffers must be large enough to handle the incoming network traffic while your Java application is paused during garbage collection. Most versions of UNIX have a very low default buffer limit, which should be increased to 2MB.

See also: "Socket Buffer Sizes".

5.6 JVM Recommendations
During development, developers typically use the latest Oracle HotSpot JVM or a direct derivative such as the Mac OS X JVM.

The main issues related to using a different JVM in production are:

Command line differences, which may expose problems in shell scripts and batch files;

Logging and monitoring differences, which may mean that tools used to analyze logs and monitor live JVMs during development testing may not be available in production;

Significant differences in optimal garbage collection configuration and approaches to tuning;

Differing behaviors in thread scheduling, garbage collection behavior and performance, and the performance of running code.

Make sure that regular testing has occurred on the JVM that is used in production.

Select a JVM

Refer to "System Requirements" in Installing Oracle Coherence for the minimum supported JVM version.

Often the choice of JVM is also dictated by other software. For example:

IBM only supports IBM WebSphere running on IBM JVMs. Most of the time, this is the IBM "Sovereign" or "J9" JVM, but when WebSphere runs on Oracle Solaris/Sparc, IBM builds a JVM using the Oracle JVM source code instead of its own.

Oracle WebLogic and Oracle Exalogic include specific JVM versions.

Apple Mac OS X, HP-UX, IBM AIX and other operating systems only have one JVM vendor (Apple, HP, and IBM respectively).

Certain software libraries and frameworks have minimum Java version requirements because they take advantage of relatively new Java features.

On commodity x86 servers running Linux or Windows, use the Oracle HotSpot JVM. Generally speaking, the recent update versions should be used.

Note:

Test and deploy using the latest supported Oracle HotSpot JVM based on your platform and Coherence version.

Before going to production, a JVM vendor and version should be selected and well tested, and absent any flaws appearing during testing and staging with that JVM, that should be the JVM that is used when going to production. For applications requiring continuous availability, a long-duration application load test (for example, at least two weeks) should be run with that JVM before signing off on it.

Review and follow the instructions in Platform-Specific Deployment Considerations for the JVM on which Coherence is deployed.

Set JVM Options

JVM configuration options vary over versions and between vendors, but the following are generally suggested.

Using the -server option results in substantially better performance.

Using identical heap size values for both -Xms and -Xmx yields substantially better performance on Oracle HotSpot JVM and "fail fast" memory allocation.

Using Concurrent Mark and Sweep (CMS) garbage collection results in better garbage collection performance: -XX:+UseConcMarkSweepGC

Monitor garbage collection– especially when using large heaps: -verbose:gc, -XX:+PrintGCDetails, -XX:+PrintGCTimeStamps, -XX:+PrintHeapAtGC, -XX:+PrintTenuringDistribution, -XX:+PrintGCApplicationStoppedTime -XX:+PrintGCApplicationConcurrentTime

JVMs that experience an OutOfMemoryError can be left in an indeterministic state which can have adverse effects on a cluster. Configure JVMs to exit upon encountering an OutOfMemoryError instead of allowing the JVM to attempt recovery: On Linux, -XX:OnOutOfMemoryError="kill -9 %p"; on Windows, -XX:OnOutOfMemoryError="taskkill /F /PID %p".

Capture a heap dump if the JVM experiences an out of memory error: -XX:+HeapDumpOnOutOfMemoryError.

Plan to Test Mixed JVM Environments

Coherence is pure Java software and can run in clusters composed of any combination of JVM vendors and versions and Oracle tests such configurations.

Note that it is possible for different JVMs to have slightly different serialization formats for Java objects, meaning that it is possible for an incompatibility to exist when objects are serialized by one JVM, passed over the wire, and a different JVM (vendor, version, or both) attempts to deserialize it. Fortunately, the Java serialization format has been very stable for several years, so this type of issue is extremely unlikely. However, it is highly recommended to test mixed configurations for consistent serialization before deploying in a production environment.

See also:

"Deploying to Oracle HotSpot JVMs"

"Deploying to IBM JVMs"

5.7 Oracle Exalogic Elastic Cloud Recommendations
Oracle Exalogic and the Oracle Exalogic Elastic Cloud software provide a foundation for extreme performance, reliability, and scalability. Coherence has been optimized to take advantage of this foundation especially in its use of Oracle Exabus technology. Exabus consists of unique hardware, software, firmware, device drivers, and management tools and is built on Oracle's Quad Data Rate (QDR) InfiniBand technology. Exabus forms the high-speed communication (I/O) fabric that ties all Oracle Exalogic system components together. For detailed instructions on Exalogic, see Oracle Exalogic Enterprise Deployment Guide.

Oracle Coherence includes the following optimizations:

Transport optimizations

Oracle Coherence uses the Oracle Exabus messaging API for message transport. The API is optimized on Exalogic to take advantage of InfiniBand. The API is part of the Oracle Exalogic Elastic Cloud software and is only available on Oracle Exalogic systems.

In particular, Oracle Coherence uses the InfiniBand Message Bus (IMB) provider. IMB uses a native InfiniBand protocol that supports zero message copy, kernel bypass, predictive notifications, and custom off-heap buffers. The result is decreased host processor load, increased message throughput, decreased interrupts, and decreased garbage collection pauses.

The default Coherence setup on Oracle Exalogic uses IMB for service communication (transferring data) and for cluster communication. Both defaults can be changed and additional protocols are supported. For details on changing the default protocols, see "Changing the Reliable Transport Protocol".

Elastic data optimizations

The Elastic Data feature is used to store backing map and backup data seamlessly across RAM memory and devices such as Solid State Disks (SSD). The feature enables near memory speed while storing and reading data from SSDs. The feature includes dynamic tuning to ensure the most efficient use of SSD memory on Exalogic systems. For details about enabling and configuring elastic data, see Developing Applications with Oracle Coherence.

Coherence*Web optimizations

Coherence*Web naturally benefits on Exalogic systems because of the increased performance of the network between WebLogic Servers and Coherence servers. Enhancements also include less network usage and better performance by enabling optimized session state management when locking is disabled (coherence.session.optimizeModifiedSessions=true). For details about Coherence*Web context parameters, see Administering HTTP Session Management with Oracle Coherence*Web.

Consider Using Fewer JVMs with Larger Heaps

The IMB protocol requires more CPU usage (especially at lower loads) to achieve lower latencies. If you are using many JVMs, JVMs with smaller heaps (under 12GB), or many JVMs and smaller heaps, then consider consolidating the JVMs as much as possible. Large heap sizes up to 20GB are common and larger heaps can be used depending on the application and its tolerance to garbage collection. For details about heap sizing, see "JVM Tuning".

Disable Application Support for Huge (Large) Pages

Support for huge pages (also called large pages) is enabled in the Linux OS on Exalogic nodes by default. However, due to JVM stability issues, Java applications should not enable large pages. For example, do not use the -XX:+UseLargePages option when starting the JVM. Depending on the specific version of the JVM in use, large pages may be enabled with the JVM by default and thus the safest configuration is to explicitly disable them using -XX:-UseLargePages.

Note:

Updates to large page support is scheduled for a future release of Exalogic.

Changing the Reliable Transport Protocol

On Oracle Exalogic, Coherence automatically selects the best reliable transport available for the environment. The default Coherence setup uses the InfiniBand Message Bus (IMB) for service communication (transferring data) and for cluster communication unless SSL is enabled, in which case SDMBS is used. You can use a different transport protocol and check for improved performance. However, you should only consider changing the protocol after following the previous recommendations in this section.

Note:

The only time the default transport protocol may need to be explicitly set is in a Solaris Super Cluster environment. The recommended transport protocol is SDMB or (if supported by the environment) IMB.

The following transport protocols are available on Exalogic:

datagram – Specifies the use of UDP.

tmb – Specifies the TCP Message Bus (TMB) protocol. TMB provides support for TCP/IP.

tmbs – TCP/IP message bus protocol with SSL support. TMBS requires the use of an SSL socket provider. See Developing Applications with Oracle Coherence.

sdmb – Specifies the Sockets Direct Protocol Message Bus (SDMB). The Sockets Direct Protocol (SDP) provides support for stream connections over the InfiniBand fabric. SDP allows existing socket-based implementations to transparently use InfiniBand.

sdmbs – SDP message bus with SSL support. SDMBS requires the use of an SSL socket provider. See Developing Applications with Oracle Coherence.

imb (default on Exalogic) – InfiniBand message bus (IMB). IMB is automatically used on Exalogic systems as long as TCMP has not been configured with SSL.

To configure a reliable transport for all cluster (unicast) communication, edit the operational override file and within the <unicast-listener> element add a <reliable-transport> element that is set to a protocol:

Note:

By default, all services use the configured protocol and share a single transport instance. In general, a shared transport instance uses less resources than a service-specific transport instance.

<?xml version="1.0"?>
<coherence xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
   xmlns="http://xmlns.oracle.com/coherence/coherence-operational-config"
   xsi:schemaLocation="http://xmlns.oracle.com/coherence/
   coherence-operational-config coherence-operational-config.xsd">
   <cluster-config>
      <unicast-listener>
         <reliable-transport 
            system-property="coherence.transport.reliable">imb
         </reliable-transport>
      </unicast-listener>
   </cluster-config>
</coherence>
The coherence.transport.reliable system property also configures the reliable transport. For example:

-Dcoherence.transport.reliable=imb
To configure reliable transport for a service, edit the cache configuration file and within a scheme definition add a <reliable-transport> element that is set to a protocol. The following example demonstrates setting the reliable transport for a partitioned cache service instance called ExampleService:

Note:

Specifying a reliable transport for a service results in the use of a service-specific transport instance rather then the shared transport instance that is defined by the <unicast-listener> element. A service-specific transport instance can result in higher performance but at the cost of increased resource consumption and should be used sparingly for select, high priority services. In general, a shared transport instance uses less resource consumption than service-specific transport instances.

<?xml version="1.0"?>
<cache-config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
   xmlns="http://xmlns.oracle.com/coherence/coherence-cache-config"
   xsi:schemaLocation="http://xmlns.oracle.com/coherence/coherence-cache-config
   coherence-cache-config.xsd">

   <caching-scheme-mapping>
      <cache-mapping>
         <cache-name>example</cache-name>
         <scheme-name>distributed</scheme-name>
      </cache-mapping>
   </caching-scheme-mapping>
   
   <caching-schemes>
      <distributed-scheme>
         <scheme-name>distributed</scheme-name>
         <service-name>ExampleService</service-name>
         <reliable-transport>imb</reliable-transport>
         <backing-map-scheme>
           <local-scheme/>
         </backing-map-scheme>
         <autostart>true</autostart>
      </distributed-scheme>
   </caching-schemes>
</cache-config>
Each service type also has a system property that sets the reliable transport, respectively. The system property sets the reliable transport for all instances of a service type. The system properties are:

coherence.distributed.transport.reliable
coherence.replicated.transport.reliable
coherence.optimistic.transport.reliable
coherence.invocation.transport.reliable
coherence.proxy.transport.reliable
5.8 Security Recommendations
Ensure Security Privileges

The minimum set of privileges required for Coherence to function are specified in the security.policy file which is included as part of the Coherence installation. This file can be found in coherence/lib/security/security.policy. If using the Java Security Manager, these privileges must be granted in order for Coherence to function properly.

Plan for SSL Requirements

Coherence-based applications may chose to implement varying levels of security as required, including SSL-based security between cluster members and between Coherence*Extend clients and the cluster. If SSL is a requirement, ensure that all servers have a digital certificate that has been verified and signed by a trusted certificate authority and that the digital certificate is imported into the servers' key store and trust store as required. Coherence*Extend clients must include a trust key store that contains the certificate authority's digital certificate that was used to sign the proxy's digital certificate. See Securing Oracle Coherence for detailed instructions on setting up SSL.

5.9 Application Instrumentation Recommendations
Some Java-based management and monitoring solutions use instrumentation (for example, bytecode-manipulation and ClassLoader substitution). Oracle has observed issues with such solutions in the past. Use these solutions cautiously even though there are no current issues reported with the major vendors.

5.10 Coherence Modes and Editions
Select the Production Mode

Coherence may be configured to operate in either evaluation, development, or production mode. These modes do not limit access to features, but instead alter some default configuration settings. For instance, development mode allows for faster cluster startup to ease the development process.

The development mode is used for all pre-production activities, such as development and testing. This is an important safety feature because development nodes are restricted from joining with production nodes. Development mode is the default mode. Production mode must be explicitly specified when using Coherence in a production environment. To change the mode to production mode, edit the tangosol-coherence.xml (located in coherence.jar) and enter prod as the value for the <license-mode> element. For example:

...
<license-config>
   ...
   <license-mode system-property="coherence.mode">prod</license-mode>
</license-config>
...
The coherence.mode system property is used to specify the license mode instead of using the operational deployment descriptor. For example:

-Dcoherence.mode=prod
In addition to preventing mixed mode clustering, the license-mode also dictates the operational override file to use. When in eval mode the tangosol-coherence-override-eval.xml file is used; when in dev mode the tangosol-coherence-override-dev.xml file is used; whereas, the tangosol-coherence-override-prod.xml file is used when the prod mode is specified. A tangosol-coherence-override.xml file (if it is included in the classpath before the coherence.jar file) is used no matter which mode is selected and overrides any mode-specific override files.

Select the Edition

Note:

The edition switches no longer enforce license restrictions. Do not change the default setting (GE).

All nodes within a cluster must use the same license edition and mode. Be sure to obtain enough licenses for the all the cluster members in the production environment. The servers hardware configuration (number or type of processor sockets, processor packages or CPU cores) may be verified using ProcessorInfo utility included with Coherence. For example:

java -cp coherence.jar com.tangosol.license.ProcessorInfo
If the result of the ProcessorInfo program differs from the licensed configuration, send the program's output and the actual configuration as a support issue.

The default edition is grid edition. To change the edition, edit the operational override file and add an <edition-name> element, within the <license-config> element, that includes an edition name as defined in Table 5-1. For example:

...
<license-config>
   <edition-name system-property="tangosol.coherence.edition">EE</edition-name>
</license-config>
...
The coherence.edition system property is used to specify the license edition instead of using the operational deployment descriptor. For example:

-Dcoherence.edition=EE
Table 5-1 Coherence Editions

Value	Coherence Edition	Compatible Editions
GE

Grid Edition

RTC, DC

EE

Enterprise Edition

DC

SE

Standard Edition

DC

RTC

Real-Time Client

GE

DC

Data Client

GE, EE, SE

Note:

clusters running different editions may connect by using Coherence*Extend as a Data Client.

Ensuring that RTC Nodes do Not Use Coherence TCMP

Real-Time client nodes can connect to clusters using either Coherence TCMP or Coherence*Extend. If the intention is to use extend clients, disable TCMP on the client to ensure that it only connects to a cluster using Coherence*Extend. Otherwise, The client may become a member of the cluster. See Developing Remote Clients for Oracle Coherence for details on disabling TCMP communication.

5.11 Coherence Operational Configuration Recommendations
Operational configuration relates to cluster-level configuration that is defined in the tangosol-coherence.xml file and includes such items as:

Cluster and cluster member settings

Network settings

Management settings

Security settings

The operational aspects are typically configured by using a tangosol-coherence-override.xml file. See Developing Applications with Oracle Coherence for more information on specifying an operational override file.

The contents of this file often differs between development and production. It is recommended that these variants be maintained independently due to the significant differences between these environments. The production operational configuration file should be maintained by systems administrators who are far more familiar with the workings of the production systems.

All cluster nodes should use the same operational configuration override file and any node-specific values should be specified by using system properties. See Developing Applications with Oracle Coherence for more information on system properties. A centralized configuration file may be maintained and accessed by specifying a URL as the value of the coherence.override system property on each cluster node. For example:

-Dcoherence.override=/net/mylocation/tangosol-coherence-override.xml
The override file need only contain the operational elements that are being changed. In addition, always include the id and system-property attributes if they are defined for an element.

See Developing Applications with Oracle Coherence for a detailed reference of each operational element.

5.12 Coherence Cache Configuration Recommendations
Cache configuration relates to cache-level configuration and includes such things as:

Cache topology (<distributed-scheme>, <near-scheme>, and so on)

Cache capacities (<high-units>)

Cache redundancy level (<backup-count>)

The cache configuration aspects are typically configured by using a coherence-cache-config.xml file. See Developing Applications with Oracle Coherence for more information on specifying a cache configuration file.

The default coherence-cache-config.xml file included within coherence.jar is intended only as an example and is not suitable for production use. Always use a cache configuration file with definitions that are specific to the application.

All cluster nodes should use the same cache configuration descriptor if possible. A centralized configuration file may be maintained and accessed by specifying a URL as the value the coherence.cacheconfig system property on each cluster node. For example:

-Dcoherence.cacheconfig=/net/mylocation/coherence-cache-config.xml
Caches can be categorized as either partial or complete. In the former case, the application does not rely on having the entire data set in memory (even if it expects that to be the case). Most caches that use cache loaders or that use a side cache pattern are partial caches. Complete caches require the entire data set to be in cache for the application to work correctly (most commonly because the application is issuing non-primary-key queries against the cache). Caches that are partial should always have a size limit based on the allocated JVM heap size. The limits protect an application from OutOfMemoryExceptions errors. Set the limits even if the cache is not expected to be fully loaded to protect against changing expectations. See JVM Tuning for sizing recommendations. Conversely, if a size limit is set for a complete cache, it may cause incorrect results.

It is important to note that when multiple cache schemes are defined for the same cache service name, the first to be loaded dictates the service level parameters. Specifically the <partition-count>, <backup-count>, and <thread-count> subelements of <distributed-scheme> are shared by all caches of the same service. It is recommended that a single service be defined and inherited by the various cache-schemes. If you want different values for these items on a cache by cache basis then multiple services may be configured.

For partitioned caches, Coherence evenly distributes the storage responsibilities to all cache servers, regardless of their cache configuration or heap size. For this reason, it is recommended that all cache server processes be configured with the same heap size. For computers with additional resources multiple cache servers may be used to effectively make use of the computer's resources.

To ensure even storage responsibility across a partitioned cache the <partition-count> subelement of the <distributed-scheme> element, should be set to a prime number which is at least the square of the number of expected cache servers.

For caches which are backed by a cache store, it is recommended that the parent service be configured with a thread pool as requests to the cache store may block on I/O. Thread pools are also recommended for caches that perform CPU-intensive operations on the cache server (queries, aggreations, some entry processors, and so on). The pool is enabled by using the <thread-count> subelement of <distributed-scheme> element. For non-CacheStore-based caches more threads are unlikely to improve performance and should be left disabled.

Unless explicitly specified, all cluster nodes are storage enabled, that is, they act as cache servers. It is important to control which nodes in your production environment are storage enabled and storage disabled. The coherence.distributed.localstorage system property may be used to control storage, setting it to either true or false. Generally, only dedicated cache servers (including proxy servers) should have storage enabled. All other cluster nodes should be configured as storage disabled. This is especially important for short lived processes which may join the cluster to perform some work and then exit the cluster. Having these nodes as storage enabled introduces unneeded re-partitioning.

See Developing Applications with Oracle Coherence for a detailed reference of each cache configuration element.

5.13 Large Cluster Configuration Recommendations
Distributed caches on large clusters of more than 16 cache servers require more partitions to ensure optimal performance. The default partition count is 257 and should be increased relative to the number of cache servers in the cluster and the amount of data being stored in each partition. For details on configuring and calculating the partition count, see Developing Applications with Oracle Coherence.

The maximum packet size on large clusters of more than 400 cluster members must be increased to ensure better performance. The default of 1468 should be increased relative to the size of the cluster, that is, a 600 node cluster would need the maximum packet size increased by 50%. A simple formula is to allow four bytes per node, that is, maximum_packet_size >= maximum_cluster_size * 4B. The maximum packet size is configured as part of the coherence operational configuration file, see Developing Applications with Oracle Coherence for details on configuring the maximum packet size.

Multicast cluster communication should be enabled on large clusters that have hundreds of cluster members because it provides more efficient cluster-wide transmissions. These cluster-wide transmissions are rare, but when they do occur multicast can provide noticeable benefits. Multicast is enabled in an operational configuration file, see Developing Applications with Oracle Coherence for details.

5.14 Death Detection Recommendations
The Coherence death detection algorithms are based on sustained loss of connectivity between two or more cluster nodes. When a node identifies that it has lost connectivity with any other node, it consults with other cluster nodes to determine what action should be taken.

In attempting to consult with others, the node may find that it cannot communicate with any other nodes and assumes that it has been disconnected from the cluster. Such a condition could be triggered by physically unplugging a node's network adapter. In such an event, the isolated node restarts its clustered services and attempts to rejoin the cluster.

If connectivity with other cluster nodes remains unavailable, the node may (depending on well known address configuration) form a new isolated cluster, or continue searching for the larger cluster. In either case, the previously isolated cluster nodes rejoins the running cluster when connectivity is restored. As part of rejoining the cluster, the nodes former cluster state is discarded, including any cache data it may have held, as the remainder of the cluster has taken on ownership of that data (restoring from backups).

It is obviously not possible for a node to identify the state of other nodes without connectivity. To a single node, local network adapter failure and network wide switch failure looks identical and are handled in the same way, as described above. The important difference is that for a switch failure all nodes are attempting to re-join the cluster, which is the equivalent of a full cluster restart, and all prior state and data is dropped.

Dropping all data is not desirable and, to avoid this as part of a sustained switch failure, you must take additional precautions. Options include:

Increase detection intervals: The cluster relies on a deterministic process-level death detection using the TcpRing component and hardware death detection using the IpMonitor component. Process-level detection is performed within milliseconds and network or machine failures are detected within 15 seconds by default. Increasing these value allows the cluster to wait longer for connectivity to return. Death detection is enabled by default and is configured within the <tcp-ring-listener> element. See Developing Applications with Oracle Coherence for details on configuring death detection.

Persist data to external storage: By using a Read Write Backing Map, the cluster persists data to external storage, and can retrieve it after a cluster restart. So long as write-behind is disabled (the <write-delay> subelement of <read-write-backing-map-scheme>) no data would be lost if a switch fails. The downside here is that synchronously writing through to external storage increases the latency of cache update operations, and the external storage may become a bottleneck.

Decide on a cluster quorum: The cluster quorum policy mandates the minimum number of cluster members that must remain in the cluster when the cluster service is terminating suspect members. During intermittent network outages, a high number of cluster members may be removed from the cluster. Using a cluster quorum, a certain number of members are maintained during the outage and are available when the network recovers. See Developing Applications with Oracle Coherence for details on configuring cluster quorum.

Note:

To ensure that Windows does not disable a network adapter when it is disconnected, add the following Windows registry DWORD, setting it to 1:HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters\DisableDHCPMediaSense. This setting also affects static IPs despite the name.

Add network level fault tolerance: Adding a redundant layer to the cluster's network infrastructure allows for individual pieces of networking equipment to fail without disrupting connectivity. This is commonly achieved by using at least two network adapters per computer, and having each adapter connected to a separate switch. This is not a feature of Coherence but rather of the underlying operating system or network driver. The only change to Coherence is that it should be configured to bind to the virtual rather then physical network adapter. This form of network redundancy goes by different names depending on the operating system, see Linux bonding, Solaris trunking and Windows teaming for further details.

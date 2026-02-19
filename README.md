### For the full source code, please request a zip file from mfinci67@gmail.com

# Router Feed Server (Low-Latency / Market Data Handler, In-Memory Order Book)

As only designer and implementer, I professionally implemented a project collecting KPI and troubleshooting data millions routers and cable modems in a high frequency and applied logics for troubleshooting and business decision perspective.

Now personally, implemented the project simulating a market-data feed handler using an “ISP router telemetry feed” analogy.

The main goal is to demonstrate realistic hot-path optimizations (copy avoidance, cache friendliness, bounded lock-free queues, memory pools, propagated custom memory allocators, slab pool, huge pages / memory locking, thread pinning), while keeping the code readable and modular. While it is open to improve more in multiple ways as described at the end.

- VM Ubuntu simulated as routers, act like exchange feed publishers sending binary updates via UDP
- one thread simulates UDP (Multicast) Server and TCP (Gap recovery) receiving binary feeds from routers and publishes raw binary frames into an lock-free SPSC and slab pool data path. It uses epoll ET I/O. 
- one thread receives raw feed from shared SPSC and slab pool and parses and updates in-memory engine book, router/order books, asks and bids maps. It also implements per-router sequence numbers and gap detection if any. If so, it populates a binary message format and publish another lock-free SPSC and slab pool .Moreover if it detects very high cpu usage, similarly, it populates binary reboot message and then publishes similarly.
- one thread receives from this SPSC and pool, sends control-plane messages back to routers over TCP (gap recovery, reboot and also start/stop)


High-level mapping:

- **Router MAC** ↔ symbol
- **CPU usage (%)** ↔ price level
- **High CPU** ↔ asks (descending levels)
- **Low CPU** ↔ bids (ascending levels)
- **UDP telemetry** ↔ market data packets
- **Per-router sequence numbers + gap detection** ↔ feed sequence + gap-fill

## Key Optimizations
### 1. Hot Path: “No Copy, No Lock” UDP Receive Using a Packet Slab Pool + SPSC
- Incoming UDP payload bytes are written **directly into reusable slabs** , not into a per-message inline array.
- The SPSC ring carries small metadata only; payload bytes live in a recycled shared buffer pool.
- Hot path's thread is pinned to core asmd isolcpus and  Real-Time Scheduling.
- Branch Prediction [[likely]]/[[unlikely]] or __builtin_expect(!!(x), 1) ...0)
- 
### 2. Memory Management
- Fixed-size block pool with pre-allocated objects
- Custom pool allocators for STL containers with backward capabilities (C++11 can use it)
- Allocator Propagation for Nested Containers (scoped_allocator_adapter)
- Object deletion is to assign new values to the existing object.
- Zero dynamic allocations in hot path, unless pool exhausted
- Huge Page Region and Best-Effort Memory lock to physical memory and preveting swapping.
- Reduces TLB misses

 ### 3. Lock-Free SPSC Queues (Data Plane + Control Plane)
- Single producer / single consumer ⇒ no CAS loops needed
- Fixed-size bounded ring (power-of-2 capacity)
- head and tail are aligned to cache lines (alignas(64)) to reduce false sharing
- O(1) push/pop
- Acquire/Release ordering is used at the producer/consumer boundary
- USE CASE
- Pass raw receive metadata across threads without locks
- Pass control events (gap recovery / stop / reboot) without contention

### 4. Networking Details (UDP + epoll ET + TCP control plane)
- UDP receive path (UdpServer, IoThread)
- SO_RCVBUF (best-effort; sysctl may cap)
- SO_BUSY_POLL when available (best-effort demo knob)
- TCP_NODELAY to disable Nagle's algorithm
- SO_REUSEPORT for load balancing in kernel.
- IO thread uses epoll edge-triggered (EPOLLET) and drains recvfrom() until EAGAIN
- Avoids repeated epoll wakeups for a burst of packets
- Network RX & TX buffers Tuning (sysctl -w net.core.r/wmem_max <higher#> )
 TCP (control plane) 
- TcpServer accepts inbound TCP connections (optional optimization) 
- Keeps a small registry mapping router_ip → accepted_fd
- enables the control thread to reuse an existing connection instead of connecting each time
- connect-send-close for a control event when no accepted connection exists
- control plane is intentionally not in the hot path

### 5. Compiler Optimizations
- -O3 -march=native -mtune=native
-flto                    # Link-time optimization
-fno-rtti                # No runtime type information
-funroll-loops           # Loop unrolling
-fno-exceptions          # No exception handling overhead
-finline-functions       # Aggressive inlining
-fomit-frame-pointer     # Omit frame pointers
-ffast-math              # Aggressive floating-point optimizations

### 6.  Next Optimizations
- NUMA Awareness; Allocates memory on the same NUMA node as the CPU and reduces cross-NUMA memory access latency
- DPDK for kernel bypass
- CPU Isolation (via kernel parameters): isolcpus=2,3 nohz_full=2,3 rcu_nocbs=2,3 (given random# for demo)
  
### CSV (previewable): [router_feed_binary_protocol.csv](router_feed_binary_protocol.csv) 
 
### Quick Start
- mkdir -p build && cd build
- cmake .. -DCMAKE_BUILD_TYPE=Release
- cmake --build . -j
- ./router_feed_server ./tools/server_config.csv
- **Router Feed publisher** (simulated exchange market data provider) and control message receiver
- ./router_sim --server 127.0.0.1 --udp-port 9000 --tcp-port 9001 --rate 2000 --gap-every 2000

#### Repository Layout (Key Files)
##### Protocol & data model
- include/protocol.hpp — binary protocol parsing/building
- include/router_book.hpp, src/router_book.cpp — per-router state & gap sequence + high cpu detection 
##### Hot-path data movement
- include/spsc_ring.hpp — lock-free SPSC ring
- include/packet_slab_pool.hpp — reusable packet buffers (slab pool)
- include/raw_items.hpp — metadata pushed through SPSC ring (no payload copy)
##### Memory management
- include/fixed_block_pool.hpp — FixedBlockPool + PoolAllocator (fallback to default allocator + nested propagation)
- include/huge_page_region.hpp, src/huge_page_region.cpp — huge-page backed region, memorylock
##### Networking
- include/net/udp_server.hpp, src/net/udp_server.cpp
- include/net/tcp_server.hpp, src/net/tcp_server.cpp
- include/net/tcp_client.hpp, src/net/tcp_client.cpp
##### Threads
- include/threads/io_thread.hpp, src/threads/io_thread.cpp — epoll ET, UDP recv, TCP accept
- include/threads/parser_thread.hpp, src/threads/parser_thread.cpp — parse + update + emit control events
- include/threads/gap_thread.hpp, src/threads/gap_thread.cpp — TCP control sender
- include/threads/thread_util.hpp, src/threads/thread_util.cpp — pinning + RT scheduling helpers
##### Config
- include/config_loader.hpp, src/config_loader.cpp — CSV config loader
- tools/server_config.csv — example config

##### High Level Design Diagram
Routers  --------------------->  [ IO Thread ] 
- UDP Feed Binary
- And (if any) GAP Recovery + control msg Binary 
- accepts inbound ctrl conns, EPoll ET
      
[ IO Thread ] ---->  [ Parser Thread ]
 SPSC raw_ring (byte metadata) + slab pool
   
[ Parser Thread ]
- read + parse bytes
- update RouterBook
- detect seq gaps
- detect very high cpu
- emit CtrlEvent (SPSC)

[ Parser Thread ] -------->  [ Control Thread ]
- SPSC raw_ring (byte metadata) + slab pool

[ Control Thread ]
- Populates Gap Recovery Request + Reboot + Start + Stop Binary Messages
- reuse accepted conn if present
- else connect-send-close 

[ Control Thread ]  -------------> Router via TCP

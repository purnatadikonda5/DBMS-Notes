# DBMS Architecture: Clustering, Replica Sets & CDNs

Hello there! Welcome back to the classroom. Today we are exploring the absolute bedrock of modern, large-scale distributed systems. 

Imagine a single database server acting like a solo librarian. If one person asks for a book, they can handle it. But if a million people ask at once, or if the librarian gets sick, the whole system collapses. This is the **Single Point of Failure (SPOF)** and the bottleneck that clustering, replica sets, and CDNs solve.

By the end of this masterclass, you will understand exactly how modern databases distribute massive workloads, keep data perfectly synced across the globe, handle simultaneous user conflicts, and deliver content at the speed of light.

---

## ⭐ Part 1: The Basics (Clusters vs. Replica Sets) [Basic Interview Focus]
While the terms are often used interchangeably, they have distinct roles in system design.

> [!NOTE]
> **What is a Node?**
> A "node" is just a single server. In the modern cloud, it is usually a Virtual Machine or a Container with its own dedicated CPU, memory, and storage, running a single instance of the database software.

### 1. Database Cluster
A Database Cluster is a group of interconnected servers (nodes) that act as a single system. Users connect to the cluster thinking it's one giant machine, but behind the scenes, multiple servers are working together.

### 2. Replica Set
A Replica Set is a specific type of cluster where multiple nodes hold the exact same copy of your data. Its primary job is **High Availability**. If one server crashes, another immediately takes its place.

In a standard replica set, nodes are given specific roles:
* **Primary Node (Leader/Master):** The boss. It is usually the only node allowed to accept new data or changes (Writes).
* **Secondary Nodes (Followers/Slaves):** These nodes maintain a strict copy of the Primary's data. They handle read requests (like "show me my profile") but cannot accept writes.

```mermaid
flowchart TD
    subgraph Cluster [Database Replica Set]
        direction TB
        Primary[("⭐ Primary Node\n(Receives Writes)\n[Exact DB Copy]")]:::primary
        Sec1[("Secondary Node\n(Receives Reads)\n[Exact DB Copy]")]:::secondary
        Sec2[("Secondary Node\n(Receives Reads)\n[Exact DB Copy]")]:::secondary
        
        Primary -. "Syncs Data" .-> Sec1
        Primary -. "Syncs Data" .-> Sec2
    end

    classDef primary fill:#f3e5f5,stroke:#8e24aa,color:#000,stroke-width:2px;
    classDef secondary fill:#e8f5e9,stroke:#43a047,color:#000,stroke-width:2px;
```

## ⭐ Part 2: The Load Balancer & The Traffic Cops [Basic Interview Focus]
If you have five servers in a cluster, how does a user's phone know which one to talk to? It doesn't. The traffic is managed by routing layers.

It's critical to understand the separation between the Application Tier and the Database Tier.

> [!IMPORTANT]
> **Stateless vs. Stateful**
> * **Application Servers (Backend)** are **Stateless**. They run logic (Node.js, Python), calculate results, and forget them. You can spin up 100 identical App Servers and they don't need to sync with each other.
> * **Database Servers** are **Stateful**. They are the permanent memory. Clustering them requires complex replication and synchronization protocols.

### How Traffic is Routed

1. **The App Load Balancer:** Sits at the edge of the backend network and distributes incoming user HTTP requests across the stateless Application Servers.
2. **The Database Routing (Two Methods):**
   * **Method A (Smart Database Driver):** The Application Server code uses an internal driver that maintains two separate connection pools: one specifically pointing to the Primary node for Writes, and one pointing to a DB Load Balancer for Reads.

```mermaid
flowchart TD
    App["Application Server\n(with Smart DB Driver)"]
    
    subgraph Connection_Pools [Driver Connection Pools]
        direction LR
        WritePool["Write Connection Pool"]
        ReadPool["Read Connection Pool"]
    end
    
    App --> WritePool
    App --> ReadPool
    
    WritePool -- "Direct Route" --> Primary["Primary Node"]:::primary
    ReadPool -- "Distributes load" --> DBLB["DB Load Balancer"]:::lb
    
    DBLB --> Sec1["Secondary Node"]:::secondary
    DBLB --> Sec2["Secondary Node"]:::secondary
    
    classDef primary fill:#f3e5f5,stroke:#8e24aa,color:#000,stroke-width:2px;
    classDef secondary fill:#e8f5e9,stroke:#43a047,color:#000,stroke-width:2px;
    classDef lb fill:#fff3e0,stroke:#fb8c00,color:#000,stroke-width:2px;
    class Primary primary
    class Sec1,Sec2 secondary
    class DBLB lb
```
   * **Method B (Smart Database Proxy):** The App Server just sends all queries to a single endpoint: a **Database Proxy** (like ProxySQL). The proxy sits in front of the database cluster, parses the SQL in real-time, and automatically routes `UPDATE`/`INSERT` commands to the Primary and `SELECT` commands to the Secondaries.

```mermaid
flowchart TD
    User["User (Browser)"] --> ALB["App Load Balancer"]
    ALB --> App1["App Server 1 (Stateless)"]
    ALB --> App2["App Server 2 (Stateless)"]
    
    App1 --> DBProxy["Database Proxy / DB Load Balancer"]
    App2 --> DBProxy
    
    subgraph Database_Cluster [Database Cluster]
        direction TB
        DBProxy -- "Writes (UPDATE/INSERT)" --> Primary["Primary Node (Read/Write)"]
        DBProxy -- "Reads (SELECT)" --> Sec1["Secondary Node (Read Only)"]
        DBProxy -- "Reads (SELECT)" --> Sec2["Secondary Node (Read Only)"]
    end
    
    %% Styles
    classDef default fill:#fafafa,stroke:#e0e0e0,color:#333
    classDef root fill:#f3e5f5,stroke:#8e24aa,color:#000,stroke-width:2px;
    classDef intermediate fill:#e3f2fd,stroke:#1e88e5,color:#000,stroke-width:2px;
    classDef leaf fill:#e8f5e9,stroke:#43a047,color:#000,stroke-width:2px;
    classDef storage fill:#fff3e0,stroke:#fb8c00,color:#000,stroke-width:2px;
    
    class User root;
    class ALB,App1,App2 intermediate;
    class DBProxy,Primary,Sec1,Sec2 leaf;
```

## Part 3: Keeping Data Synced (Replication)
When the Primary node receives a new piece of data, it must inform the Secondary nodes. Every action the Primary takes is written to a **Write-Ahead Log (WAL)** or Oplog. The Secondary nodes constantly read this log and apply the exact same actions to their own data.

This sync happens in one of two ways:

### 1. Synchronous Replication (Maximum Safety)
The Primary gets the data, sends it to the Secondaries, and **waits** for them to say "Got it!" before telling the user "Success."
* **Pros:** Zero data loss. Everyone is always perfectly in sync.
* **Cons:** Slow. If a Secondary server is lagging or offline, the user has to wait.

### 2. Asynchronous Replication (Maximum Speed)
The Primary gets the data, immediately tells the user "Success!", and then sends the update to the Secondaries in the background.
* **Pros:** Lightning fast.
* **Cons:** Replication Lag. For a few milliseconds, the Secondaries are out of date (serving stale data).

## Part 4: Solving the "Stale Read" Problem
In asynchronous replication, if you update your profile picture (Write to Primary) and immediately refresh the page (Read from Secondary), you might see your old picture because the Secondary hasn't synced yet. 

To fix this, systems use **Read-After-Write Consistency**. The system "remembers" that you just made an edit and actively routes your subsequent reads to the Primary node for a few seconds.

### How is the Session ID Stored Efficiently?
Systems use a blazing fast in-memory cache like **Redis**.

> [!TIP]
> **The Redis TTL Trick:** 
> Redis uses a flat Key-Value structure with a Time-To-Live (TTL), making the lookup O(1) regardless of millions of users. 

**The Exact Flow:**
1. **The Write:** You update your profile. The App Server executes the write on the Primary database.
2. **The Cache Insert:** The App Server immediately sends a command to Redis: `SET user_123_write_flag TRUE EX 5`. The `EX 5` tells Redis to delete this key after exactly 5 seconds.
3. **The Read:** You refresh the page. The App Server asks Redis: `GET user_123_write_flag`.
4. **The Route:**
   * If Redis returns `TRUE` (within 5 seconds), the App Server routes your read strictly to the Primary Node. You see your updated data.
   * If Redis returns `NULL` (after 5 seconds), the App Server confidently routes your read to the Secondary Node.

```mermaid
flowchart TD
    User["User Request"] --> App["Application Server"]
    App -- "Checks Cache" --> Redis[("Redis Cache\n(5 Second Timer)")]
    
    Redis -- "All Writes & Reads < 5s\n(Timer Active)" --> Primary["Primary Node\n(Fresh Data)"]:::primary
    Redis -- "Reads > 5s\n(Timer Expired)" --> Secondary["Secondary Node\n(Replicated Data)"]:::secondary
    
    classDef primary fill:#f3e5f5,stroke:#8e24aa,color:#000,stroke-width:2px;
    classDef secondary fill:#e8f5e9,stroke:#43a047,color:#000,stroke-width:2px;
```

> [!IMPORTANT]
> Alternatively, a **Smart Database Proxy** can do this entirely on its own by intercepting the SQL, logging recently modified User IDs in its own memory, and overriding its load-balancing rules for those IDs for a few seconds.

```mermaid
flowchart TD
    App["Application Server"] --> Proxy["Smart Database Proxy\n(e.g., ProxySQL)"]:::proxy
    
    subgraph Proxy_Internal [Inside the Proxy]
        direction LR
        SQLParser["SQL Parser"]
        ProxyCache[("Internal Cache\n(Recent Writers)")]
        Proxy --> SQLParser
        SQLParser <--> ProxyCache
    end
    
    SQLParser -- "Recent writer OR Write query" --> Primary["Primary Node"]:::primary
    SQLParser -- "Normal Read query" --> Sec1["Secondary Node"]:::secondary
    SQLParser -- "Normal Read query" --> Sec2["Secondary Node"]:::secondary
    
    classDef primary fill:#f3e5f5,stroke:#8e24aa,color:#000,stroke-width:2px;
    classDef secondary fill:#e8f5e9,stroke:#43a047,color:#000,stroke-width:2px;
    classDef proxy fill:#e3f2fd,stroke:#1e88e5,color:#000,stroke-width:2px;
```

## Part 5: Multi-Primary Clusters & Conflict Resolution
In massive global systems (like Amazon or Google), relying on a single Primary node is too slow. They use a **Multi-Primary (Active-Active)** setup where multiple servers can accept writes simultaneously.

But what if a user in India and a user in the US update the exact same data simultaneously on two different Primary nodes?

### 1. The Multi-Primary Conflict Problem
If a user in India and a user in the US update the exact same data simultaneously on two different Primary nodes, you get a conflict. Simple rules like **Last Write Wins (LWW)** are dangerous here because one user's update is entirely lost.

To solve this, databases use two different strategies depending on how critical the data is:

### 2. Strategy A: Vector Clocks & CRDTs (Eventual Consistency)
For non-critical or mergeable data (like a YouTube "like" counter or Google Docs), databases allow the conflict to happen and resolve it after the fact.
* **Vector Clocks (The Detector):** Because server clocks aren't perfectly synced, databases use an array tracking the history of changes (e.g., `[India_Updates: 1, US_Updates: 1]`). If the database detects a "Branch", it knows a simultaneous edit occurred.
* **CRDTs (The Resolver):** Conflict-free Replicated Data Types automatically merge the math. Instead of sending the final state (`11`), the nodes send the operation (`INCREMENT BY 1`), resulting in `12`. 
* **Fallback:** If it cannot be merged by math, it prompts the user (e.g., "Merge Conflict" in Git).

### 3. Strategy B: Global Locks / Two-Phase Commit (Strict Consistency)
For critical data where conflicts are unacceptable (like reserving the last flight seat), relying on CRDTs isn't enough. Modern global databases (like Google Spanner) solve this using **Distributed Global Locks** (Two-Phase Commit).
* Before the India node executes a write, it sends a global lock request to the other primary nodes.
* It waits until it receives acknowledgements confirming the row is locked.
* While locked, if the US user tries to book the seat, the US node tells them to wait.
* Once the India node writes the data, it releases the lock. This perfectly prevents conflicts before they even happen.

## Part 6: Coordinators & Distributed Deadlocks
To prevent conflicts on absolutely critical data (like booking a flight seat), databases use **Distributed Locks**. But if Node A locks and asks B, and Node B locks and asks A, you get a **Distributed Deadlock** and the system freezes.

> [!WARNING]
> **Quorums:** Databases never wait for *all* nodes to agree (too slow). They only need a strict majority (e.g., 3 out of 5). This is called a Quorum.

### Breaking the Tie
There are two ways to break a distributed deadlock:

**Method 1: The Consensus Coordinator (ZooKeeper / etcd)**
Enterprise systems use a specialized micro-cluster (the Coordinator) as the single referee. 
* It uses an **Odd Number Rule** (3, 5, or 7 nodes) to ensure a majority vote and prevent "split-brain" scenarios.
* Nodes ask the Coordinator to create a lock file (`CREATE FILE: /locks/User_123`).
* Because the Coordinator executes atomically, it is physically impossible for two nodes to create the file simultaneously. The first one gets the lock; the second gets a "File exists" error and must wait.
* **Ephemeral Keys (Leases):** The lock is tied to a heartbeat. If the node holding the lock dies, the Coordinator deletes the lock after a few seconds so the system doesn't freeze permanently.
* **The Magic of the Majority:** Ultimately, the Coordinator is just a voting system. If a lock request gets a majority of votes from the internal Coordinator nodes, the action is performed. Because there can mathematically only be *one* majority in an odd-numbered cluster, any competing request instantly fails to get enough votes and is forced to wait.

**Method 2: Timestamp Tie-Breakers (Peer-to-Peer)**
Nodes stamp transactions with precise timestamps. If a deadlock occurs, algorithms like **Wait-Die** kick in:
* If an *older* transaction needs a lock held by a *younger* one, it is allowed to wait.
* If a *younger* transaction needs a lock held by an *older* one, the younger one must die (abort and retry).

## Part 7: The Global Edge (Content Delivery Networks)
CDNs solve the one problem your database cluster cannot fix: the speed of light. 

A CDN is a globally distributed network of massive cache servers (Edge Nodes / PoPs) located geographically close to the users.

### The Request Flow (Dynamic vs Static)

```mermaid
flowchart TD
    User["👨‍💻 User\n(India)"]
    
    subgraph CDN_Layer [🌍 Global CDN Layer - Edge Nodes]
        direction LR
        Mumbai["📍 Edge Node\n(Mumbai)"]:::edge
        Tokyo["📍 Edge Node\n(Tokyo)"]:::edge
        London["📍 Edge Node\n(London)"]:::edge
    end

    subgraph Origin_Layer [🏢 Origin Data Center - USA]
        direction TB
        App["⚙️ Application Server\n(API Logic)"]:::backend
        DB[("🗄️ Database\n(Dynamic JSON)")]:::database
        S3[("🖼️ Object Storage\n(Static Files)")]:::storage
        
        App <--> DB
    end

    User -->|DNS routes to nearest PoP| Mumbai

    %% Path 1: Dynamic Data
    Mumbai -. "1. API Request (GET /api/user)\n⚠️ Bypasses Cache" .-> App
    App -. "Returns JSON" .-> Mumbai

    %% Path 2: Static Data
    Mumbai == "2. Image Request (GET /logo.png)" ==> CacheCheck{"Has File?"}
    
    CacheCheck -- "✅ YES (Cache Hit)" --> Mumbai
    CacheCheck -- "❌ NO (Cache Miss)" --> S3
    S3 -- "Returns Image" --> Mumbai
    
    Mumbai -- "Delivers Response to User" --> User
    
    classDef edge fill:#e3f2fd,stroke:#1e88e5,color:#000,stroke-width:2px;
    classDef backend fill:#fff3e0,stroke:#fb8c00,color:#000,stroke-width:2px;
    classDef database fill:#f3e5f5,stroke:#8e24aa,color:#000,stroke-width:2px;
    classDef storage fill:#e8f5e9,stroke:#43a047,color:#000,stroke-width:2px;
```

The secret to CDNs is that they do not cache everything. They strictly separate **Dynamic API Data** from **Static Assets**.

Here is the exact step-by-step flow of how the CDN handles a user loading their profile:

**Path 1: Fetching the Data (The API Call)**
1. The user's browser requests their profile data (`GET /api/user`).
2. The request hits the nearest **CDN Edge Node (Mumbai)**.
3. Because it is an `/api/` route, the CDN knows this is dynamic data. It **bypasses the cache** and forwards the request across the ocean.
4. The **Application Server (USA)** processes the logic, queries the **Database**, and sends back a lightweight JSON response: `{"name": "John", "image": "/img/user123.jpg"}`.

**Path 2: Fetching the Image (The Static Asset)**
1. The browser reads the JSON and realizes it needs to display an image. It makes a second request: `GET /img/user123.jpg`.
2. This request hits the **Mumbai CDN** again. Because it's an image file, the CDN looks inside its local cache.
3. **If Cache Hit:** The CDN already has the image in memory. It sends it back to the user instantly. The USA server does zero work.
4. **If Cache Miss:** The CDN does not have it. It fetches the image from the **USA Object Storage (S3)**, saves a copy locally in Mumbai for the next person, and sends it to the user.

> [!IMPORTANT]
> By offloading 90% of the heavy static traffic (images, videos, CSS) to the CDN, your Origin Database is protected from crashing during massive traffic spikes.

### Cache Invalidation (Handling Stale Assets)
If you update a logo, how do you force the CDN to drop the old one?
1. **Time-To-Live (TTL):** The asset expires automatically after X hours.
2. **Cache Purging:** You send an API command to the CDN to manually delete the specific file across the globe.
3. **Versioning (Cache-Busting) [Best Practice]:** Never overwrite a file. Upload the new logo as `logo_v2.png` and update your database/HTML to point to the new filename. The CDN has never seen `v2`, so it is forced to fetch the fresh copy. The old `v1` simply rots away in the cache until its TTL expires.



## Appendix: One-Line Definitions to Remember
| Term | Definition |
| :--- | :--- |
| **Cluster** | Interconnected servers acting as a single system. |
| **Replica Set** | A cluster specifically holding identical copies of data for High Availability. |
| **Primary Node** | The master node that accepts write operations. |
| **Secondary Node** | A read-only node that asynchronously synchronizes with the Primary. |
| **Load Balancer** | A router that distributes traffic to prevent overwhelming a single server. |
| **Database Proxy** | A smart load balancer specifically for SQL queries. |
| **Write-Ahead Log (WAL)** | The log of changes the Primary sends to Secondaries to keep them synced. |
| **Vector Clock** | An array tracking version history to detect simultaneous edit conflicts. |
| **CRDT** | A data structure that mathematically resolves merge conflicts automatically. |
| **Coordinator (ZooKeeper)** | A specialized micro-cluster that issues locks to prevent deadlocks. |
| **Quorum** | A strict majority vote required for a distributed system to make a decision. |
| **CDN (Edge Node)** | A global network of cache servers physically close to users. |
| **Origin Server** | Your actual backend and database servers acting as the single source of truth. |
| **Cache Busting** | Forcing a CDN to fetch fresh data by changing the filename (versioning). |

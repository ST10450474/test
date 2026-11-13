# Smart-X IoT Mesh Ecosystem (Final PoE)

> PROG7312 Portfolio of Evidence: Programming 3B, Advanced Application Development
> Student: ST10450474

Smart-X is a simulated Internet of Things (IoT) monitoring ecosystem for distributed ESP32 sensors such as hydroponic moisture probes, smart power meters and relay/pump actuators. It ingests and validates telemetry, processes it in real time, and models the wireless mesh the sensors form so that routes, failures and power use can be analysed.

| Part | Module | What it adds |
|------|--------|--------------|
| 1 | Sensor Data Ingestion and Telemetry | Sensor registration, validation, encrypted file attachments, alert triage, engagement features (daily challenge, streaks, missions) |
| 2 | Real-Time Command Stream and History | Queues, priority queue, stack-based undo, dictionaries, HashSet fault registry, predictive suggestions |
| **Final** | **Network Topology and Mesh Routing** | **AVL tree history queries, mesh graph, BFS routing and self-healing, min-heap, Prim's minimum spanning tree** |

---

## Table of Contents

1. [Technology Stack and Project Structure](#1-technology-stack-and-project-structure)
2. [Prerequisites](#2-prerequisites)
3. [How to Compile and Run](#3-how-to-compile-and-run)
4. [How to Use the Application](#4-how-to-use-the-application)
5. [API Reference](#5-api-reference)
6. [How to Test Each Feature](#6-how-to-test-each-feature)
7. [Data Structure Explanations (Network Topology)](#7-data-structure-explanations-network-topology)
8. [Data Structures from Parts 1 and 2](#8-data-structures-from-parts-1-and-2)
9. [Troubleshooting](#9-troubleshooting)
10. [Demo Videos](#10-demo-videos)
11. [References](#11-references)
12. [Updates Made From Lecturer Feedback](#12-updates-made-from-lecturer-feedback)

---

## 1. Technology Stack and Project Structure

| Layer | Technology |
|-------|-----------|
| Backend | ASP.NET Core Minimal API, .NET 10 |
| Frontend | Blazor WebAssembly (.NET 10) |
| Shared library | `SmartX.Shared` (models, DTOs, validation, custom collections) |
| Containers | Docker and Docker Compose (API on its own image, client served by nginx) |

```text
.
├── SmartX.Api/                  Backend Web API
│   ├── Data/                    Stores, pipeline, mesh network, history tree, pattern engine
│   │   ├── MeshNetwork.cs       Mesh service: graph, BFS routing, MST, failures
│   │   ├── TelemetryHistoryTree.cs  Per-sensor AVL trees of timestamped history
│   │   ├── FileVault.cs         AES-256-GCM encryption for uploaded attachments
│   │   └── ...
│   ├── Endpoints/               Minimal API endpoint groups
│   │   └── TopologyEndpoints.cs /api/mesh/* and /api/history/*
│   ├── Dockerfile
│   └── Program.cs
├── SmartX.Client/               Blazor WebAssembly frontend
│   ├── Pages/                   Home, Dashboard, SensorIngestion, CommandStream, NetworkTopology, About
│   ├── Services/                SmartXApiClient (typed HTTP client), ApiSettings (validated connection settings), page-state holders
│   ├── Dockerfile / nginx.conf
│   └── wwwroot/appsettings.json ApiBaseUrl setting
├── SmartX.Shared/               Referenced by both API and client
│   ├── Collections/             AvlTree, MinHeap, MeshGraph (+ BFS, Prim), queues, ring buffer
│   ├── Dtos/  Models/  Validation/  Processing/
├── docker-compose.yml
├── FEEDBACK_UPDATES.md          What changed after Part 1 and 2 feedback
└── SmartX.slnx
```

---

## 2. Prerequisites

- **Git**
- **Either**
  - **Docker Desktop** (recommended; this is how the project was built and tested), **or**
  - **.NET 10 SDK** (to run without Docker). Check with `dotnet --version`.
- A modern browser (Chrome, Edge or Firefox).
- Free local ports **5216** (API) and **5274** (client).

---

## 3. How to Compile and Run

### 3.1 Clone the repository

```bash
git clone https://github.com/EMKNDN/emkndn-prog7312-g2-2026-prog7312-poe-st10450474.git
cd emkndn-prog7312-g2-2026-prog7312-poe-st10450474
```

### 3.2 Option 1: Docker (recommended)

Make sure Docker Desktop is running, then from the repository root:

```bash
docker compose up --build
```

| Service | URL |
|---------|-----|
| Client (Blazor UI) | http://localhost:5274 |
| API | http://localhost:5216 |
| API health check | http://localhost:5216/api/ping |

The first build downloads the .NET SDK and runtime images, so it takes a few minutes. Stop the stack with `Ctrl + C`, then:

```bash
docker compose down
```

### 3.3 Option 2: Without Docker (.NET 10 SDK)

**Step 1: restore dependencies and compile** (from the repository root):

```bash
dotnet restore
dotnet build
```

The build should finish with `0 Error(s)`.

**Step 2: start the backend API** (first terminal):

```bash
cd SmartX.Api
dotnet run --launch-profile http
```

The API listens on http://localhost:5216. Confirm it with http://localhost:5216/api/ping.

**Step 3: start the client** (second terminal):

```bash
cd SmartX.Client
dotnet run --launch-profile http
```

Open http://localhost:5274.

> Start the API **before** the client. The client calls the API as soon as a page loads.

### 3.4 Configuration

| Setting | File | Default | Purpose |
|---------|------|---------|---------|
| `ApiBaseUrl` | `SmartX.Client/wwwroot/appsettings.json` | `http://localhost:5216/` | Where the client finds the API. Must be a valid http(s) address; if it is missing or invalid the client falls back to the default and shows a yellow warning banner |
| `ApiTimeoutSeconds` | `SmartX.Client/wwwroot/appsettings.json` | `30` | How long the client waits for the API before giving up (1 to 300) |
| `Uploads:EncryptionKey` | `SmartX.Api/appsettings.json` (optional) | generated | Base64 string of exactly 32 bytes used to encrypt attachments. If not set, a key is generated once into `SmartX.Api/App_Data/vault.key` |
| `Cors:AllowedOrigins` | `SmartX.Api/appsettings.json` | `http://localhost:5274`, `https://localhost:7293` | Origins the API accepts calls from |

If you change a port, change **both** settings so they agree. The top bar shows a live **API online / offline** indicator that checks `/api/ping` every 10 seconds.

### 3.5 What happens on startup

- The API seeds itself with **9 mock devices** (4 moisture sensors, 3 smart meters, 2 actuators) and historical readings built from jagged arrays.
- A **telemetry simulator** keeps the seeded devices reporting, so the UI always has live data without any hardware.
- The **mesh graph** is built from a seeded layout of 14 weighted links around a central gateway (`SMARTX-GATEWAY`).
- All state is **in memory**. Restarting the API resets everything to the seeded state.
- Uploaded attachments are the exception: the encrypted files and the encryption key live in `SmartX.Api/App_Data` (a Docker volume named `smartx-data`, ignored by Git). Without a volume or the key file, old files can no longer be opened after a restart.

---

## 4. How to Use the Application

The top navigation bar links to every page. The home page shows the three modules as cards.

### 4.1 Sensor Data Ingestion and Telemetry (Part 1)

1. Open **Sensor Ingestion**.
2. Register a sensor: enter a MAC address (`AA:BB:CC:DD:EE:01`), choose a deployment location (for example `Facility A > Zone 1 > Sub-Zone B`) and a category (Environmental, Power, Actuator). Invalid MACs and locations are rejected with a clear message.
3. Submit telemetry by hand for a registered device. Environmental readings are a fraction between 0.0 and 1.0, power readings are watts (integer).
4. Attach a configuration file, log or photo to a sensor profile (`.txt`, `.log`, `.json`, `.csv`, `.cfg`, `.png`, `.jpg`, `.pdf` and a few others, up to 20 MB). The file is streamed to the API and **encrypted (AES-256-GCM)** before it is stored. Click a file name in the list to download it; the API decrypts it on the way out.
5. Use the **Dashboard** for the fleet health score, alert triage (acknowledge/resolve), streaks and missions.
6. The Dashboard also has a **Daily Challenge** (for example "Acknowledge 2 alerts today") that changes every day and pays 30 XP once when completed, and a **Come back for** prompt showing the next streak reward (3, 7, 14 and 30 days).

### 4.2 Real-Time Command Stream and History (Part 2)

Refreshes every 2 seconds. It shows queue statistics, active faults, a devices table with search and filters, a per-device timeline, manual overrides with **Undo/Redo**, and **Automated Insights and Suggested Actions**. See [section 6](#6-how-to-test-each-feature) for step-by-step tests.

### 4.3 Network Topology and Mesh Routing (Final PoE)

Open **Network Topology**. The page refreshes every 3 seconds and remembers your selection when you navigate away and back.

| Area | What it does |
|------|--------------|
| **Summary row** | Nodes in the mesh, nodes that reach the gateway, nodes down, nodes isolated, and the power saved by the MST backbone |
| **Mesh map** | SVG drawing of the live mesh. Circles are nodes (the number inside is the hop count to the gateway), lines are links (the label is the power cost). Green is online, grey is silent, red is failed, orange is isolated. MST links are highlighted when **Highlight MST links** is ticked |
| **Track a node** | Click a node, or type a MAC address or device name into the search box (the box offers every MAC as you type) |
| **Route tracker** | For the selected node: its current route to the gateway (fewest hops, found with BFS), its normal route if it has been rerouted, and its lowest-power route along the MST with a cost comparison. **Simulate failure** and **Restore node** buttons let you break and heal the mesh |
| **Failures, detours and subnet connectivity** | Lists down and isolated nodes and a table of *where traffic hops instead*. The **Subnet / zone** box (for example `Facility A`) lists every device in that zone that can still reach the gateway |
| **Telemetry history** | For the selected node: a time window (last hour up to 7 days) or a custom From/To range, answered by the AVL tree. Shows readings in the window, how many are stored, how many tree nodes were examined, tree height and rotations, then a newest-first table |
| **Lowest-power backbone** | Prim's MST statistics: backbone cost, cost with all links powered, saving, idle links, nodes spanned, and a per-node comparison of MST route against fewest-hop route |

---

## 5. API Reference

Base URL: `http://localhost:5216`

### Network topology (final PoE)

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/mesh?subnet=Facility A` | Whole mesh in one call: nodes, links, down/isolated nodes, reroutes, and (optionally) connected devices in a subnet |
| GET | `/api/mesh/route/{mac}` | Hop-by-hop route from one node to the gateway, with its normal route and a plain-English summary |
| GET | `/api/mesh/mst` | Prim's minimum spanning tree over the working nodes, with cost comparisons |
| POST | `/api/mesh/nodes/{mac}/fail` | Simulate a hardware failure (400 for the gateway or an unknown MAC) |
| POST | `/api/mesh/nodes/{mac}/restore` | Bring a failed node back |
| GET | `/api/history/{mac}?from=&to=&max=` | Time-window query answered by the AVL tree (`from`/`to` are ISO-8601 UTC; defaults to the last 24 hours) |

### Parts 1 and 2

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET / POST | `/api/sensors` | List / register sensors |
| GET | `/api/sensors/tree` | Valid deployment hierarchy |
| GET | `/api/sensors/{mac}` | One sensor profile |
| POST | `/api/sensors/{mac}/upload` | Multipart file attach (encrypted at rest) |
| GET | `/api/sensors/{mac}/files` | List attached file names |
| GET | `/api/sensors/{mac}/files/{name}` | Decrypt and download one attachment |
| POST | `/api/telemetry` | Submit a reading (`{"deviceMac":"...","rawValue":"0.42"}`) |
| GET | `/api/telemetry/{mac}` | 50 most recent readings |
| GET | `/api/alerts` | Visible alerts; `POST /api/alerts/{id}/acknowledge` and `/resolve` |
| POST / GET | `/api/engagement/visit` / `/summary` | Operator profile, streak, XP, missions |
| GET | `/api/stream/snapshot` | Everything the Command Stream page shows |
| POST | `/api/stream/search-event` | Teach the prediction engine a search |
| POST | `/api/commands`, `/undo`, `/redo` | Manual overrides and undo/redo; `GET /api/commands/history` |
| POST | `/api/simulate/burst`, `/api/simulate/critical` | Fire simulated packets at the intake |
| GET | `/api/ping` | Health check |

---

## 6. How to Test Each Feature

Start the app first (see [section 3](#3-how-to-compile-and-run)) and open http://localhost:5274.

### 6.0 Part 1 fixes: encryption, daily challenge, connection

**Test encrypted uploads**
1. Open **Sensor Ingestion**, choose a device and a small `.txt` file, then click **Upload**.
2. Expected: "Uploaded" message, and the file appears in the list with a lock icon.
3. Click the file name. Expected: the original text downloads and opens normally.
4. Open `SmartX.Api/App_Data/uploads/<mac>/` (in Docker: `docker compose exec api ls /app/App_Data/uploads`). Expected: the stored file ends in `.enc` and is unreadable ciphertext.

**Test the Daily Challenge**
1. Open **Dashboard**. Expected: a **Daily challenge** card with today's goal and a progress bar.
2. Do the task (for example acknowledge two alerts). Expected: the bar fills, the card turns green, and 30 XP is added once.

**Test the connection indicator**
1. Stop the API (`docker compose stop api`). Expected within 10 seconds: the top bar shows **API offline**, and forms say "Could not reach the Smart-X API" instead of crashing.
2. Start it again (`docker compose start api`). Expected: the pill returns to **API online**.

### 6.1 Final PoE: Network Topology

**Test the mesh graph and map**
1. Open **Network Topology**.
2. Expected: 10 nodes (gateway plus 9 devices), all online, summary shows `0` down and `0` isolated.

**Test route tracking by MAC**
1. Type `A1:B2:C3:01:00:03` into **Track a node** (or click Smart Meter 3).
2. Expected: *Smart Meter 3 reaches the gateway in 3 hop(s): Smart Meter 3 -> Moisture Sensor B -> Moisture Sensor A -> Smart-X Gateway.*

**Test BFS self-healing**
1. Select **Moisture Sensor A** and click **Simulate failure**.
2. Expected within 3 seconds: the node turns red, "Down: Moisture Sensor A" appears, and **Where traffic hops instead** lists Moisture Sensor B, Smart Meter 2 and Smart Meter 3 with new routes (Smart Meter 3 now goes via Moisture Sensor B and Moisture Sensor C).
3. Click **Restore node**. The mesh returns to normal.

**Test subnet connectivity**
1. Type `Facility A` into the **Subnet / zone** box.
2. Expected: a list of Facility A devices that can still reach the gateway. Fail a node and watch the list change.

**Test the AVL history query**
1. Select any sensor. In **Telemetry history** pick **Last hour**, or tick **Use From / To** and choose a narrow range.
2. Expected: *Tree nodes examined* is far smaller than *Stored in tree* for narrow windows, and *Tree height* stays small.

**Test the MST**
1. Scroll to **Lowest-power backbone**.
2. Expected (seeded layout): backbone cost **25.1** against **54.6** with every link powered, a **54%** saving, **5** idle links, **10 / 10** nodes spanned.
3. Fail Moisture Sensor A. Expected: **9 / 10** nodes spanned, and "Not reached by the tree" names the failed node.

**API checks** (optional, any terminal):

```bash
curl http://localhost:5216/api/mesh/mst
curl http://localhost:5216/api/mesh/route/A1:B2:C3:01:00:03
curl -X POST http://localhost:5216/api/mesh/nodes/A1:B2:C3:00:00:01/fail
curl "http://localhost:5216/api/history/A1:B2:C3:00:00:01?max=50"
curl -X POST http://localhost:5216/api/mesh/nodes/A1:B2:C3:00:00:01/restore
```

### 6.2 Part 2: queues, dictionaries, sets, stack

Open **Command Stream**.

| Feature | Steps | Expected result |
|---------|-------|-----------------|
| FIFO queue | **Simulate 60 packets** | *Packets processed* rises; the Standard lane shows packets in arrival order |
| Priority queue | **Simulate 60 packets + 1 critical** | The critical packet was sent last but is processed first, via the Priority lane |
| Dictionary | Type `pump` or a MAC into **Search** | Table filters instantly; each packet is matched to its device by MAC |
| Sorted dictionary | Click **Timeline** on Moisture Sensor A | Readings appear oldest to newest |
| HashSet | Click **Simulate a critical alert** several times | The fault appears once; the *Duplicate alerts blocked* counter rises |
| Stack (undo) | Turn ON Water Pump A, then Smart Relay B, then **Undo last command** twice | Relay B reverts first, then Pump A (last in, first out) |
| Suggested actions | Search `pump a` and press Enter | A *Turn ON Water Pump A* suggestion appears with its reason |

To send a critical reading directly to the API like a device would:

```bash
curl -X POST http://localhost:5216/api/telemetry -H "Content-Type: application/json" \
  -d '{"deviceMac":"A1:B2:C3:00:00:01","rawValue":"0.05"}'
```

Moisture is a fraction (0.0 to 1.0), so `0.05` is critically dry. For power use a value such as `4800` on a smart meter.

### 6.3 Navigation state

Set a search and select a device on Command Stream, select a node on Network Topology, move between pages and return. Searches, filters and selections are kept (they live in scoped state services that last as long as the browser tab).

---

## 7. Data Structure Explanations (Network Topology)

### 7.1 Overview: why these structures

The Network Topology page has to answer four questions quickly and repeatedly, every 3 seconds, while the mesh changes underneath it:

| Question | Structure | Where |
|----------|-----------|-------|
| *Give me this sensor's readings between 14:00 and 16:00* | **AVL tree** (self-balancing binary search tree) | `SmartX.Shared/Collections/AvlTree.cs`, `SmartX.Api/Data/TelemetryHistoryTree.cs` |
| *Which nodes are connected to which, and at what power cost?* | **Weighted undirected graph** (adjacency list) | `SmartX.Shared/Collections/MeshGraph.cs` |
| *Which nodes can still reach the gateway, how many hops, and where do they hop if a node is down?* | **Breadth-first search (BFS)** | `MeshGraph.Bfs` |
| *What is the cheapest set of links that keeps every node connected?* | **Binary min-heap** + **Prim's minimum spanning tree** | `MinHeap.cs`, `MeshGraph.Mst.cs` |

`MeshNetwork` (`SmartX.Api/Data/MeshNetwork.cs`) combines all three graph pieces behind one lock and one `Evaluate()` call so that every response is a consistent snapshot of the mesh at one moment.

```text
            POST /api/telemetry                         GET /api/mesh, /api/mesh/mst
                    │                                              │
          TelemetryProcessor                                MeshNetwork.Evaluate()
                    │                                              │
       LiveTelemetryIndex.Add ──► TelemetryHistoryTree      Up/Down per node (last-seen < 45 s?)
                                   (one AVL tree per MAC)          │
                                          ▲                  ┌─────┴───────────┐
          GET /api/history/{mac} ─────────┘                  │ BFS (all nodes) │ → normal routes
          RangeQuery(low, high)                              │ BFS (up nodes)  │ → live routes, reroutes
                                                             │ Prim (up nodes) │ → MST backbone
                                                             └─────────────────┘
```

---

### 7.2 AVL tree: historical telemetry queries

**File:** `SmartX.Shared/Collections/AvlTree.cs`

#### Role in the Network Topology feature

Every processed reading is stored in a per-sensor AVL tree keyed by timestamp (`LiveTelemetryIndex` calls `TelemetryHistoryTree.Add` for each reading, including the seeded history). The **Telemetry history** panel then asks for "readings between *from* and *to*", which is a range query on the key.

An unorganised list would have to examine every stored reading for every query. A plain binary search tree would fix that for random data, but telemetry arrives in **increasing timestamp order**, which turns an unbalanced BST into a linked list (height *n*). The AVL tree rebalances with rotations so its height always stays logarithmic.

#### Key design

Readings that share the same timestamp must not overwrite each other, so the key is a composite of the timestamp ticks and an arrival sequence number:

```csharp
public readonly record struct TelemetryKey(long Ticks, long Sequence) : IComparable<TelemetryKey>
{
    public int CompareTo(TelemetryKey other)
    {
        int byTime = Ticks.CompareTo(other.Ticks);
        return byTime != 0 ? byTime : Sequence.CompareTo(other.Sequence);
    }
}
```

The tree itself is generic, so it can hold any key that implements `IComparable<TKey>`:

```csharp
public sealed class AvlTree<TKey, TValue> where TKey : IComparable<TKey>
```

Each node stores its own `Height`, which is how balance is measured cheaply.

#### Insertion and self-balancing

Insertion is recursive. After inserting into a subtree, each node on the way back up recalculates its height and rebalances:

```csharp
private Node Insert(Node? node, TKey key, TValue value)
{
    if (node == null) { Count++; return new Node(key, value); }

    int cmp = key.CompareTo(node.Key);
    if (cmp < 0)      node.Left  = Insert(node.Left, key, value);
    else if (cmp > 0) node.Right = Insert(node.Right, key, value);
    else { node.Value = value; return node; }

    return Rebalance(node);
}

private Node Rebalance(Node node)
{
    UpdateHeight(node);
    int balance = HeightOf(node.Left) - HeightOf(node.Right);

    if (balance > 1)   // left-heavy
    {
        if (HeightOf(node.Left!.Left) < HeightOf(node.Left.Right))
            node.Left = RotateLeft(node.Left);   // left-right case
        return RotateRight(node);
    }
    if (balance < -1)  // right-heavy
    {
        if (HeightOf(node.Right!.Right) < HeightOf(node.Right.Left))
            node.Right = RotateRight(node.Right); // right-left case
        return RotateLeft(node);
    }
    return node;
}
```

The AVL invariant is that at every node the left and right subtree heights differ by at most 1. That guarantees height ≤ about 1.44 × log₂(n), so insert, delete and lookup are all **O(log n)**. All four classic cases are handled: left-left and right-right with one rotation, left-right and right-left with two.

#### The range query (the part that makes history fast)

```csharp
private static void Collect(Node? node, TKey low, TKey high, List<TValue> result, ref int visited)
{
    if (node == null) return;

    visited++;
    int vsLow  = low.CompareTo(node.Key);   // < 0 => node is above the lower bound
    int vsHigh = high.CompareTo(node.Key);  // > 0 => node is below the upper bound

    if (vsLow < 0)                          // only go left if keys >= low could be there
        Collect(node.Left, low, high, result, ref visited);

    if (vsLow <= 0 && vsHigh >= 0)
        result.Add(node.Value);

    if (vsHigh > 0)                         // only go right if keys <= high could be there
        Collect(node.Right, low, high, result, ref visited);
}
```

This is an in-order traversal that **prunes**: a whole left subtree is skipped when the current node is already at or below the lower bound, and a whole right subtree is skipped when the node is already at or above the upper bound. Results come out in ascending timestamp order, so no sort is needed. The cost is **O(log n + k)**, where *k* is the number of readings returned, instead of O(n).

`TelemetryHistoryTree.Query` builds the bounds so that every reading at the boundary timestamps is included, whatever its sequence number:

```csharp
var low  = new TelemetryKey(fromUtc.Ticks, long.MinValue);
var high = new TelemetryKey(toUtc.Ticks,   long.MaxValue);
var matches = history.Tree.RangeQuery(low, high, out int visited);
```

#### Memory protection

Each device keeps at most 5,000 readings. When it goes over, the oldest are evicted by taking the minimum key out of the tree, also **O(log n)** per eviction:

```csharp
history.Tree.Insert(new TelemetryKey(entry.TimestampUtc.Ticks, ++history.Sequence), entry);

while (history.Tree.Count > MaxPerDevice && history.Tree.TryGetMin(out var oldest))
    history.Tree.Remove(oldest);
```

Each device has its own tree and its own lock, so a query for one sensor never blocks ingestion for another.

#### Proof of efficiency in the UI

The API returns the cost of every query and the page displays it:

| Statistic | Meaning |
|-----------|---------|
| Readings in window | *k*, the matches |
| Stored in tree | *n* |
| Tree nodes examined | How many nodes `Collect` actually touched |
| Tree height | Stays near log₂(n) |
| Rotations | Total rebalancing work done so far |

In a sample run, a narrow window matching 7 of 32 stored readings examined only **12 nodes** on a tree of height 6. A list scan would examine all 32. At the 5,000-reading cap the gap is much larger: a balanced tree has height of about 13 to 17, so a query touches roughly that many nodes plus the matches, against 5,000 for a scan.

#### Complexity summary

| Operation | AVL tree | Unsorted list |
|-----------|----------|---------------|
| Insert a reading | O(log n) | O(1) |
| Evict oldest | O(log n) | O(n) |
| Range query | **O(log n + k)** | **O(n)** |
| Results in time order | Free (in-order) | Needs an O(n log n) sort |

---

### 7.3 Graph: modelling the mesh

**File:** `SmartX.Shared/Collections/MeshGraph.cs`

#### Role in the Network Topology feature

The graph **is** the mesh. Each ESP32 sensor (and the gateway) is a **vertex**, and each wireless link is an **undirected weighted edge**. The weight is the relative **transmit power cost** of that hop: lower is cheaper. Routing, failure analysis and the MST are all questions about this one structure.

#### Representation: adjacency list

```csharp
public readonly record struct Edge(string To, double Weight);

private readonly Dictionary<string, List<Edge>> _adjacency = new(StringComparer.OrdinalIgnoreCase);
```

An adjacency list stores, for every vertex, only the links it actually has. A mesh is **sparse** (each node hears a handful of neighbours, not thousands), so this takes **O(V + E)** memory, whereas an adjacency matrix would take O(V²). With the brief's "thousands of nodes", a matrix of millions of cells would be almost entirely empty. The `Dictionary` gives O(1) lookup of a node by MAC, and the case-insensitive comparer means `a1:b2:...` and `A1:B2:...` are the same node.

An undirected link is stored at both ends so that a node can discover its neighbours without searching:

```csharp
public bool AddEdge(string a, string b, double weight)
{
    if (string.Equals(a, b, StringComparison.OrdinalIgnoreCase) || HasEdge(a, b))
        return false;                                   // no self-loops, no duplicate links

    if (!_adjacency.TryGetValue(a, out var fromA) || !_adjacency.TryGetValue(b, out var fromB))
        return false;                                   // both ends must already exist

    fromA.Add(new Edge(b, weight));
    fromB.Add(new Edge(a, weight));
    EdgeCount++;
    return true;
}
```

#### Seeded layout

The gateway and nine devices are joined by 14 links (for example gateway to Moisture Sensor A at cost 4.0, Moisture Sensor A to Moisture Sensor B at 2.0). Several nodes have two or more routes to the gateway, which is what makes self-healing possible:

```csharp
(GatewayMac, "A1:B2:C3:00:00:01", 4.0),
(GatewayMac, "A1:B2:C3:00:00:03", 7.0),
("A1:B2:C3:00:00:01", "A1:B2:C3:00:00:02", 2.0),
("A1:B2:C3:00:00:02", "A1:B2:C3:00:00:03", 5.0),
...
```

#### Growing with the network

When a new sensor is registered it joins the graph automatically. `LinkToNeighbours` links it to the (up to) two registered nodes that share the most segments of its deployment path (`Facility > Zone > Sub-Zone`), and falls back to the gateway so a node is never left disconnected. Nearer in the hierarchy means a cheaper link:

```csharp
foreach (var (mac, shared) in nearest)
    _graph.AddEdge(device.MacAddress, mac, 1.5 + (3 - Math.Min(3, shared)) * 2.0);

if (nearest.Count == 0)
    _graph.AddEdge(device.MacAddress, GatewayMac, 8.0);
```

#### Up, down and isolated

A node is **up** when it is the gateway, or it has not been failed *and* its device has reported within the 45-second disconnect threshold:

```csharp
up[mac] = IsGateway(mac) ||
          (!_failed.Contains(mac) &&
           _store.Devices.TryGetValue(mac, out var device) &&
           now - device.LastSeen <= AlertClassifier.DisconnectThreshold);
```

So the topology reacts both to *simulated* failures (the **Simulate failure** button) and to *real* silence from a sensor that stops sending. Each node is classified as:

| Status | Meaning |
|--------|---------|
| Online | Up, and a path of working nodes leads to the gateway |
| Offline | Silent for more than 45 seconds |
| Failed | Switched off with **Simulate failure** |
| Isolated | Up, but every path to the gateway runs through a down node |

---

### 7.4 Graph traversal: BFS routing and self-healing

**File:** `SmartX.Shared/Collections/MeshGraph.cs` (`Bfs` and `BfsResult`)

#### Role in the Network Topology feature

BFS answers three of the brief's requirements with one algorithm:

1. **Which devices are connected to the gateway or a subnet?** Everything BFS reaches from the gateway over working nodes.
2. **Which are down?** Nodes that are not up, plus up nodes BFS could not reach (isolated).
3. **Where do they hop if a node is down?** The BFS route over working nodes only.

BFS explores the graph in rings: first every node one hop from the start, then two hops, and so on. Because all nodes at distance *d* are visited before any at distance *d + 1*, the first time BFS reaches a node it has found a path with the **fewest hops**.

```csharp
public BfsResult Bfs(string start, Func<string, bool>? isUsable = null)
{
    var result = new BfsResult(start);

    if (!_adjacency.ContainsKey(start) || (isUsable != null && !isUsable(start)))
        return result;

    var queue = new Queue<string>();
    result.Visit(start, null, 0);
    queue.Enqueue(start);

    while (queue.Count > 0)
    {
        var current = queue.Dequeue();

        foreach (var edge in _adjacency[current])
        {
            if (result.Contains(edge.To)) continue;                    // already reached
            if (isUsable != null && !isUsable(edge.To)) continue;      // skip down nodes

            result.Visit(edge.To, current, result.Depth[current] + 1);
            queue.Enqueue(edge.To);
        }
    }
    return result;
}
```

The `isUsable` predicate is what turns plain BFS into a **fault-aware router**: down nodes are simply never entered. `BfsResult` records each node's parent, so a full route can be rebuilt by walking parents back to the start:

```csharp
public List<string> PathTo(string id)
{
    var path = new List<string>();
    if (!Contains(id)) return path;          // empty path = unreachable

    string? current = id;
    while (current != null) { path.Add(current); current = Parent[current]; }

    path.Reverse();
    return path;
}
```

#### Two BFS passes give the self-healing view

`MeshNetwork.Evaluate()` runs BFS twice from the gateway:

```csharp
var normal = _graph.Bfs(GatewayMac);                 // every node usable: the intended routes
var live   = _graph.Bfs(GatewayMac, mac => up[mac]); // only working nodes: what is possible now
```

Comparing the two shows exactly what a failure changed. For any working node whose *normal* route passes through a down node, the snapshot records what blocked it and the *new* route:

```csharp
var blockedBy = normalRoute
    .Where(hop => hop != mac && !state.Up[hop])
    .ToList();

if (blockedBy.Count == 0) continue;
...
snapshot.Reroutes.Add(new RerouteDto { Mac = mac, BlockedBy = blockedBy,
    NormalRoute = ..., NewRoute = newRoute, Reachable = newRoute.Count > 0 });
```

**Worked example from the seeded mesh.** Smart Meter 3 normally reaches the gateway as `Smart Meter 3 -> Moisture Sensor B -> Moisture Sensor A -> Gateway`. Fail Moisture Sensor A and BFS over the working nodes returns `Smart Meter 3 -> Moisture Sensor B -> Moisture Sensor C -> Gateway`. This is the "holding hands" self-healing behaviour from the brief, computed rather than hard-coded. If no working path exists, the route is empty and the node is shown as **Isolated**.

#### Subnet connectivity

The *subnet* query reuses the live BFS result and filters by deployment path:

```csharp
snapshot.SubnetConnected = _graph.Vertices
    .Where(mac => !IsGateway(mac) && state.Live.Contains(mac) &&
                  LocationOf(mac).StartsWith(snapshot.Subnet, StringComparison.OrdinalIgnoreCase))
    .ToList();
```

#### Why BFS and not DFS

DFS would also find which nodes are connected, but it follows one branch to the end first, so the path it returns to a node can be long and winding. BFS is the right fit for mesh routing because it returns the **fewest-hop** route, and fewer hops mean lower latency and fewer chances for a packet to be dropped.

**Complexity:** O(V + E) time (every node dequeued once, every link examined from each end), O(V) extra space for the queue and result.

---

### 7.5 Heap: the priority structure behind Prim's algorithm

**File:** `SmartX.Shared/Collections/MinHeap.cs`

#### Role in the Network Topology feature

Prim's algorithm repeatedly needs "the cheapest link leaving the tree built so far". A binary **min-heap** answers that in O(1) to read and O(log n) to update, instead of re-sorting or rescanning every candidate link on every step.

The heap is a complete binary tree stored in a flat `List<T>`. For the item at index *i*, its parent is at `(i - 1) / 2` and its children at `2i + 1` and `2i + 2`, so no node objects or pointers are needed. The invariant is that every parent is less than or equal to both children, which keeps the minimum at index 0.

```csharp
public void Push(T item)
{
    _items.Add(item);
    SiftUp(_items.Count - 1);                 // bubble the new item up
}

public bool TryPop([MaybeNullWhen(false)] out T item)
{
    if (_items.Count == 0) { item = default; return false; }

    item = _items[0];                          // the minimum

    int last = _items.Count - 1;
    _items[0] = _items[last];                  // move the last item to the root
    _items.RemoveAt(last);

    if (_items.Count > 0) SiftDown(0);         // let it sink to its place
    return true;
}

private void SiftUp(int index)
{
    while (index > 0)
    {
        int parent = (index - 1) / 2;
        if (_comparer.Compare(_items[index], _items[parent]) >= 0) break;
        Swap(index, parent);
        index = parent;
    }
}
```

`MinHeap<T>` is generic and takes an `IComparer<T>`, so the same class can order anything. The mesh supplies a comparer that orders candidate links by **weight first**, then by node name, so ties are broken the same way on every run and the MST is deterministic:

```csharp
private static readonly Comparer<(double Weight, string From, string To)> CandidateOrder =
    Comparer<(double Weight, string From, string To)>.Create((x, y) =>
    {
        int byWeight = x.Weight.CompareTo(y.Weight);
        if (byWeight != 0) return byWeight;

        int byTo = string.CompareOrdinal(x.To, y.To);
        return byTo != 0 ? byTo : string.CompareOrdinal(x.From, y.From);
    });
```

| Operation | Heap | Sorted list | Unsorted list |
|-----------|------|-------------|---------------|
| Push | O(log n) | O(n) | O(1) |
| Pop minimum | O(log n) | O(1) | O(n) |
| Peek minimum | O(1) | O(1) | O(n) |

---

### 7.6 Minimum Spanning Tree: Prim's algorithm and power-efficient routing

**File:** `SmartX.Shared/Collections/MeshGraph.Mst.cs`

#### Role in the Network Topology feature

A **spanning tree** connects every node using exactly V − 1 links with no cycles. The **minimum** spanning tree is the one whose links have the lowest total weight. Because the edge weights are power costs, the MST is the cheapest *backbone* that keeps every working node connected to the gateway. Links outside the tree can idle (or sleep) to save power, and stay available as spares if the backbone is cut. This mirrors how mesh protocols such as Zigbee, Thread and ESP-NOW look for efficient routes.

#### How Prim's algorithm works here

1. Start with the gateway as the only node in the tree.
2. Push every link leaving the gateway onto the min-heap.
3. Pop the cheapest link. If it leads to a node that is already in the tree, discard it (a cheaper link got there first). Otherwise add the node and that link to the tree, and push the new node's links onto the heap.
4. Repeat until the heap is empty.

```csharp
public MstResult PrimMst(string start, Func<string, bool>? isUsable = null)
{
    var result = new MstResult(start);

    if (!_adjacency.ContainsKey(start) || (isUsable != null && !isUsable(start)))
        return result;

    var heap = new MinHeap<(double Weight, string From, string To)>(CandidateOrder);

    void PushLinksFrom(string node)
    {
        foreach (var edge in _adjacency[node])
        {
            if (result.Contains(edge.To)) continue;
            if (isUsable != null && !isUsable(edge.To)) continue;   // never route through down nodes
            heap.Push((edge.Weight, node, edge.To));
        }
    }

    result.AddRoot(start);
    PushLinksFrom(start);

    while (heap.TryPop(out var candidate))
    {
        // A cheaper link may already have pulled this node into the tree.
        if (result.Contains(candidate.To)) continue;

        result.AddEdge(candidate.From, candidate.To, candidate.Weight);
        PushLinksFrom(candidate.To);
    }
    return result;
}
```

`MstResult` records each node's `Parent` in the tree, so a node's route back to the gateway along the backbone is a simple walk up the parents:

```csharp
public List<string> RouteToRoot(string id)
{
    var route = new List<string>();
    if (!Contains(id)) return route;

    string? current = id;
    while (current != null) { route.Add(current); current = Parent[current]; }
    return route;
}
```

The same `isUsable` filter as BFS is passed in (`mac => state.Up[mac]`), so the MST is always computed over **working nodes only** and is recomputed whenever the mesh changes. That is why it drops from 10 / 10 to 9 / 10 nodes spanned when a node fails.

**Complexity:** O(E log V) with a binary heap (each link is pushed at most once from each end and each push or pop costs O(log E)). A version without a heap, scanning all candidates each step, would be O(V · E).

#### Results on the seeded mesh

Hand-checked against the 14 seeded links:

| Measure | Value |
|---------|-------|
| Links in the mesh | 14 (total cost **54.6**) |
| Links in the MST | 9 (V − 1 for 10 nodes), total cost **25.1** |
| Power saved by idling the other 5 links | 29.5, which is **54%** |
| MST with Moisture Sensor A failed | 9 of 10 nodes spanned, cost 27.1, failed node listed under "Not reached" |

The MST links in order of selection are: gateway-Sensor A (4.0), Sensor A-Meter 2 (1.5), Sensor A-Sensor B (2.0), Sensor B-Meter 3 (1.8), Sensor B-Sensor C (5.0), Sensor C-Pump A (1.2), gateway-Sensor D (6.0), Sensor D-Relay B (1.4), Relay B-Meter 1 (2.2).

#### An honest trade-off, shown in the UI

The MST minimises the **total** power of the backbone. It does not minimise each node's *individual* path cost. The page therefore compares both routes for every node:

| Node | MST route | Fewest-hop route (BFS) |
|------|-----------|------------------------|
| Moisture Sensor A | 1 hop, cost 4.0 | 1 hop, cost 4.0 (same) |
| Moisture Sensor B | 2 hops, cost 6.0 | 2 hops, cost 6.0 (same) |
| Moisture Sensor C | 3 hops, cost 11.0 | 1 hop, cost 7.0 |

Sensor C's direct link to the gateway (7.0) is not in the tree, because routing C through B and A reuses links the tree already powers, so keeping the direct link on would cost extra for the network as a whole. This is the real trade-off in mesh design: the MST backbone saves energy overall at the price of longer routes for some nodes. Showing BFS and MST side by side lets an operator choose between latency (fewest hops) and power (backbone).

---

### 7.7 How the structures work together (one refresh)

Every 3 seconds the page calls `GET /api/mesh` and `GET /api/mesh/mst`, then (if a node is selected) `GET /api/mesh/route/{mac}` and `GET /api/history/{mac}`:

1. `MeshNetwork` takes its lock, adds any newly registered sensors to the **graph**, and works out which nodes are up.
2. **BFS** runs over all nodes (normal routes) and over working nodes (live routes). Differences become the reroute table.
3. **Prim's algorithm**, driven by the **min-heap**, builds the MST over working nodes. Its links are flagged on the map.
4. For the selected node, the **AVL tree** answers the history window.

The graph work is O(V + E) for BFS and O(E log V) for Prim's algorithm, and the history query is O(log n + k). None of these scans an unorganised list, which is what lets the topology page stay responsive as the mesh and the history grow.

---

## 8. Data Structures from Parts 1 and 2

| Requirement | Structure | Where | Purpose |
|-------------|-----------|-------|---------|
| Generic wrapper | `TelemetryPacket<T>` (`float`, `int`, `bool`) | `SmartX.Shared/Models` | One wrapper for every reading type without boxing |
| Operator overloading | `PowerReading` (`+`, `-`, `>`, `<`, `>=`, `<=`) | `SmartX.Shared/Models` | Aggregate and compare power readings directly |
| Jagged arrays to `List<T>` | `float[][]`, `int[][]` batches | `HistoricalBatchProcessor` | Seed sequential history, then flatten into typed lists |
| Recursion | `LocationValidator.ValidateRecursive` | `SmartX.Shared/Validation` | Validate `Facility > Zone > Sub-Zone` paths |
| Custom collection | `TelemetryBuffer<T>` (ring buffer) | `SmartX.Shared/Collections` | Fixed-capacity store, newest-first reads without sorting |
| FIFO queue | `MessageQueue<T>` over `Queue<T>` | `SmartX.Shared/Collections` | Standard telemetry processing in arrival order |
| Priority queue | `AlertPriorityQueue<T>` over `PriorityQueue<,>` | `SmartX.Shared/Collections` | Critical alerts bypass the standard lane |
| Stack | `Stack<T>` in `CommandCenter` | `SmartX.Api/Data` | Undo of manual override commands |
| Dictionary | `Dictionary<string, ...>` keyed by MAC | `SmartX.Api/Data` | O(1) device lookup per incoming packet |
| Sorted collection | Per-device ordered logs | `SmartX.Api/Data` | Timeline rendered in timestamp order |
| HashSet | Per-MAC fault sets and a disconnected set | `FaultRegistry` | Unique active faults, duplicate alerts blocked |
| Encryption | `FileVault` (AES-256-GCM, 64 KB chunks) | `SmartX.Api/Data` | Attachments encrypted at rest without loading the whole file into memory |
| Error handling | try/catch in every data, collection and validation class | All projects | Failures are logged and return a safe result or a clear error instead of crashing |
| Prediction | `PatternEngine` | `SmartX.Api/Data` | Association rules, sequence tracking and trend forecasting for Suggested Actions |

**Error handling.** Each public method in the API data classes, shared collections and validators wraps a private `...Core` method in a try/catch. The catch logs the error, then either returns a safe value (`false`, an empty list, a `(null, message)` result) or rethrows a clear `InvalidOperationException`. A global exception handler in `SmartX.Api/Program.cs` turns anything still uncaught into a plain "unexpected error" 500 response, and the client wrapper `SmartXApiClient` turns network failures into friendly messages.

The final PoE reuses the Part 1 and 2 foundations: the mesh decides a node is **up** using the same 45-second `LastSeen` heartbeat the disconnect alerts use, node names come from the same device labels, and the history tree is fed from the same ingestion pipeline.

---

## 9. Troubleshooting

| Problem | Likely cause and fix |
|---------|----------------------|
| UI shows "Can't reach the Smart-X API" | The API is not running, or `ApiBaseUrl` in `SmartX.Client/wwwroot/appsettings.json` does not match its address. Start the API first and check http://localhost:5216/api/ping |
| Browser console shows a CORS error | The client's origin is missing from `Cors:AllowedOrigins` in `SmartX.Api/appsettings.json`. Add it and restart the API |
| "Address already in use" | Another program uses port 5216 or 5274. Stop it, or change the ports and update both settings above |
| `dotnet build` says the SDK is too old | Install the .NET 10 SDK (`dotnet --version` must show 10.x) |
| `docker compose up` fails on build | Make sure Docker Desktop is running, then retry with `docker compose build --no-cache` |
| Mesh or Command Stream looks empty after a restart | State is in memory. Wait a few seconds for the simulator and seeded data to load |
| Yellow banner says `ApiBaseUrl` is invalid or not set | Fix `ApiBaseUrl` in `SmartX.Client/wwwroot/appsettings.json` (it must start with `http://` or `https://`). The client is using the default `http://localhost:5216/` meanwhile |
| An attachment fails with "failed its integrity check" | The file was changed on disk, or the encryption key changed. Keep `App_Data/vault.key` (or the `Uploads:EncryptionKey` setting) the same between runs |
| Attachments disappear after `docker compose down -v` | The `-v` flag deletes the `smartx-data` volume that holds them. Use plain `docker compose down` |
| A node shows Offline instead of Failed | It stopped reporting for over 45 seconds (for example the API was paused). It recovers on its next reading |

---

## 10. Demo Videos

- Part 1: https://youtu.be/7QHhyl1f91w
- Part 2: https://youtu.be/gaW7_h13Xx8
- Final PoE: _add link_

---

## 11. References

Almasi, S., Bahaadinbeigy, K., Ahmadi, H., Sohrabei, S. and Rabiei, R., 2023. Usability evaluation of dashboards: a systematic literature review of tools. BioMed Research International, 2023, Article 9990933. Available: https://doi.org/10.1155/2023/9990933

Cormen, T.H., Leiserson, C.E., Rivest, R.L. and Stein, C., 2022. Introduction to algorithms. 4th ed. Cambridge, MA: MIT Press.

Lavalle, A., Teruel, M.A., Maté, A. and Trujillo, J., 2020. Fostering sustainability through visualisation techniques for real-time IoT data: a case study based on gas turbines for electricity production. Sensors, 20(16), Article 4556. Available: https://doi.org/10.3390/s20164556

Prim, R.C., 1957. Shortest connection networks and some generalizations. Bell System Technical Journal, 36(6), pp.1389-1401.

Stahmann, P., Rodda, A., Nebel, M., van der Staay, A., Janiesch, C. and Teuteberg, F., 2025. Advanced analytics in real-time operational dashboards in smart manufacturing: effects on users' cognitive load and task load. Enterprise Information Systems, [online], Article 2593245. Available: https://doi.org/10.1080/12460125.2025.2593245

Voutsas, F., Violos, J. and Leivadeas, A., 2024. Mitigating alert fatigue in cloud monitoring systems: a machine learning perspective. Computer Networks, 250, Article 110543. Available: https://doi.org/10.1016/j.comnet.2024.110543

Xiao, R., Wu, Z. and Hamari, J., 2022. Internet-of-Gamification: a review of literature on IoT-enabled gamification for user engagement. International Journal of Human-Computer Interaction, 38(12), pp.1113-1137. Available: https://doi.org/10.1080/10447318.2021.1990517

---

## 12. Updates Made From Lecturer Feedback

The full table (feedback, what changed and where) is in. Summary:

| Feedback | Update |
|----------|--------|
| Add the DOI to references | DOIs added to the research report's reference list |
| Incorrect connection string | `ApiSettings` validates `ApiBaseUrl`, applies a timeout, falls back with a visible warning; live API online/offline indicator |
| Upload layout and "full encryption" | AES-256-GCM encryption at rest, files stored outside `wwwroot`, decrypting download endpoint, tidier upload form |
| User must keep coming back | Daily Challenge, streak milestone rewards and a "Come back for" prompt |
| Needs Home, About and Dashboard | Present in the top bar; About and Dashboard now link to the live Network Topology module |
| try/catch in all classes | Added across the API, shared collections, validators, upload endpoints and client wrapper, plus a global exception handler |

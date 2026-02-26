# fun-notes

This is simple collaborative note taking application, and it has the CLI support

## Backend (Go + Gorilla WebSockets)

- Note Room Management: Each room is identified by a unique note-room-id.
- Postgres Schema (Long-term Storage):
- Table note_rooms: id (PK), name, active_user_count, total_notes.
    - Table notes: id (PK), title, body, is_important, is_completed, current_index (FLOAT), note_room_id (FK), sequence_id (BIGINT), updated_at (TIMESTAMP).
- Indexing: A B-tree index on note_room_id in the notes table to ensure $O(\log n)$ retrieval of a room's full state.
- Stateless Architecture: Servers are stateless; client state (which room they are in) is handled via the connection and Redis.
- The "Catch-up" Pipeline (Double-Join):
  - When a user joins, the backend fetches the "Cold" snapshot from Postgres.
  - Immediately after, it fetches the "Hot Gap" from a Redis Sorted Set (ZSET) using ZRANGEBYSCORE where the score > last_postgres_sequence_id.
- Redis Pub/Sub: Notes are published to a Redis channel sharded by note-room-id to broadcast updates to all distributed Go server instances.
- Write-Behind Persistence: WebSockets update the Redis Write-Through Cache immediately for real-time feel, while a worker pool debounces and batches writes to Postgres asynchronously to protect DB IOPS.

## Frontend (React)

- Room Entry: A landing page allowing users to join an existing note-room-id or create a new room.
- Live Dashboard: A real-time table of all active rooms showing live active_user_count and total_notes powered by a global WebSocket stream.
- Collaborative Workspace:
  - Notes are displayed in a list that updates in real-time as other users type.
  - Drag & Drop: Integration of react-beautiful-dnd or dnd-kit. Swapping a note updates its currentIndex and triggers a REORDER event via WebSockets.
- Optimistic UI: The UI updates locally immediately and reconciles with the server's sequence_id to ensure correctness without flickering.

## CLI (Go + Bubble Tea)

- TUI Representation: A terminal-based interface using the Bubble Tea framework to mimic the React dashboard.
- Interactivity: Support for keyboard-driven navigation and "Drag & Drop" (Move Up/Down) functionality using the Lip Gloss and Bubbles components.
- Real-time: Maintains a persistent WebSocket connection to the Go backend for instant terminal updates.



## Improvements & Technical Challenges

- Local Caching (Offline Support):
  - CLI: Uses a local SQLite database to store notes. It performs a three-way sync: Remote Snapshot -> Local Changes -> Merge.
  - UI: Uses IndexedDB or LocalStorage for browser caching.
- Conflict Resolution: Implementation of Last-Write-Wins (LWW) or Operational Transformation (OT) logic to ensure that if two people edit the same character simultaneously, the state remains consistent across all clients.
- Scaling (10M Active Users):
  - Redis Cluster: Sharding data across nodes to handle memory overhead.
  - Load Balancing: Using Nginx or HAProxy to distribute WebSocket connections across a horizontal fleet of Go servers.
- Resource Optimization:
  - TTL: Redis ZSETs and metadata keys use a sliding-window TTL (e.g., 6 hours), resetting on every user interaction to ensure inactive rooms don't consume RAM.
- Security: Implementation of JWT-based Authentication and RBAC (Role-Based Access Control) to ensure only invited users can join specific note-room-ids.


## Tech Stack


| Component | Technology |
|-----------|-----------|
| Backend Language | Go (Golang) |
| WebSocket Framework | Gorilla WebSocket |
| Primary Database | PostgreSQL |
| Cache/Message Broker | Redis (Cluster + Pub/Sub + ZSET) |
| Frontend Library | React.js |
| CLI Framework | Bubble Tea (Cousin of the Elm Architecture) |
| Local DB (CLI) | SQLite |
| Containerization | Docker + Kubernetes (for scaling) |



## Overall System Flow

1. **Entry**: User enters a note-room-id or creates a new one via the React UI.

2. **Hydration (The Catch-up)**:
    - The client establishes a WebSocket connection.
    - Backend fetches the "Cold" snapshot from Postgres (e.g., updates up to Sequence #1000).
    - Backend immediately checks the Redis ZSET for "Hot" updates (e.g., Sequence #1001 to #1005) that haven't been persisted to Postgres yet.
    - The client merges these to show the absolute latest state.

3. **Real-time Collaboration**:
    - Any user update (editing text, dragging to reorder) is sent via WebSocket.
    - The server atomically updates the Redis ZSET, resets the TTL, and publishes to the room's Redis channel.
    - The server triggers an async pipeline to eventually save the change to Postgres.

4. **Live Monitoring**: The Room List table uses a global WebSocket stream to reflect real-time user counts and note totals across all active rooms.


## Backend Architecture

- **API Gateway / Load Balancer**: Nginx or HAProxy to distribute WebSocket connections across multiple Go server instances.
- **Stateful Connection Layer**: Go (Gorilla WebSocket) servers that handle individual socket lifecycles.
- **Transient Storage (Hot Data)**: Redis Cluster
    - Sorted Sets (ZSET): Stores the last ~100 updates per room to fill the gap between DB lag and real-time.
    - Pub/Sub: Handles message broadcasting across sharded backend servers.
- **Persistent Storage (Cold Data)**: PostgreSQL
    - Primary source of truth for long-term storage and room metadata.
    - Indexed on note-room-id for high-speed initial fetches.
- **Async Worker Pool**: A queue (Go Channels or Redis Stream) that debounces frequent keystrokes and batches writes to Postgres to save IOPS.



## Handling Race Conditions, Scaling, and Real-Time Reordering

### 1. Handling the "Missing Note D" Race Condition

By using the Last Sequence Number logic, we ensure consistency. Every update in a room gets a strictly increasing ID.

- If the DB snapshot provides notes up to ID: 100, but Redis has IDs: 101, 102, the backend sends the ZSET data immediately after the DB result.
- This prevents the "Note D" problem where a user joins while a note is still stuck in the async save pipeline.

### 2. Scaling for 10M Users

**Redis TTL (Sliding Window)**
- We use a 6-hour TTL on room ZSETs
- Every interaction (keystroke/ping) runs `EXPIRE room:updates:{id} 21600`
- If a room is inactive, Redis clears memory automatically
- Postgres remains the permanent backup

**Debouncing Writes**
- To avoid crashing Postgres with 10M users' typing, the backend only pushes to the DB write-queue after a "user-stop-typing" event or every 30 seconds
- This significantly reduces database load while maintaining data durability

### 3. Note Reordering (Drag & Drop)

- Each note has a `currentIndex` field
- When a user drags a note, a `REORDER` event is published via WebSocket
- The backend updates the `currentIndex` in the Redis ZSET using an atomic Lua script
- Since all clients receive the same `currentIndex` updates via Pub/Sub, the React frontend updates its local state
- Notes "swap" live on everyone's screen with consistent ordering across all clients


## To implement the catch-up and real-time synchronization logic for 10 million users

To implement the catch-up and real-time synchronization logic for 10 million users using Redis Sorted Sets (ZSET), you need a system that treats the ZSET as a circular buffer of recent activity.

Here is how you would structure the commands and the logic to handle the massive scale and the "missing message" problem.

### 1. Data Structure: Redis ZSET per Room

Instead of a simple Redis List, use a Sorted Set (ZSET).

- **Key**: `room:updates:{note-room-id}`
- **Score**: The sequence_number (or a high-precision timestamp)
- **Value**: The JSON string of the note update

### 2. The "Update" Logic (The Producer)

When a user updates a note, the backend server performs a Redis Transaction (MULTI/EXEC) or a Lua script to ensure atomicity:

```bash
# 1. Add the new update to the ZSET
ZADD room:updates:123 1005 "{\"id\": 'note_D', \"body\": '...'}"

# 2. Reset the TTL (Sliding Window)
EXPIRE room:updates:123 21600  # 6 hours in seconds

# 3. Cap the buffer size (Keep only the last 100 updates to save memory)
ZREMRANGEBYRANK room:updates:123 0 -101

# 4. Notify active listeners
PUBLISH room:123 "{\"seq\": 1005, \"data\": {...}}"
```

### 3. The "Join/Reconnection" Logic (The Consumer)

This is where you solve your race condition. When a user connects, the server performs a two-step "Hydration" process.

**Step A: Fetch Snapshot**

The client fetches the initial list from Postgres (or the primary cache). Let's say the last message in the DB has sequence_number: 1000.

**Step B: Fetch the "Gap" from Redis**

While the WebSocket is initializing, the server queries Redis for any updates that happened after the DB snapshot but before the live stream started.

```bash
# Fetch all updates with a score higher than 1000 (the DB version)
ZRANGEBYSCORE room:updates:123 (1000 +inf
```

### 4. Handling 10M Users: Memory Optimization

At 10M users, memory is your most expensive resource.

- **Compact Values**: Instead of storing the full note body in every update in the ZSET, store only the Diff (changes) or a reference ID if the bodies are large.
- **Shard the ZSETs**: Use Redis Cluster. Sharding ensures that the `room:updates:123` key and the `room:123` Pub/Sub channel live on the same node, reducing cross-node traffic.
- **The "Silent Room" Strategy**: If a room hasn't had an update in 1 hour, remove the ZSET. Any user joining later will just get a slightly "older" snapshot from Postgres, which is acceptable since no recent changes exist.

### 5. Summary of the Flow

1. **Client Joins**: Receives Snapshot from DB (up to seq 1000).
2. **Server Checks Redis**: Finds seq 1001-1004 in the ZSET.
3. **Client Catch-up**: Receives 1001-1004 via the new WebSocket.
4. **Live Stream**: Receives seq 1005 via Pub/Sub.

This architecture is stateless for the app server, resilient to DB lag, and scalable to millions of concurrent rooms.



## To Build This in Go

To build this in Go, we will focus on the "Catch-up" logic. This code demonstrates how to upgrade an HTTP connection to a WebSocket, fetch the "Source of Truth" from Postgres, and then fill the gap using the "Recent Buffer" in Redis.

### 1. The Strategy: The "Double-Join" Pattern

When a user connects, the server performs these three steps in order:

1. **Freeze the ID**: Query Postgres for the latest notes and note down the Last_Sequence_ID.
2. **Fill the Gap**: Query Redis ZSET for any updates with an ID greater than the Postgres ID.
3. **Go Live**: Connect to the Redis Pub/Sub stream for future updates.

### 2. The Implementation

You will need the `gorilla/websocket` and `redis/go-redis` packages.

```go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"log"
	"net/http"
	"strconv"

	"github.com/gorilla/websocket"
	"github.com/redis/go-redis/v9"
)

var ctx = context.Background()

// Note Structure
type NoteUpdate struct {
	ID      string `json:"id"`
	RoomID  string `json:"room_id"`
	Body    string `json:"body"`
	SeqID   int64  `json:"seq_id"`
}

var upgrader = websocket.Upgrader{
	CheckOrigin: func(r *http.Request) bool { return true },
}

// Global Redis Client
var rdb = redis.NewClient(&redis.Options{
	Addr: "localhost:6379",
})

func handleNoteRoom(w http.ResponseWriter, r *http.Request) {
	roomID := r.URL.Query().Get("roomID")
	conn, err := upgrader.Upgrade(w, r, nil)
	if err != nil {
		log.Println("Upgrade error:", err)
		return
	}
	defer conn.Close()

	// --- STEP 1: Fetch from Postgres (The Baseline) ---
	// In a real app, you'd query your DB here.
	// Assume DB says: "The last sequence I saved is 1000"
	var lastSavedSeq int64 = 1000 
	initialNotes := fetchNotesFromPostgres(roomID)
	conn.WriteJSON(map[string]interface{}{
		"type": "SNAPSHOT",
		"data": initialNotes,
	})

	// --- STEP 2: Fill the Gap from Redis ZSET ---
	// We ask Redis: "Give me everything between (1000 and +infinity]"
	gapUpdates, _ := rdb.ZRangeByScore(ctx, "room:updates:"+roomID, &redis.ZRangeBy{
		Min: "(" + strconv.FormatInt(lastSavedSeq, 10),
		Max: "+inf",
	}).Result()

	for _, updateStr := range gapUpdates {
		var update NoteUpdate
		json.Unmarshal([]byte(updateStr), &update)
		conn.WriteJSON(map[string]interface{}{
			"type": "GAP_FILL",
			"data": update,
		})
	}

	// --- STEP 3: Listen to Real-Time Pub/Sub ---
	pubsub := rdb.Subscribe(ctx, "room:pubsub:"+roomID)
	defer pubsub.Close()

	ch := pubsub.Channel()
	
	// Message Loop
	for msg := range ch {
		// Optimization: We could check if msg.SeqID <= latest sent to avoid duplicates
		// but Step 2 and 3 usually overlap safely if IDs are unique.
		conn.WriteMessage(websocket.TextMessage, []byte(msg.Payload))
	}
}

func fetchNotesFromPostgres(roomID string) []NoteUpdate {
	// Dummy Postgres response
	return []NoteUpdate{{ID: "1", Body: "Initial Note Content", SeqID: 1000}}
}

func main() {
	http.HandleFunc("/ws", handleNoteRoom)
	log.Println("Server started on :8080")
	log.Fatal(http.ListenAndServe(":8080", nil))
}
```
## Key Design Details for 10M Users

### Redis Lua Scripting (The Producer side)

To prevent the "Note D" problem, the server that receives the update must ensure the ZSET and Pub/Sub are updated atomically. Use a Lua script to keep the SeqID incrementing correctly within the room.

### Connection Sharding

With 10M active users, one Go server cannot hold all WebSocket connections (usually limited by RAM and file descriptors). You should:

- **Nginx/HAProxy**: Distribute connections across 100+ Go instances.
- **Redis Pub/Sub**: Since connections are distributed, Server A needs to know what Server B published. Redis Pub/Sub handles this "broadcast" across the entire cluster.

### Memory Management (ZSET Capping)

In your Lua script or Go code, always run `ZREMRANGEBYRANK room:updates:ID 0 -101`. This ensures that even with millions of rooms, Redis only stores the "gap" data (the last 100 messages), while Postgres handles the "cold" history.

### TTL Reset

Use `EXPIRE room:updates:ID 7200` every time a message is sent. If the room is inactive for 2 hours, Redis cleans itself up automatically.

---



## Handle Real-Time Reordering Across Millions of Users

To handle real-time reordering across millions of users while ensuring long-term persistence in Postgres, you need a Reorder Event Pipeline. This involves updating a numerical index (currentIndex) for the notes and broadcasting that change. Because reordering can cause a "shuffle" of many rows, we must optimize how we handle this in Redis versus Postgres.

### 1. Real-Time Reordering Flow

When User A drags a note from position 5 to position 2:

- **Frontend Action**: The UI (React or Bubble Tea) calculates the new currentIndex values for the affected notes.
- **WebSocket Message**: User A sends a REORDER event containing the note IDs and their new indices.
- **Atomic Redis Update**: The Go backend executes a Lua Script in Redis.
    - The script updates the currentIndex inside the Redis ZSET values.
    - It updates the sequence_id for the room (this acts as the version number for the reorder).
- **Pub/Sub Broadcast**: Redis publishes the reorder event to the room channel.
- **UI Sync**: User B's client receives the message and triggers a local list re-sort based on the new indices.

### 2. Persistence in Postgres (The "Deferred Batch" Strategy)

Updating 100 rows in Postgres every time a user drags a note is too expensive for 10M users. Instead, we use a Write-Behind approach.

- **Redis as the Buffer**: The Redis ZSET always holds the current "truth" of the order.
- **Debounced Save**: The backend doesn't write to Postgres immediately. It waits for a "quiet period" (e.g., 5 seconds after the last drag) or a maximum timeout.
- **The Write**: The Go worker pool takes the current state from Redis and performs a Bulk Update in Postgres using a single transaction.

```sql
-- Example of a bulk positional update
UPDATE notes 
SET current_index = data.new_index
FROM (VALUES (101, 1), (102, 2), (105, 3)) AS data(id, new_index)
WHERE notes.id = data.id AND notes.note_room_id = 'room_123';
```

### 3. Handling Reordering Conflicts

In a collaborative environment, User A and User B might try to move the same note simultaneously.

- **Sequence ID as the Lock**: Every reorder event carries a sequence_id.
- **Server Validation**: If User B sends a reorder request based on "Version 10," but the server is already at "Version 11," the server rejects User B's move and sends back the latest state.
- **Last-Write-Wins (LWW)**: For simpler logic, the server accepts the latest move and the sequence_id ensures that any "joining" users get the absolute latest layout from the ZSET gap-fill logic we discussed earlier.

### 4. Database Schema for Ordering

To ensure that after "some days" the order remains exactly the same, your Postgres query must always include an ORDER BY clause.

**Postgres Table: notes**

| Column | Role | Importance |
| :--- | :--- | :--- |
| id | PK | Unique ID for the note. |
| note_room_id | FK | Sharding/Indexing key. |
| current_index | Order Key | A FLOAT or INT used for sorting. |
| updated_at | Metadata | Used to resolve ties. |

```sql
SELECT * FROM notes 
WHERE note_room_id = $1 
ORDER BY current_index ASC;
```

**Tip for 10M users**: Use Fractional Indexing (storing current_index as a float). If you move a note between index 1 and 2, you can simply give it index 1.5 without having to re-index all other notes in the database. This significantly reduces Postgres write load.

## Refferences

- https://stackblitz.com/edit/draggable-mui-list?file=App.tsx
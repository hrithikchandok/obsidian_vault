```
const express = require('express');
const http = require('http');
const WebSocket = require('ws');
const { createClient } = require('redis');

// Generate a unique ID for this server instance (using port for simplicity in testing)
const PORT = process.env.PORT || 3000;
const SERVER_ID = `server-${PORT}`;

const app = express();
app.use(express.json());

const server = http.createServer(app);

// Attach the standard WebSocket server to the HTTP server
const wss = new WebSocket.Server({ server });

// Local map to hold connections: Map<email, WebSocket instance>
const localClients = new Map();

// Initialize Redis Clients
// Note: Redis v4 requires separate clients for standard commands, publishing, and subscribing.
const redisCache = createClient();
const pubClient = redisCache.duplicate();
const subClient = redisCache.duplicate();

async function initRedis() {
    await redisCache.connect();
    await pubClient.connect();
    await subClient.connect();

    // Subscribe to this specific server's channel to listen for cross-server messages
    await subClient.subscribe(`channel:${SERVER_ID}`, (message) => {
        const { email, action } = JSON.parse(message);
        
        if (action === 'toggle' && localClients.has(email)) {
            const ws = localClients.get(email);
            // Send the toggle event to the locally connected client
            ws.send(JSON.stringify({ message: 'Toggle event triggered from another server instance!' }));
            console.log(`[${SERVER_ID}] Handled toggle for ${email} via Pub/Sub`);
        }
    });
    
    console.log(`[${SERVER_ID}] Redis connected and subscribed to channel:${SERVER_ID}`);
}
initRedis().catch(console.error);

// ---------------------------------------------------------
// Route 1: WebSocket Connection (ws://localhost:PORT?email=xxx)
// ---------------------------------------------------------
wss.on('connection', async (ws, req) => {
    // Extract email from query params for simplicity
    const url = new URL(req.url, `http://${req.headers.host}`);
    const email = url.searchParams.get('email');

    if (!email) {
        ws.close(1008, 'Email is required');
        return;
    }

    console.log(`[${SERVER_ID}] New WS connection for ${email}`);
    
    // Store in local map
    localClients.set(email, ws);
    
    // Store routing info in Redis: email -> SERVER_ID
    await redisCache.set(`user:${email}`, SERVER_ID);

    ws.on('close', async () => {
        console.log(`[${SERVER_ID}] WS disconnected for ${email}`);
        localClients.delete(email);
        
        // Only delete from Redis if it still points to THIS server
        // (Handles the edge case where the user reconnected to a new server before the old connection fully closed)
        const currentServer = await redisCache.get(`user:${email}`);
        if (currentServer === SERVER_ID) {
            await redisCache.del(`user:${email}`);
        }
    });
});

// ---------------------------------------------------------
// Route 2: HTTP Toggle Request (POST /toggle)
// ---------------------------------------------------------
app.post('/toggle', async (req, res) => {
    const { email } = req.body;

    if (!email) return res.status(400).json({ error: 'Email is required' });

    // Look up which server holds the WebSocket connection for this email
    const targetServerId = await redisCache.get(`user:${email}`);

    if (!targetServerId) {
        return res.status(404).json({ error: 'User is not currently connected' });
    }

    if (targetServerId === SERVER_ID) {
        // The user is connected to THIS server instance. Handle it directly.
        const ws = localClients.get(email);
        if (ws) {
            ws.send(JSON.stringify({ message: 'Toggle event triggered locally!' }));
            console.log(`[${SERVER_ID}] Handled toggle locally for ${email}`);
            return res.status(200).json({ success: true, routed: false });
        }
    } else {
        // The user is connected to a DIFFERENT server instance. Broadcast via Pub/Sub.
        console.log(`[${SERVER_ID}] Forwarding toggle for ${email} to ${targetServerId}`);
        await pubClient.publish(
            `channel:${targetServerId}`, 
            JSON.stringify({ email, action: 'toggle' })
        );
        return res.status(200).json({ success: true, routed: true, targetServer: targetServerId });
    }
});

server.listen(PORT, () => {
    console.log(`[${SERVER_ID}] Listening on port ${PORT}`);
});
```

Explanation 
Awesome, I am glad to hear you got it up and running!

Understanding how this code bridges the gap between different server instances is crucial for scaling backend architectures. Let's break down the `server.js` file step-by-step so you can see exactly how the magic happens.

### 1. The Setup & Imports

JavaScript

```
const express = require('express');
const http = require('http');
const WebSocket = require('ws');
const { createClient } = require('redis');

const PORT = process.env.PORT || 3000;
const SERVER_ID = `server-${PORT}`;
```

- **The Core Libraries:** We pull in `express` for standard REST APIs, `http` to create the foundational web server, `ws` for WebSockets, and `redis` to talk to our Redis database.
    
- **Server Identity:** We generate a unique `SERVER_ID`. In a real production environment (like AWS ECS or Kubernetes), this might be a random UUID, a container ID, or an EC2 instance ID. We use the port number here just to make local testing easier to read.
    

### 2. Merging HTTP and WebSockets

JavaScript

```
const app = express();
app.use(express.json());

const server = http.createServer(app);
const wss = new WebSocket.Server({ server });
const localClients = new Map();
```

- **The Shared Server:** Notice that we don't call `app.listen()`. Instead, we pass the Express `app` into `http.createServer()`. We then pass _that_ server into our `WebSocket.Server`. This allows both standard HTTP requests (like POST `/toggle`) and WebSocket upgrade requests (like `ws://...`) to live side-by-side on the exact same port.
    
- **`localClients`:** This is our in-memory map. It only knows about the users connected directly to _this specific instance_.
    

### 3. The Three Redis Clients

JavaScript

```
const redisCache = createClient();
const pubClient = redisCache.duplicate();
const subClient = redisCache.duplicate();
```

- **Why three?** The modern Node Redis library (v4+) requires dedicated clients for different tasks.
    
    - `redisCache` is used for standard database commands (like `SET` and `GET`).
        
    - `subClient` is put into "subscription mode." Once a client is listening for channels, it _cannot_ be used to send other commands.
        
    - `pubClient` is used to broadcast messages out to other servers.
        

### 4. Listening for Cross-Server Messages

JavaScript

```
await subClient.subscribe(`channel:${SERVER_ID}`, (message) => {
    const { email, action } = JSON.parse(message);
    
    if (action === 'toggle' && localClients.has(email)) {
        const ws = localClients.get(email);
        ws.send(JSON.stringify({ message: 'Toggle event...' }));
    }
});
```

- **The Mailbox:** Every server instance subscribes to a unique Redis channel named after itself (e.g., `channel:server-3001`). It sits there quietly listening.
    
- **The Action:** If another server sends a message to this channel, this callback fires. The server checks if it actually holds the WebSocket for that email. If it does, it pushes the message down the socket to the client's browser.
    

### 5. Handling New WebSocket Connections

JavaScript

```
wss.on('connection', async (ws, req) => {
    // ... extract email ...
    localClients.set(email, ws);
    await redisCache.set(`user:${email}`, SERVER_ID);
```

- **The Registry:** When a user connects, the server saves their connection object in the local memory map (`localClients`).
    
- **The Global State:** It then writes a record to Redis: `"user:dev@example.com" -> "server-3001"`. This acts as a global directory. Now, _any_ server in the cluster can ask Redis, "Where is dev@example.com?" and get the answer.
    

### 6. Handling Disconnections Safely

JavaScript

```
ws.on('close', async () => {
    localClients.delete(email);
    
    const currentServer = await redisCache.get(`user:${email}`);
    if (currentServer === SERVER_ID) {
        await redisCache.del(`user:${email}`);
    }
});
```

- **Cleanup:** When a user drops off, we remove them from local memory.
    
- **The Edge Case:** Why do we check `currentServer === SERVER_ID` before deleting from Redis? Imagine a user has a spotty mobile connection. They disconnect from Server A, instantly reconnect, and the load balancer sends them to Server B. If Server A's slow "close" event fires _after_ Server B registers the new connection, Server A might accidentally delete the brand new, valid Redis record! This check prevents that race condition.
    

### 7. The Routing Logic (The HTTP Route)

JavaScript

```
app.post('/toggle', async (req, res) => {
    // ... check email ...
    const targetServerId = await redisCache.get(`user:${email}`);
```

- **The Lookup:** An HTTP request comes in to trigger a toggle. The server immediately asks Redis where this user's active WebSocket is currently living.
    

JavaScript

```
    if (targetServerId === SERVER_ID) {
        // Handle Locally
        const ws = localClients.get(email);
        ws.send(JSON.stringify({ message: 'Triggered locally!' }));
```

- **Scenario A (Lucky Hit):** The load balancer coincidentally sent the HTTP request to the exact same server holding the WebSocket connection. The server bypasses Redis Pub/Sub entirely and just sends the message directly using local memory.
    

JavaScript

```
    } else {
        // Broadcast via Pub/Sub
        await pubClient.publish(
            `channel:${targetServerId}`, 
            JSON.stringify({ email, action: 'toggle' })
        );
```

- **Scenario B (Distributed Routing):** The user is connected to a different server. The current server uses its `pubClient` to shout into the target server's specific Redis channel. The target server (which is listening via step 4) hears it and pushes the data to the user.
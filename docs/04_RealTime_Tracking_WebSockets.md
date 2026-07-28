# 4. Real-Time Tracking & WebSockets

When buses are moving, they send their GPS coordinates to the server every few seconds. If we save every single coordinate update directly to the PostgreSQL database, the database might slow down or crash due to the high volume of writes.

To solve this, we use **Redis** (an in-memory, super-fast storage) to hold the *current* location of the bus, and **Socket.io** (WebSockets) to broadcast that location to the mobile apps instantly.

## 1. Setting up Socket.io & Redis

Below is an example of how the Node.js server listens for GPS updates from the bus, saves it to Redis, and broadcasts it to any connected students.

```typescript
// server.ts
import express from 'express';
import http from 'http';
import { Server } from 'socket.io';
import { createClient } from 'redis';

const app = express();
const server = http.createServer(app);
const io = new Server(server, {
  cors: { origin: "*" }
});

// Setup Redis Client
const redisClient = createClient({ url: process.env.REDIS_URL });
redisClient.connect().catch(console.error);

// When a mobile app or a bus connects to the WebSocket
io.on('connection', (socket) => {
  console.log('A client connected:', socket.id);

  // 1. Bus sends its location every 3 seconds
  socket.on('bus_location_update', async (data) => {
    const { bus_id, latitude, longitude, speed } = data;

    // Save the latest location in Redis (overwrites the old one)
    // Key: bus_location:UUID
    await redisClient.set(\`bus_location:\${bus_id}\`, JSON.stringify({
      latitude,
      longitude,
      speed,
      timestamp: Date.now()
    }));

    // Broadcast this location ONLY to users interested in this bus
    // We use Socket.io "rooms". The room name is the bus_id.
    io.to(bus_id).emit('live_location', { latitude, longitude, speed });
  });

  // 2. Student joins a "room" to listen for their specific bus
  socket.on('subscribe_to_bus', async (bus_id) => {
    socket.join(bus_id);
    console.log(\`Socket \${socket.id} joined room: \${bus_id}\`);

    // Immediately send the student the last known location from Redis
    const lastLocation = await redisClient.get(\`bus_location:\${bus_id}\`);
    if (lastLocation) {
      socket.emit('live_location', JSON.parse(lastLocation));
    }
  });

  socket.on('disconnect', () => {
    console.log('Client disconnected:', socket.id);
  });
});

server.listen(3000, () => {
  console.log('Server is running on port 3000');
});
```

## How the Mobile App uses this:
1. When John (the student) opens the app, the app knows he is assigned to `Bus-11`.
2. The app connects to the Socket.io server and emits `subscribe_to_bus("Bus-11-UUID")`.
3. The app instantly receives the last known location from Redis.
4. As the bus drives, it emits `bus_location_update` to the server.
5. The server pushes `live_location` to John's app, and the bus icon moves on his map!

In the next and final document, we will see how the Flutter app is structured to handle this map and WebSocket logic.

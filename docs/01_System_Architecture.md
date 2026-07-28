# 1. System Architecture Overview

Welcome to the Campus Bus Tracking & Automated Attendance System! Since you are a beginner, we will break down the entire platform into understandable pieces. 

This platform consists of **four main components** that talk to each other in real-time.

## The Big Picture (How it Works)
1. **The IoT Cameras (On the Bus):** A small computer with a camera (like a Raspberry Pi or an edge device) is installed on every bus. When a student boards the bus, the camera scans their face, identifies them using Facial Recognition, and instantly sends their attendance record to the central server via 4G/5G.
2. **The Backend Server (The Brain):** We are using **Node.js (TypeScript)**. This acts as the central hub. It receives the facial recognition data, marks the student as present in the database, and also receives continuous GPS locations from the bus.
3. **The Database (The Memory):** We are using **PostgreSQL** with a special plugin called **PostGIS**. PostGIS allows us to store and query geographical data (like "Is the bus inside this geofenced area?"). We are using **Prisma** as the ORM (Object-Relational Mapper) which is a tool that lets our Node.js code talk to the database easily.
4. **The Mobile App (For Students & Parents):** We will use **Flutter** for the mobile app because it builds beautiful, high-performance apps for both iOS and Android from a single codebase. It connects to the Backend Server using WebSockets to see the bus moving on a map in real-time.

---

## Detailed Tech Stack Selection

Based on your preferences, here is our finalized technology stack:

- **Backend Runtime:** Node.js with TypeScript and Express.js framework. (TypeScript helps prevent bugs by enforcing strict variable types).
- **Database:** PostgreSQL.
- **Geospatial Engine:** PostGIS (A PostgreSQL extension for GPS logic).
- **Database ORM:** Prisma (Excellent developer experience for Node.js).
- **Real-Time Data (Cache):** Redis (To handle hundreds of GPS updates per second without crashing the main database).
- **WebSockets:** Socket.io (For streaming live locations to the mobile app).
- **Mobile App:** Flutter (with Google Maps or Mapbox integration).

---

## The Data Flow (A Real-Life Scenario)

Let's trace what happens when John, a student, takes the bus in the morning.

1. **Bus Starts Trip:** The driver starts the bus. The onboard GPS starts sending coordinates to the Node.js Server every 3 seconds via WebSockets.
2. **Server Updates Redis:** The Node.js server receives these coordinates and quickly saves the latest location in **Redis** (super fast in-memory storage). It also broadcasts this new location to any student watching the app.
3. **John Opens App:** John opens his Flutter mobile app. The app connects to the server and instantly receives the bus's live location from Redis. John sees the bus icon moving on his map.
4. **John Boards Bus:** John steps onto the bus. The IoT Camera scans his face.
5. **IoT Sends Data:** The camera sends a secure HTTP request (`POST /api/v1/iot/attendance/scan`) to the Node.js server with John's ID and the exact time and GPS location.
6. **Attendance Logged:** The Node.js server verifies John's fee status, confirms he belongs on this route, and saves an `attendance_logs` record in the PostgreSQL database.
7. **Notification Sent:** The server can trigger a Push Notification (via Firebase) to John's parents, saying: *"John has boarded Bus-11 safely."*

In the next document, we will look at exactly how we design the Database to store all this information!

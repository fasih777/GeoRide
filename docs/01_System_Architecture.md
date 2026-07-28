# 1. System Architecture Overview

Welcome to the Campus Bus Tracking & Automated Attendance System! Since you are a beginner, we will break down the entire platform into understandable pieces. 

This platform consists of **three main web portals** and a **mobile app** that talk to each other in real-time.

## The Big Picture (How it Works)
1. **The 3 Web Portals:**
   - **User Portal:** For students and parents to manage their profiles, pay fees, and view history.
   - **Admin Portal:** For college transport administrators to allocate buses, manage routes, and oversee the entire fleet.
   - **Bus In-Charge Portal:** A dedicated, mobile-friendly web application used exclusively by the staff member in charge of the bus. They use this to take manual attendance as students board.
2. **The Backend Server (The Brain):** We are using **Node.js (TypeScript)**. This acts as the central hub. It receives the manual attendance data from the Bus In-Charge, marks the student as present in the database, triggers the SMS notifications to parents, and receives continuous GPS locations.
3. **The Database (The Memory):** We are using **PostgreSQL** with a special plugin called **PostGIS**. PostGIS allows us to store and query geographical data (e.g., calculating when a bus is exactly 500 meters away from campus). We are using **Prisma** as our ORM.
4. **The Mobile App (Live Tracking):** Built with **Flutter**, this connects to the Backend Server using WebSockets to see the bus moving on a map in real-time.

---

## Detailed Tech Stack Selection

- **Backend Runtime:** Node.js with TypeScript and Express.js framework.
- **Database:** PostgreSQL with PostGIS.
- **Database ORM:** Prisma.
- **Real-Time Data:** Redis (for GPS buffers) and Socket.io (for WebSockets).
- **Web Portals:** React.js or Next.js (Tailwind CSS for styling).
- **Mobile App:** Flutter (with Google Maps integration).
- **Notification Engine:** Twilio or Firebase Cloud Messaging for SMS and push notifications.

---

## The Data Flow (A Real-Life Scenario)

Let's trace what happens when John, a student, takes the bus in the morning.

1. **Bus Starts Trip:** The driver starts the bus. A GPS tracker (or the In-Charge's device) starts sending coordinates to the Node.js Server.
2. **John Boards Bus:** John steps onto the bus. The Bus In-Charge opens their dedicated Web Portal on a tablet and taps "Present" next to John's name.
3. **Backend Processes Attendance:** The web app sends a request to the Node.js server. The server logs John's attendance in the database.
4. **Notification Sent:** The server immediately sends an SMS to John's parents: 
   *"Your ward John (12345) boarded Bus Bus-11 at Mehdipatnam. Date: 2026-07-28, Time: 08:00 AM. Track live bus location here: https://georide.app/track/bus-11"*
5. **Absence Check (Geofencing):** If John *didn't* board, the server waits until the bus arrives at the college campus. Once the GPS geofence triggers "Arrived at College", the server sends an absent SMS for all students who didn't board.

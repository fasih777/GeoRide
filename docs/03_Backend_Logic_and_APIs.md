# 3. Backend Logic & APIs

Our Node.js backend handles the "thinking" for our system. It is responsible for logging student attendance and deciding which bus a student should be assigned to based on where they live.

## 1. Admin Onboarding & Student Allocation Engine

When an Admin adds a student to the system, they need to verify their fee and allocate them a bus.

### A. Fee Verification Endpoint
This API confirms the student has paid their transport fees.

```typescript
// routes/admin.ts
import { Router } from 'express';
import { PrismaClient } from '@prisma/client';

const router = Router();
const prisma = new PrismaClient();

// POST /api/v1/admin/verify-student
router.post('/verify-student', async (req, res) => {
  try {
    const { student_id } = req.body;
    
    // Update the fee_verified flag to true
    const updatedStudent = await prisma.student.update({
      where: { id: student_id },
      data: { fee_verified: true }
    });

    res.json({ message: "Student fee verified successfully", student: updatedStudent });
  } catch (error) {
    res.status(500).json({ error: "Failed to verify student fee" });
  }
});
```

### B. Auto-Allocation Logic
When a student selects a `designated_stop_id` (like "Mehdipatnam"), the system automatically finds buses that pass through that stop and assigns one.

```typescript
// services/allocationService.ts
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient();

export async function allocateBusToStudent(studentId: string, stopId: string) {
  // 1. Find all routes that pass through the requested stop
  const availableRoutes = await prisma.routeStop.findMany({
    where: { stop_id: stopId },
    include: { route: true }
  });

  if (availableRoutes.length === 0) {
    throw new Error("No routes found for this stop.");
  }

  // 2. Select the first available route (Logic can be expanded for capacity checking)
  const selectedRouteId = availableRoutes[0].route_id;

  // 3. Find a bus assigned to this route (Assuming we have a BusRoute relation, 
  // or we just pick a default bus for simplicity in this example)
  const defaultBus = await prisma.bus.findFirst({
    where: { current_status: 'IDLE' }
  });

  if (!defaultBus) throw new Error("No buses available.");

  // 4. Update the student's allocated bus
  const updatedStudent = await prisma.student.update({
    where: { id: studentId },
    data: { 
      designated_stop_id: stopId,
      allocated_bus_id: defaultBus.id 
    }
  });

  return updatedStudent;
}
```

## 2. IoT Onboard Facial Recognition API

When the camera scans a face on the bus, it sends an HTTP POST request to our server. We must ensure this request is authentic using a secret key (`x-bus-device-key`).

```typescript
// routes/iot.ts
import { Router } from 'express';
import { PrismaClient } from '@prisma/client';

const router = Router();
const prisma = new PrismaClient();

// POST /api/v1/iot/attendance/scan
router.post('/attendance/scan', async (req, res) => {
  try {
    // 1. Authenticate the Bus Device
    const deviceKey = req.headers['x-bus-device-key'];
    if (deviceKey !== process.env.BUS_SECRET_KEY) {
      return res.status(401).json({ error: "Unauthorized IoT Device" });
    }

    const { 
      bus_id, 
      student_admission_number, 
      timestamp, 
      latitude, 
      longitude, 
      trip_type 
    } = req.body;

    // 2. Find the student by admission number
    const student = await prisma.student.findUnique({
      where: { admission_number: student_admission_number }
    });

    if (!student) {
      return res.status(404).json({ error: "Student not found" });
    }

    // 3. Optional: Verify if the student is assigned to this bus
    if (student.allocated_bus_id !== bus_id) {
      console.warn(\`Student \${student.id} boarded the wrong bus!\`);
      // We can trigger an alert here!
    }

    // 4. Log the attendance using raw SQL to handle the PostGIS Geometry
    const scanDate = new Date(timestamp);
    
    // We use Prisma's $executeRaw for PostGIS insertion
    await prisma.$executeRaw\`
      INSERT INTO "AttendanceLog" (
        "id", "student_id", "bus_id", "trip_type", "status", "scan_timestamp", "date", "scan_location"
      ) VALUES (
        gen_random_uuid(), 
        \${student.id}::uuid, 
        \${bus_id}::uuid, 
        \${trip_type}::"TripType", 
        'BOARDED'::"AttendanceStatus", 
        \${scanDate}, 
        \${scanDate},
        ST_SetSRID(ST_MakePoint(\${longitude}, \${latitude}), 4326)
      )
    \`;

    // 5. Here you would trigger Firebase Cloud Messaging (FCM) to the parents
    // sendPushNotificationToParent(student.parent_id, "Boarded safely");

    res.status(201).json({ message: "Attendance logged successfully" });

  } catch (error) {
    console.error(error);
    res.status(500).json({ error: "Internal Server Error" });
  }
});
```

In the next document, we will look at how we stream the Bus GPS locations to the Mobile App in real-time!

//test line 
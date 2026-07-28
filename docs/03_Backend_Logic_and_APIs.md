# 3. Backend Logic, APIs, & Notification Engine

Our Node.js backend handles manual attendance entries from the Bus In-Charge and orchestrates the SMS notification rules based on your specific requirements.

## 1. Bus In-Charge Attendance Endpoint

When the Bus In-Charge taps "Present" for a student on their web portal, it hits this endpoint.

```typescript
// routes/incharge.ts
import { Router } from 'express';
import { PrismaClient } from '@prisma/client';
import { sendSMS } from '../services/smsService';

const router = Router();
const prisma = new PrismaClient();

// POST /api/v1/incharge/attendance
router.post('/attendance', async (req, res) => {
  try {
    const { incharge_id, bus_id, student_id, trip_type, timestamp, location_name } = req.body;

    // 1. Verify In-Charge owns this bus (Security check omitted for brevity)
    
    // 2. Fetch Student and Parent Info
    const student = await prisma.student.findUnique({
      where: { id: student_id },
      include: { parent: true, user: true }
    });

    if (!student) return res.status(404).json({ error: "Student not found" });

    // 3. Log the attendance in Database
    const scanDate = new Date(timestamp);
    await prisma.attendanceLog.create({
      data: {
        student_id: student.id,
        bus_id: bus_id,
        trip_type: trip_type,
        status: 'BOARDED',
        scan_timestamp: scanDate,
        date: scanDate
      }
    });

    // 4. Trigger the Notification based on Trip Type
    const liveLink = \`https://georide.app/track/\${bus_id}\`;
    const formattedDate = scanDate.toLocaleDateString();
    const formattedTime = scanDate.toLocaleTimeString();

    let smsBody = "";

    if (trip_type === 'MORNING_INBOUND') {
      smsBody = \`Your ward \${student.user.full_name} (\${student.admission_number}) boarded Bus \${bus_id} at \${location_name}. Date: \${formattedDate}, Time: \${formattedTime}. Track live bus location here: \${liveLink}\`;
    } else if (trip_type === 'EVENING_OUTBOUND') {
      smsBody = \`Your ward \${student.user.full_name} (\${student.admission_number}) boarded Bus \${bus_id} for the evening drop-off. Date: \${formattedDate}, Time: \${formattedTime}. Track live bus location here: \${liveLink}\`;
    }

    // 5. Send SMS and Log it
    await sendSMS(student.parent.phone_number, smsBody);
    
    await prisma.notificationLog.create({
      data: {
        parent_id: student.parent.id,
        student_id: student.id,
        message_body: smsBody
      }
    });

    res.status(201).json({ message: "Attendance logged and SMS sent successfully" });

  } catch (error) {
    res.status(500).json({ error: "Internal Server Error" });
  }
});
```

## 2. Geofenced Absence Triggers

Absence notifications are *not* triggered immediately when the bus starts. They are triggered based on geographical events.

### Morning Absence Trigger
**Rule:** *"Triggered upon bus arrival at college"*

**Implementation:**
When the continuous GPS stream (via WebSockets) detects that the bus coordinates have entered the College Campus Geofence, the backend runs a job:
1. Find all students allocated to this bus.
2. Filter out students who already have a `BOARDED` status for today's `MORNING_INBOUND`.
3. For the remaining students, log them as `ABSENT` and send the SMS:
   > *"Your ward [Student_Name] ([Roll_Number]) was marked absent on the morning bus route today, [Date]."*

### Evening Absence Trigger
**Rule:** *"Triggered 500 meters after campus departure"*

**Implementation:**
Using **PostGIS**, we calculate the distance between the bus's live GPS coordinates and the campus boundary.
When `ST_Distance(bus_location, campus_location) > 500 meters`:
1. The backend triggers the evening absence job.
2. Finds allocated students without a `BOARDED` status for `EVENING_OUTBOUND`.
3. Sends the SMS:
   > *"Alert: Your ward [Student_Name] ([Roll_Number]) did not board the evening bus today, [Date]. Bus departed campus at [Timestamp]."*
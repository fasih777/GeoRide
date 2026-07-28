# 2. Database Schema & Prisma Models

For our database, we are using **PostgreSQL** combined with **Prisma ORM**. 

### What is Prisma?
Prisma is a modern tool for Node.js that replaces writing raw SQL queries (like `SELECT * FROM users`). Instead, you define your database structure in a file called `schema.prisma`. Prisma then reads this file and automatically creates the database tables for you, and gives you auto-completing JavaScript code to read/write data.

### PostGIS
Because we are tracking buses, we need to store GPS coordinates (Latitude/Longitude). Standard databases aren't great at asking questions like *"Find all stops within 500 meters of this location."* The **PostGIS** extension gives PostgreSQL "superpowers" to do exactly that using a special data type called `Geometry`.

---

## The `schema.prisma` File

Below is the complete, production-ready Prisma schema for your system. We use standard UUIDs for all IDs to ensure absolute uniqueness across the system.

```prisma
// prisma/schema.prisma

generator client {
  provider        = "prisma-client-js"
  // We enable the postgis extension feature if needed by specific prisma versions
  previewFeatures = ["postgresqlExtensions"]
}

datasource db {
  provider   = "postgresql"
  url        = env("DATABASE_URL")
  extensions = [postgis(version: "3.3.2")] // Enable PostGIS
}

enum Role {
  ADMIN
  STUDENT
  PARENT
}

enum BusStatus {
  IDLE
  IN_TRANSIT
  STOPPED_TRAFFIC
  BREAKDOWN
  COMPLETED
}

enum TripType {
  MORNING_INBOUND
  EVENING_OUTBOUND
}

enum AttendanceStatus {
  BOARDED
  ABSENT
}

model User {
  id           String   @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  full_name    String   @db.VarChar(255)
  role         Role
  phone_number String   @unique @db.VarChar(20)
  created_at   DateTime @default(now()) @db.Timestamptz(6)

  // Relationships
  student_profile Student? @relation("StudentProfile")
  parent_students Student[] @relation("ParentToStudent")
}

model Student {
  id                 String   @id @db.Uuid // Matches User.id
  admission_number   String   @unique @db.VarChar(50)
  department         String   @db.VarChar(100)
  parent_id          String   @db.Uuid
  fee_verified       Boolean  @default(false)
  designated_stop_id String   @db.Uuid
  allocated_bus_id   String   @db.Uuid

  // Relationships
  user            User             @relation("StudentProfile", fields: [id], references: [id], onDelete: Cascade)
  parent          User             @relation("ParentToStudent", fields: [parent_id], references: [id])
  designated_stop BusStop          @relation(fields: [designated_stop_id], references: [id])
  allocated_bus   Bus              @relation(fields: [allocated_bus_id], references: [id])
  attendance_logs AttendanceLog[]
}

model BusStop {
  id         String   @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  stop_name  String   @db.VarChar(100)
  
  // PostGIS Geometry column for Lat/Lng
  // Prisma handles PostGIS using Unsupported("geometry(Point, 4326)")
  location   Unsupported("geometry(Point, 4326)")

  // Relationships
  students    Student[]
  route_stops RouteStop[]
}

model Bus {
  id            String    @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  bus_number    String    @unique @db.VarChar(20)
  driver_name   String    @db.VarChar(100)
  driver_phone  String    @db.VarChar(20)
  current_status BusStatus @default(IDLE)
  status_reason String?   @db.Text

  // Relationships
  students        Student[]
  attendance_logs AttendanceLog[]
}

model Route {
  id         String   @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  route_name String   @db.VarChar(100)

  // Relationships
  route_stops RouteStop[]
}

model RouteStop {
  id         String   @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  route_id   String   @db.Uuid
  stop_id    String   @db.Uuid
  stop_order Int

  // Relationships
  route      Route    @relation(fields: [route_id], references: [id], onDelete: Cascade)
  bus_stop   BusStop  @relation(fields: [stop_id], references: [id], onDelete: Cascade)

  @@unique([route_id, stop_order])
}

model AttendanceLog {
  id             String           @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  student_id     String           @db.Uuid
  bus_id         String           @db.Uuid
  trip_type      TripType
  status         AttendanceStatus
  scan_timestamp DateTime?        @db.Timestamptz(6)
  
  // GPS location where the scan happened
  scan_location  Unsupported("geometry(Point, 4326)")?
  date           DateTime         @db.Date

  // Relationships
  student        Student @relation(fields: [student_id], references: [id])
  bus            Bus     @relation(fields: [bus_id], references: [id])
}
```

### Note on PostGIS in Prisma
Notice the `Unsupported("geometry(Point, 4326)")` type. Prisma does not have native, fully typed support for PostGIS geometry yet. We define it as unsupported in the schema so Prisma creates the database tables correctly. However, when we write or read locations, we will use Prisma's `$queryRaw` to execute raw SQL commands to interact with these specific location fields (e.g., `ST_GeomFromText('POINT(longitude latitude)', 4326)`).

In the next document, we will build the Backend APIs that interact with this database.

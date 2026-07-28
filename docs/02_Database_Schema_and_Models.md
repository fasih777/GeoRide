# 2. Database Schema & Prisma Models

For our database, we are using **PostgreSQL** combined with **Prisma ORM**. 

## The `schema.prisma` File

Below is the complete, production-ready Prisma schema updated for the 3-portal architecture.

```prisma
generator client {
  provider        = "prisma-client-js"
  previewFeatures = ["postgresqlExtensions"]
}

datasource db {
  provider   = "postgresql"
  url        = env("DATABASE_URL")
  extensions = [postgis(version: "3.3.2")]
}

enum Role {
  ADMIN
  STUDENT
  PARENT
  BUS_INCHARGE  // Added new role
}

enum BusStatus {
  IDLE
  IN_TRANSIT
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

  student_profile Student? @relation("StudentProfile")
  parent_students Student[] @relation("ParentToStudent")
  managed_buses   Bus[]    @relation("BusIncharge")
}

model Student {
  id                 String   @id @db.Uuid
  admission_number   String   @unique @db.VarChar(50)
  department         String   @db.VarChar(100)
  parent_id          String   @db.Uuid
  fee_verified       Boolean  @default(false)
  designated_stop_id String   @db.Uuid
  allocated_bus_id   String   @db.Uuid

  user            User             @relation("StudentProfile", fields: [id], references: [id], onDelete: Cascade)
  parent          User             @relation("ParentToStudent", fields: [parent_id], references: [id])
  designated_stop BusStop          @relation(fields: [designated_stop_id], references: [id])
  allocated_bus   Bus              @relation(fields: [allocated_bus_id], references: [id])
  attendance_logs AttendanceLog[]
}

model BusStop {
  id         String   @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  stop_name  String   @db.VarChar(100)
  
  // PostGIS Geometry for Lat/Lng
  location   Unsupported("geometry(Point, 4326)")

  students    Student[]
  route_stops RouteStop[]
}

model Bus {
  id            String    @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  bus_number    String    @unique @db.VarChar(20)
  incharge_id   String?   @db.Uuid // Links to User with BUS_INCHARGE role
  current_status BusStatus @default(IDLE)

  incharge        User?           @relation("BusIncharge", fields: [incharge_id], references: [id])
  students        Student[]
  attendance_logs AttendanceLog[]
}

model Route {
  id         String   @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  route_name String   @db.VarChar(100)

  route_stops RouteStop[]
}

model RouteStop {
  id         String   @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  route_id   String   @db.Uuid
  stop_id    String   @db.Uuid
  stop_order Int

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
  date           DateTime         @db.Date

  student        Student @relation(fields: [student_id], references: [id])
  bus            Bus     @relation(fields: [bus_id], references: [id])
}

// New Model to track SMS/Notifications sent
model NotificationLog {
  id             String   @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  parent_id      String   @db.Uuid
  student_id     String   @db.Uuid
  message_body   String   @db.Text
  sent_at        DateTime @default(now()) @db.Timestamptz(6)
}
```

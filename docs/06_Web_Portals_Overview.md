# 6. The 3-Portal Web Application Overview

To replace the IoT camera system, we are building a suite of interconnected web applications. These portals will be responsive (usable on phones, tablets, and desktops) and built using a modern frontend framework like **Next.js** or **React.js** with **Tailwind CSS**.

## 1. Bus In-Charge Portal (The Core Tool)
**Primary User:** The staff member riding the bus.
**Device:** Mobile Phone or Tablet (provided by the college).

### Features:
- **Daily Trip Initialization:** The in-charge selects "Start Morning Trip" or "Start Evening Trip".
- **Live Roster:** Displays a clean, scrollable list of all students assigned to that specific bus route.
- **One-Tap Attendance:** 
  - Each student row has a large, green "Mark Present" button.
  - Tapping this button instantly hits our backend API, logging the attendance and triggering the parent SMS.
- **Emergency Override:** Ability to manually mark a student as absent before the geofence trigger, or report a breakdown.

## 2. Admin Portal
**Primary User:** Transport Department / College Administrators.
**Device:** Desktop Computer.

### Features:
- **Fleet Dashboard:** A map view showing the live location of all 30 buses simultaneously.
- **Student Allocation Engine:** Drag-and-drop interface to assign students to specific routes and stops based on fee verification.
- **Route Management:** Define GPS geofences for the campus and draw the routes on a map.
- **Reporting:** Export daily attendance reports and view the `NotificationLog` to ensure parents are receiving SMS alerts successfully.

## 3. User Portal
**Primary User:** Parents and Students.
**Device:** Desktop or Mobile Browser.

### Features:
- **Profile Management:** Update contact numbers (crucial for SMS alerts).
- **Fee Payment & Verification:** Upload transport fee receipts or pay online.
- **History:** View a calendar showing the student's daily bus attendance records.
- *(Note: Live tracking is primarily handled by the Flutter Mobile App, but a simplified web tracking view can be included here).*

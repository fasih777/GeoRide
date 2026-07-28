# 5. Mobile App Integration (Flutter)

For the mobile app, we use **Flutter**. Flutter allows us to build an app for both iOS and Android using the **Dart** programming language. 

To display the map and connect to our Node.js WebSocket server, we will use two main packages:
1. `google_maps_flutter` (or `mapbox_gl`): To show the interactive map.
2. `socket_io_client`: To connect to our Node.js server and listen for live GPS updates.

## Flutter App Structure

Here is a simplified example of how the Flutter app connects to the WebSocket and updates the Google Map when the bus moves.

### 1. Add Dependencies (`pubspec.yaml`)
```yaml
dependencies:
  flutter:
    sdk: flutter
  google_maps_flutter: ^2.5.0
  socket_io_client: ^2.0.3
```

### 2. The Live Tracking Screen (`live_tracking_screen.dart`)

```dart
import 'package:flutter/material.dart';
import 'package:google_maps_flutter/google_maps_flutter.dart';
import 'package:socket_io_client/socket_io_client.dart' as IO;

class LiveTrackingScreen extends StatefulWidget {
  final String busId;

  LiveTrackingScreen({required this.busId});

  @override
  _LiveTrackingScreenState createState() => _LiveTrackingScreenState();
}

class _LiveTrackingScreenState extends State<LiveTrackingScreen> {
  late IO.Socket socket;
  GoogleMapController? mapController;
  
  // Starting position (default fallback)
  LatLng busLocation = LatLng(17.3916, 78.4482); 
  Set<Marker> markers = {};

  @override
  void initState() {
    super.initState();
    initSocket();
  }

  void initSocket() {
    // 1. Connect to the Node.js Server
    socket = IO.io('http://YOUR_BACKEND_IP:3000', <String, dynamic>{
      'transports': ['websocket'],
      'autoConnect': true,
    });

    socket.onConnect((_) {
      print('Connected to WebSockets');
      
      // 2. Tell the server we want updates for our specific bus
      socket.emit('subscribe_to_bus', widget.busId);
    });

    // 3. Listen for the 'live_location' event emitted by Node.js
    socket.on('live_location', (data) {
      double lat = data['latitude'];
      double lng = data['longitude'];

      // 4. Update the map and marker when we receive a new location
      setState(() {
        busLocation = LatLng(lat, lng);
        markers.clear();
        markers.add(
          Marker(
            markerId: MarkerId('bus_marker'),
            position: busLocation,
            icon: BitmapDescriptor.defaultMarkerWithHue(BitmapDescriptor.hueBlue),
            infoWindow: InfoWindow(title: 'Your Bus'),
          ),
        );
      });

      // Move the map camera to follow the bus
      mapController?.animateCamera(
        CameraUpdate.newLatLng(busLocation),
      );
    });
  }

  @override
  void dispose() {
    socket.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text('Live Bus Tracking')),
      body: GoogleMap(
        initialCameraPosition: CameraPosition(
          target: busLocation,
          zoom: 15.0,
        ),
        markers: markers,
        onMapCreated: (GoogleMapController controller) {
          mapController = controller;
        },
      ),
    );
  }
}
```

### Summary of the Mobile App Flow
1. **`initState()`**: When the screen loads, the app connects to the WebSocket server using `socket_io_client`.
2. **`subscribe_to_bus`**: The app sends the assigned `busId` to the server so it only receives updates for that specific bus.
3. **`socket.on('live_location')`**: Every time the server broadcasts a new location (every ~3 seconds), this callback fires.
4. **`setState()`**: We update the `busLocation` variable and re-draw the Map Marker at the new Latitude and Longitude.
5. **`animateCamera()`**: We smoothly move the map view so the bus stays in the center of the screen.

---

# Congratulations!
You now have the complete architectural blueprint and foundational code structure for the **Campus Bus Tracking & Automated Attendance System**. 

### Next Steps
If you're ready to start building, you would typically:
1. Initialize a Node.js project (`npm init`).
2. Install Prisma and Express.
3. Create a Flutter project (`flutter create my_bus_app`).

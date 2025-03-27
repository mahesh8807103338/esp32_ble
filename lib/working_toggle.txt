import 'package:flutter/material.dart';
import 'package:flutter_blue_plus/flutter_blue_plus.dart';
import 'package:skm_notification_listener/skm_notification_listener.dart';
import 'package:permission_handler/permission_handler.dart';
import 'dart:developer';

void main() {
  runApp(NotificationApp());
}

class NotificationApp extends StatefulWidget {
  @override
  _NotificationAppState createState() => _NotificationAppState();
}

class _NotificationAppState extends State<NotificationApp> {
  String notificationText = "Waiting for notifications...";
  bool isConnected = false;
  BluetoothDevice? selectedDevice;
  BluetoothCharacteristic? writeCharacteristic;
  List<BluetoothDevice> availableDevices = [];
  bool isScanning = false;

  final String esp32ServiceUUID = "12345678-1234-5678-1234-56789abcdef0";
  final String esp32CharacteristicUUID = "87654321-4321-8765-4321-abcdef012345";

  /// ✅ **List of Apps with Toggle States**
  Map<String, bool> appToggles = {
    "com.whatsapp": true,
    "com.facebook.katana": false, // Facebook
    "com.instagram.android": false,
    "com.google.android.apps.messaging": true, // ✅ Google Messages (Fixed)
    "com.google.android.gm": true, // Gmail
  };

  @override
  void initState() {
    super.initState();
    _requestPermissions();
    _startBluetoothScan();
    _startListeningNotifications();
  }

  /// ✅ **Request Necessary Permissions**
  Future<void> _requestPermissions() async {
    await Permission.bluetooth.request();
    await Permission.bluetoothScan.request();
    await Permission.bluetoothConnect.request();
    await Permission.location.request();
    await Permission.notification.request();
  }

  /// ✅ **Start Scanning for ESP32 Devices**
  void _startBluetoothScan() async {
    setState(() => isScanning = true);
    availableDevices.clear();

    FlutterBluePlus.startScan(timeout: Duration(seconds: 5));
    FlutterBluePlus.scanResults.listen((results) {
      for (ScanResult result in results) {
        if (!availableDevices.any((device) => device.id == result.device.id)) {
          setState(() {
            availableDevices.add(result.device);
          });
        }
      }
    });

    await Future.delayed(Duration(seconds: 5));
    FlutterBluePlus.stopScan();
    setState(() => isScanning = false);
  }

  /// ✅ **Listen for Notifications & Send to ESP32**
  void _startListeningNotifications() {
    SkmNotificationListener.notificationsStream.listen((event) {
      print("📩 [DEBUG] Incoming Notification");
      print("📦 Package Name Received: ${event.packageName}"); // Added for Debugging
      print("📝 [DEBUG] Title: ${event.title}");
      print("📜 [DEBUG] Content: ${event.content}");

      // ✅ **Check if the app is in appToggles**
      if (!appToggles.containsKey(event.packageName)) {
        print("⚠️ [DEBUG] ${event.packageName} is NOT in appToggles map! Skipping...");
        return;
      }

      // ✅ **Check if notifications are enabled for this app**
      bool isEnabled = appToggles[event.packageName] ?? false;
      print("🔄 [DEBUG] ${event.packageName} Notifications: ${isEnabled ? 'ENABLED' : 'DISABLED'}");

      // ✅ **Stop processing if the app is disabled**
      if (!isEnabled) {
        print("🔕 [DEBUG] Blocking notification from: ${event.packageName}");
        return;
      }

      // ✅ **Update UI**
      setState(() {
        notificationText = "${event.title}: ${event.content}";
      });

      // ✅ **Check Bluetooth connection before sending**
      if (isConnected && writeCharacteristic != null) {
        print("📡 [DEBUG] Sending notification to ESP32...");
        _sendDataToESP32(notificationText);
      } else {
        print("⚠️ [DEBUG] Not connected to ESP32. Cannot send data.");
      }
    });
  }

  /// ✅ **Connect to ESP32**
  Future<void> _connectToDevice(BluetoothDevice device) async {
    try {
      log("🔄 Connecting to ${device.name}...");
      await device.connect(autoConnect: false, timeout: Duration(seconds: 10));

      setState(() {
        selectedDevice = device;
        isConnected = true;
      });

      selectedDevice!.connectionState.listen((state) {
        log("🔄 Connection State: $state");
        if (state == BluetoothConnectionState.disconnected) {
          setState(() {
            isConnected = false;
            selectedDevice = null;
            writeCharacteristic = null;
          });
          log("❌ Disconnected from ESP32");
        }
      });

      List<BluetoothService> services = await device.discoverServices();
      for (var service in services) {
        if (service.uuid.toString() == esp32ServiceUUID) {
          for (var characteristic in service.characteristics) {
            if (characteristic.uuid.toString() == esp32CharacteristicUUID &&
                characteristic.properties.write) {
              setState(() {
                writeCharacteristic = characteristic;
              });
              log("✅ Connected & Found Characteristic");
              return;
            }
          }
        }
      }

      log("❌ No writable characteristic found!");
      setState(() => isConnected = false);
    } catch (e) {
      log("❌ Connection Failed: $e");
      setState(() => isConnected = false);
    }
  }

  /// ✅ **Send Data to ESP32**
  Future<void> _sendDataToESP32(String data) async {
    if (writeCharacteristic != null) {
      try {
        List<int> bytes = data.codeUnits;
        await writeCharacteristic!.write(bytes);
        log("✅ Sent to ESP32: $data");
      } catch (e) {
        log("❌ Error Sending Data: $e");
      }
    }
  }

  /// ✅ **Toggle App Notification State**
  void _toggleAppNotification(String appName, bool value) {
    setState(() {
      appToggles = Map.from(appToggles); // Ensure state updates
      appToggles[appName] = value;
    });
    log("🔄 ${appName} Notifications: ${value ? 'Enabled' : 'Disabled'}");
  }

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      home: Scaffold(
        appBar: AppBar(title: Text(isConnected ? "✅ Connected" : "🔴 Not Connected")),
        body: Column(
          mainAxisAlignment: MainAxisAlignment.start,
          children: [
            Padding(
              padding: EdgeInsets.all(16),
              child: Text(notificationText, textAlign: TextAlign.center),
            ),
            SizedBox(height: 10),

            /// 🔹 **App Notification Toggle List**
            Expanded(
              child: ListView(
                children: appToggles.keys.map((appName) {
                  return SwitchListTile(
                    title: Text(appName),
                    value: appToggles[appName] ?? false,
                    onChanged: (value) => _toggleAppNotification(appName, value),
                  );
                }).toList(),
              ),
            ),

            /// 🔹 **Bluetooth Device Selection**
            DropdownButton<BluetoothDevice>(
              hint: Text("Select Device"),
              value: selectedDevice,
              items: availableDevices.map((device) {
                return DropdownMenuItem(
                  value: device,
                  child: Text(device.name.isNotEmpty ? device.name : "Unknown Device"),
                );
              }).toList(),
              onChanged: (device) {
                if (device != null) {
                  _connectToDevice(device);
                }
              },
            ),
            SizedBox(height: 20),

            /// 🔹 **Bluetooth Scan Button**
            ElevatedButton(
              onPressed: isScanning ? null : _startBluetoothScan,
              child: Text(isScanning ? "Scanning..." : "Refresh Devices"),
            ),
          ],
        ),
      ),
    );
  }
}

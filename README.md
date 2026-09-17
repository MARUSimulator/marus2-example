# MARUS 2.0 Example Project

The **MARUS 2.0 Example Project** (`marus2-example`) is a reference implementation and test environment for the **MArine Robotics Unity Simulator (MARUS)** 2.0. It demonstrates autonomous surface vessel (ASV) and autonomous underwater vehicle (AUV) simulation with realistic buoyancy hydrodynamics, sensor streaming (LiDAR, IMU, GNSS, DVL), and actuation control interfaced with ROS 2 via high-throughput gRPC.

---

## Architecture & Modules

The simulation is built modularly using Unity Package Manager (UPM) packages located under `Assets/marus2-modules`:

- **`marus2-core`**: Core networking (`RosConnection`, `SensorStreamer`), vessel physics, hydrodynamics, thruster models, vehicle controllers, and spatial transformations.
- **`marus2-sensors`**: High-performance sensor simulation models (LiDAR raycasting jobs, IMU, GNSS/GPS, Sonar, Cameras) leveraging Unity's C# Job System and Burst compiler.
- **`marus2-sensors-grpc`**: High-performance gRPC serialization and streaming bridges (e.g. `LidarGrpc`, `ImuGrpc`) that convert Unity sensor data to Protobuf / ROS 2 messages.
- **`marus2-proto`**: Shared protobuf schemas and generated C# bindings matching the ROS 2 adapter message definitions.

---

## gRPC Networking & Dependencies

Communication between Unity and the ROS 2 environment is conducted using gRPC over HTTP/2 cleartext (`h2c`).

### Why `YetAnotherHttpHandler`?
In modern Unity (Mono runtime on Linux and Windows):
- The default `UnityHttpMessageHandler` routes HTTP/2 streaming through native C++ `UploadHandlerStream` (curl). It imposes a hard internal flow-control window (~64 KB – 128 KB). For large payloads—such as full 16-channel LiDAR point clouds (~3.9 MB / 160k+ points per scan) or uncompressed camera feeds—`UnityHttpMessageHandler` stalls or throws `NullReferenceException`.
- Standard .NET `SocketsHttpHandler` lacks support for unencrypted HTTP/2 (`h2c` prior knowledge) under Unity's Mono Linux runtime.

To achieve maximum throughput and sub-10ms latency for streaming sensor payloads of arbitrary size, this project uses **[YetAnotherHttpHandler](https://github.com/Cysharp/YetAnotherHttpHandler)**, an HTTP/2 handler powered by Rust's `hyper` library.

### Dependency Management via OpenUPM
Dependencies are configured seamlessly in `Packages/manifest.json` using an OpenUPM scoped registry:

```json
{
  "dependencies": {
    "com.cysharp.yetanotherhttphandler": "1.11.5",
    "org.nuget.system.io.pipelines": "8.0.0",
    "org.nuget.grpc.net.client": "2.60.0",
    "org.nuget.google.protobuf": "3.25.1"
  },
  "scopedRegistries": [
    {
      "name": "package.openupm.com",
      "url": "https://package.openupm.com",
      "scopes": [
        "org.nuget",
        "com.cysharp"
      ]
    }
  ]
}
```

- **Seamless installation**: No manual DLL copying or binary installations are required. When you open the project in Unity 6, the Unity Package Manager automatically resolves and downloads `YetAnotherHttpHandler` and `System.IO.Pipelines`.
- **Graceful fallback**: `RosConnection` dynamically detects `YetAnotherHttpHandler`. If it is absent, it cleanly falls back to `UnityHttpMessageHandler` without compiler errors.

---

## Getting Started

### 1. Prerequisites
- **Unity 6** (Unity 6000.x or newer) with High Definition Render Pipeline (HDRP).
- **ROS 2** (Humble, Iron, or Rolling) with the [`marus2_ros_adapter`](https://github.com/MARUSimulator/marus2_ros_adapter) package installed in your ROS 2 workspace.

### 2. Start the ROS 2 Adapter Server
In your ROS 2 workspace:

```bash
# Source your ROS 2 workspace
source install/setup.bash

# Run the gRPC ROS 2 server (listens on 0.0.0.0:50051 by default)
ros2 run marus2_ros_adapter server
```

### 3. Open and Run the Example Scene
1. Open this repository in **Unity 6**.
2. Open the scene at `Assets/Scenes/ExampleScene.unity`.
3. Select the `ROS` GameObject in the scene hierarchy and verify the `RosConnection` component:
   - **Server IP**: `127.0.0.1` (or the IP address of your ROS 2 machine/container)
   - **Server Port**: `50051`
4. Click **Play** in the Unity Editor.
5. `RosConnection` will connect to the ROS 2 server, initialize the vehicle and time synchronization, and begin streaming sensor data.

### 4. Verify in ROS 2
Open a terminal on your ROS 2 machine to inspect incoming sensor topics:

```bash
# List active ROS topics
ros2 topic list

# Echo incoming LiDAR point clouds
ros2 topic hz /marus_boat/lidar

# Visualize the vehicle, point cloud, and transforms
rviz2
```

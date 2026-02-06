# Tidybot Frontend

Control the Tidybot robot (Franka Panda arm + mobile base + Robotiq gripper) over the network via a REST/WebSocket API.

**API Server:** `http://<ROBOT_IP>:8080`

Replace `<ROBOT_IP>` with the robot's network address (e.g., `172.16.0.10` or the hostname of the mini PC running the robot).

---

## Quick Start

### 1. Check connectivity

```bash
curl http://<ROBOT_IP>:8080/health
```

```json
{
  "status": "ok",
  "backends": {
    "base": true,
    "franka": true,
    "gripper": true,
    "cameras": false
  }
}
```

If backends show `false`, the corresponding hardware service isn't running — contact the backend team.

### 2. Acquire a lease

All commands require a **lease** (only one controller at a time):

```bash
curl -X POST http://<ROBOT_IP>:8080/lease/acquire \
  -H "Content-Type: application/json" \
  -d '{"holder": "my-agent"}'
```

```json
{"lease_id": "abc123...", "holder": "my-agent", "queue_length": 0}
```

Save the `lease_id` — you'll pass it as `X-Lease-Id` header on all commands.

### 3. Move the robot

**Option A: Submit Python code (recommended)**

```bash
curl -X POST http://<ROBOT_IP>:8080/code/execute \
  -H "X-Lease-Id: abc123..." \
  -H "Content-Type: application/json" \
  -d '{
    "code": "from robot_sdk import arm, sensors\njoints = sensors.get_arm_joints()\nprint(f\"Current: {joints}\")\ntarget = list(joints)\ntarget[4] += 0.1\narm.move_joints(target)\nprint(\"Done!\")"
  }'
```

**Option B: Send a direct command**

```bash
curl -X POST http://<ROBOT_IP>:8080/cmd/arm/move \
  -H "X-Lease-Id: abc123..." \
  -H "Content-Type: application/json" \
  -d '{"mode": "joint_position", "values": [0, -0.785, 0, -2.356, 0, 1.571, 0.785]}'
```

### 4. Release the lease

```bash
curl -X POST http://<ROBOT_IP>:8080/lease/release \
  -H "Content-Type: application/json" \
  -d '{"lease_id": "abc123..."}'
```

---

## API Overview

| Category | Endpoints | Lease Required |
|----------|-----------|----------------|
| **Health & State** | `GET /health`, `GET /state`, `GET /cameras` | No |
| **Lease** | `POST /lease/acquire`, `POST /lease/release`, `GET /lease/status` | No |
| **Code Execution** | `POST /code/execute`, `GET /code/status`, `GET /code/result` | Execute only |
| **Arm Commands** | `POST /cmd/arm/move`, `POST /cmd/arm/stop` | Yes |
| **Base Commands** | `POST /cmd/base/move`, `POST /cmd/base/stop` | Yes |
| **Gripper Commands** | `POST /cmd/gripper` | Yes |
| **Rewind** | `POST /rewind/steps`, `POST /rewind/reset-to-home`, ... | Yes |
| **WebSocket** | `ws://.../ws/state`, `ws://.../ws/cameras` | No |

---

## Reading Robot State

### Current state (polling)

```bash
curl http://<ROBOT_IP>:8080/state
```

```json
{
  "timestamp": 1770176344.82,
  "base": {
    "pose": [0.0, 0.0, 0.0]
  },
  "arm": {
    "q": [0.28, -0.38, 0.18, -1.91, 0.29, 1.92, -0.21],
    "dq": [0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0],
    "ee_pose": [16 values],
    "ee_wrench": [0.0, 0.0, 0.0, 0.0, 0.0, 0.0],
    "mode": 0
  },
  "gripper": {
    "position": 0,
    "is_activated": false,
    "object_detected": false
  },
  "motors_moving": false
}
```

**Extracting end-effector position** from the 4x4 column-major matrix:

```python
ee_pose = state["arm"]["ee_pose"]
x, y, z = ee_pose[12], ee_pose[13], ee_pose[14]
```

**Base pose** is `[x, y, theta]` in meters and radians.

**Arm joints** (`q`) are 7 joint angles in radians.

### Real-time state (WebSocket)

Connect to `ws://<ROBOT_IP>:8080/ws/state` for ~10 Hz state updates. Same JSON format as above.

```python
import websocket, json

def on_message(ws, message):
    state = json.loads(message)
    print(f"EE position: {state['arm']['ee_pose'][12]:.3f}, "
          f"{state['arm']['ee_pose'][13]:.3f}, "
          f"{state['arm']['ee_pose'][14]:.3f}")

ws = websocket.WebSocketApp(f"ws://{ROBOT_IP}:8080/ws/state",
                            on_message=on_message)
ws.run_forever()
```

---

## Code Execution API (Recommended)

Submit Python code that runs on the robot with access to a full SDK. This is the best approach for complex behaviors — loops, conditionals, sensor feedback, error handling all work naturally.

### Workflow

```
1. POST /code/execute  →  code starts running on robot
2. GET  /code/status   →  poll until done (or use timeout)
3. GET  /code/result   →  get stdout, stderr, exit code
```

### Execute

```bash
curl -X POST http://<ROBOT_IP>:8080/code/execute \
  -H "X-Lease-Id: $LEASE_ID" \
  -H "Content-Type: application/json" \
  -d '{
    "code": "from robot_sdk import arm\narm.move_joints([0, -0.785, 0, -2.356, 0, 1.571, 0.785])\nprint(\"Home!\")",
    "timeout": 60.0
  }'
```

Returns immediately:
```json
{"success": true, "execution_id": "abc123", "message": "Code execution started"}
```

### Check status

```bash
curl http://<ROBOT_IP>:8080/code/status
```

```json
{"execution_id": "abc123", "status": "running", "is_running": true}
```

### Get result

```bash
curl http://<ROBOT_IP>:8080/code/result
```

```json
{
  "success": true,
  "result": {
    "status": "completed",
    "execution_id": "abc123",
    "exit_code": 0,
    "stdout": "Home!\n",
    "stderr": "",
    "duration": 3.45,
    "error": ""
  }
}
```

Status values: `completed`, `failed`, `timeout`, `stopped`

### Stop execution

```bash
curl -X POST http://<ROBOT_IP>:8080/code/stop \
  -H "X-Lease-Id: $LEASE_ID"
```

### Validate code (without executing)

```bash
curl -X POST http://<ROBOT_IP>:8080/code/validate \
  -H "Content-Type: application/json" \
  -d '{"code": "from robot_sdk import arm\narm.move_joints([0,0,0,0,0,0,0])"}'
```

### Get SDK documentation

```bash
# JSON format (for programmatic use)
curl http://<ROBOT_IP>:8080/code/sdk

# Markdown format (human-readable)
curl http://<ROBOT_IP>:8080/code/sdk/markdown
```

---

## Robot SDK Reference

These modules are available inside code submitted via `/code/execute`.

### `arm` — Arm Control

All movements use smooth cubic interpolation. Duration is auto-calculated.

```python
from robot_sdk import arm

# Move to joint angles (radians). Home: [0, -0.785, 0, -2.356, 0, 1.571, 0.785]
arm.move_joints([0, -0.785, 0, -2.356, 0, 1.571, 0.785])
arm.move_joints(target, duration=5.0)  # explicit 5-second motion

# Cartesian movement (meters). Unspecified axes keep current value.
arm.move_to_pose(x=0.5, y=0.0, z=0.3)
arm.move_to_pose(x=0.5, y=0.0, z=0.3, roll=3.14, pitch=0, yaw=0)

# Delta movements
arm.move_delta(dx=0.1, dz=0.05, frame="base")
arm.move_delta(dx=0.1, frame="ee")  # end-effector frame

arm.stop()  # emergency stop
```

### `base` — Mobile Base

```python
from robot_sdk import base

base.move_to_pose(x=1.0, y=0.5, theta=0.0)  # absolute global pose
base.move_delta(dx=0.5, frame="local")        # forward 0.5m in robot frame
base.move_delta(dx=0.5, frame="global")       # forward 0.5m in world frame
base.forward(0.5)                              # shorthand: forward 0.5m
base.rotate_degrees(90)                        # rotate 90° counter-clockwise
base.stop()
```

### `gripper` — Robotiq Gripper

```python
from robot_sdk import gripper

gripper.activate()              # required after power-on
gripper.open()
gripper.close()
grasped = gripper.grasp(force=100)  # True if object detected
gripper.move(position=128)      # 0=open, 255=closed
gripper.move(width=0.04)        # width in meters (requires calibration)
state = gripper.get_state()     # {"position": 0, "object_detected": False, ...}
```

### `sensors` — Read-Only State

```python
from robot_sdk import sensors

joints = sensors.get_arm_joints()         # [q0..q6] radians
velocities = sensors.get_arm_velocities() # [dq0..dq6] rad/s
ee_pos = sensors.get_ee_position()        # (x, y, z) tuple
wrench = sensors.get_ee_wrench()          # [fx, fy, fz, tx, ty, tz]
base_pose = sensors.get_base_pose()       # (x, y, theta) tuple
gripper_pos = sensors.get_gripper_position()  # 0-255
is_holding = sensors.is_gripper_holding()     # bool
all_state = sensors.get_all_state()       # everything
```

### `rewind` — Error Recovery

```python
from robot_sdk import rewind

status = rewind.get_status()
rewind.rewind_steps(5)              # go back 5 waypoints
rewind.rewind_percentage(10.0)      # go back 10% of trajectory
rewind.rewind_to_safe()             # return to last in-bounds position
rewind.reset_to_home()              # full rewind to start
rewind.clear_trajectory()           # clear recorded waypoints
```

---

## Direct Command API

For simple, one-shot commands without writing Python code. All require the `X-Lease-Id` header.

### Arm

```bash
# Joint position (7 angles in radians)
curl -X POST http://<ROBOT_IP>:8080/cmd/arm/move \
  -H "X-Lease-Id: $LEASE_ID" -H "Content-Type: application/json" \
  -d '{"mode": "joint_position", "values": [0, -0.785, 0, -2.356, 0, 1.571, 0.785]}'

# Cartesian pose (4x4 matrix, 16 values, column-major)
curl -X POST http://<ROBOT_IP>:8080/cmd/arm/move \
  -H "X-Lease-Id: $LEASE_ID" -H "Content-Type: application/json" \
  -d '{"mode": "cartesian_pose", "values": [1,0,0,0, 0,1,0,0, 0,0,1,0, 0.5,0,0.3,1]}'

# Joint velocity (7 values in rad/s)
curl -X POST http://<ROBOT_IP>:8080/cmd/arm/move \
  -H "X-Lease-Id: $LEASE_ID" -H "Content-Type: application/json" \
  -d '{"mode": "joint_velocity", "values": [0, 0, 0, 0, 0.1, 0, 0]}'

# Cartesian velocity [vx, vy, vz, wx, wy, wz]
curl -X POST http://<ROBOT_IP>:8080/cmd/arm/move \
  -H "X-Lease-Id: $LEASE_ID" -H "Content-Type: application/json" \
  -d '{"mode": "cartesian_velocity", "values": [0.05, 0, 0, 0, 0, 0]}'

# Emergency stop
curl -X POST http://<ROBOT_IP>:8080/cmd/arm/stop \
  -H "X-Lease-Id: $LEASE_ID"
```

### Base

```bash
# Absolute position (meters, radians)
curl -X POST http://<ROBOT_IP>:8080/cmd/base/move \
  -H "X-Lease-Id: $LEASE_ID" -H "Content-Type: application/json" \
  -d '{"x": 1.0, "y": 0.5, "theta": 0.0}'

# Velocity (m/s, rad/s)
curl -X POST http://<ROBOT_IP>:8080/cmd/base/move \
  -H "X-Lease-Id: $LEASE_ID" -H "Content-Type: application/json" \
  -d '{"vx": 0.1, "vy": 0, "wz": 0, "frame": "global"}'

# Stop
curl -X POST http://<ROBOT_IP>:8080/cmd/base/stop \
  -H "X-Lease-Id: $LEASE_ID"
```

### Gripper

```bash
# Activate (required first)
curl -X POST http://<ROBOT_IP>:8080/cmd/gripper \
  -H "X-Lease-Id: $LEASE_ID" -H "Content-Type: application/json" \
  -d '{"action": "activate"}'

# Open / Close
curl -X POST ... -d '{"action": "open"}'
curl -X POST ... -d '{"action": "close"}'

# Grasp with force sensing
curl -X POST ... -d '{"action": "grasp", "force": 100}'

# Move to specific position (0=open, 255=closed)
curl -X POST ... -d '{"action": "move", "position": 128}'

# Move to width in meters
curl -X POST ... -d '{"action": "move", "width": 0.04}'
```

---

## Camera Access

### List cameras

```bash
curl http://<ROBOT_IP>:8080/cameras
```

### Get a single frame

```bash
# From specific camera
curl http://<ROBOT_IP>:8080/cameras/{device_id}/frame -o frame.jpg

# Default camera
curl http://<ROBOT_IP>:8080/state/cameras -o frame.jpg
```

### Stream via WebSocket

Connect to `ws://<ROBOT_IP>:8080/ws/cameras` and subscribe:

```json
{"action": "subscribe", "streams": ["color"], "fps": 15, "quality": 80}
```

Frames arrive as binary messages:

```
[4 bytes: header length (big-endian)] [JSON header] [JPEG image data]
```

Header:
```json
{
  "type": "frame",
  "device_id": "wrist_cam",
  "stream_type": "color",
  "timestamp": 1770176344.82,
  "width": 640,
  "height": 480,
  "format": "jpeg"
}
```

---

## Rewind (Error Recovery)

The robot continuously records its trajectory. You can rewind to undo movements.

```bash
# Check trajectory length
curl http://<ROBOT_IP>:8080/rewind/status

# Rewind 5 waypoints
curl -X POST http://<ROBOT_IP>:8080/rewind/steps \
  -H "X-Lease-Id: $LEASE_ID" -H "Content-Type: application/json" \
  -d '{"steps": 5}'

# Rewind 10% of trajectory
curl -X POST http://<ROBOT_IP>:8080/rewind/percentage \
  -H "X-Lease-Id: $LEASE_ID" -H "Content-Type: application/json" \
  -d '{"percentage": 10.0}'

# Full reset to starting position
curl -X POST http://<ROBOT_IP>:8080/rewind/reset-to-home \
  -H "X-Lease-Id: $LEASE_ID"

# Clear trajectory history
curl -X POST http://<ROBOT_IP>:8080/rewind/trajectory/clear
```

---

## Complete Examples

### Python: Pick and Place

```python
import requests
import time

URL = "http://<ROBOT_IP>:8080"

# Acquire lease
lease = requests.post(f"{URL}/lease/acquire",
                      json={"holder": "pick-place-agent"}).json()
headers = {"X-Lease-Id": lease["lease_id"], "Content-Type": "application/json"}

# Submit pick-and-place code
code = """
from robot_sdk import arm, gripper, sensors
import time

# Setup
gripper.activate()
gripper.open()

# Read current state
ee = sensors.get_ee_position()
print(f"Starting EE position: x={ee[0]:.3f}, y={ee[1]:.3f}, z={ee[2]:.3f}")

# Move above object
arm.move_to_pose(x=0.5, y=0.0, z=0.3)

# Lower to grasp
arm.move_delta(dz=-0.12, frame="base")

# Grasp
grasped = gripper.grasp(force=100)
print(f"Object grasped: {grasped}")

# Lift
arm.move_delta(dz=0.15, frame="base")

# Move to place location
arm.move_to_pose(x=0.5, y=0.3, z=0.3)

# Lower and release
arm.move_delta(dz=-0.12, frame="base")
gripper.open()

# Lift away
arm.move_delta(dz=0.1, frame="base")

# Return home
arm.move_joints([0, -0.785, 0, -2.356, 0, 1.571, 0.785])
print("Done!")
"""

resp = requests.post(f"{URL}/code/execute", headers=headers,
                     json={"code": code, "timeout": 120.0})
exec_id = resp.json()["execution_id"]

# Wait for completion
while True:
    status = requests.get(f"{URL}/code/status").json()
    if not status["is_running"]:
        break
    time.sleep(0.5)

# Print result
result = requests.get(f"{URL}/code/result").json()["result"]
print(f"Status: {result['status']}")
print(f"Duration: {result['duration']:.1f}s")
print(f"Output:\n{result['stdout']}")
if result["stderr"]:
    print(f"Errors:\n{result['stderr']}")

# Release lease
requests.post(f"{URL}/lease/release",
              json={"lease_id": lease["lease_id"]})
```

### JavaScript: Real-Time Dashboard

```javascript
const ROBOT = "http://<ROBOT_IP>:8080";

// State stream
const stateWs = new WebSocket(`ws://<ROBOT_IP>:8080/ws/state`);
stateWs.onmessage = (e) => {
  const state = JSON.parse(e.data);
  document.getElementById("joints").textContent =
    state.arm.q.map(q => q.toFixed(3)).join(", ");
  document.getElementById("base").textContent =
    `x=${state.base.pose[0].toFixed(3)} y=${state.base.pose[1].toFixed(3)} θ=${state.base.pose[2].toFixed(3)}`;
  document.getElementById("gripper").textContent =
    `pos=${state.gripper.position} holding=${state.gripper.object_detected}`;
};

// Camera stream
const camWs = new WebSocket(`ws://<ROBOT_IP>:8080/ws/cameras`);
camWs.binaryType = "arraybuffer";
camWs.onopen = () => {
  camWs.send(JSON.stringify({
    action: "subscribe", streams: ["color"], fps: 15, quality: 80
  }));
};
camWs.onmessage = (e) => {
  if (e.data instanceof ArrayBuffer) {
    const view = new DataView(e.data);
    const headerLen = view.getUint32(0);
    const imageData = e.data.slice(4 + headerLen);
    const blob = new Blob([imageData], { type: "image/jpeg" });
    document.getElementById("camera").src = URL.createObjectURL(blob);
  }
};
```

---

## Lease Management

| Endpoint | Method | Body | Description |
|----------|--------|------|-------------|
| `/lease/acquire` | POST | `{"holder": "name"}` | Get control of the robot |
| `/lease/release` | POST | `{"lease_id": "..."}` | Release control |
| `/lease/extend` | POST | `{"lease_id": "..."}` | Reset lease timeout |
| `/lease/status` | GET | — | Check who holds the lease |

Only one agent can hold the lease at a time. If you can't acquire, check `/lease/status` to see who has it.

---

## Ports

| Port | Service |
|------|---------|
| **8080** | **Agent Server (this API)** |
| 5580 | Camera Server (WebSocket, internal) |

You only need port **8080** — it proxies everything.

---

## Troubleshooting

| Problem | Fix |
|---------|-----|
| `Connection refused` on port 8080 | Agent server not running — contact backend team |
| `{"detail": "backend_unavailable"}` | Hardware service crashed — check `/health` |
| `{"detail": "lease_required"}` | Acquire a lease first: `POST /lease/acquire` |
| `{"detail": "lease_invalid"}` | Your lease expired or was taken — re-acquire |
| Code execution returns `"status": "timeout"` | Increase timeout in request, or check for infinite loops |
| Code execution returns `"status": "failed"` | Check `stderr` in the result for Python errors |
| Arm not moving on direct commands | Direct commands timeout after 100ms — use code execution instead |
| Gripper not responding | Send `{"action": "activate"}` first |
| Robot in error state | Backend team needs to run recovery script |
| Camera endpoint returns empty | Camera server may not be running — check `/health` |

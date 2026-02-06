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

### 2. Get the full API & SDK documentation

The robot serves its own up-to-date documentation:

```bash
# Human-readable markdown
curl http://<ROBOT_IP>:8080/code/sdk/markdown

# JSON format (for programmatic use by agents)
curl http://<ROBOT_IP>:8080/code/sdk
```

This includes all available SDK modules (`arm`, `base`, `gripper`, `sensors`, `rewind`), method signatures, parameters, and examples.

### 3. Acquire a lease

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

### 4. Move the robot

Submit Python code that runs on the robot with access to the full SDK:

```bash
curl -X POST http://<ROBOT_IP>:8080/code/execute \
  -H "X-Lease-Id: abc123..." \
  -H "Content-Type: application/json" \
  -d '{
    "code": "from robot_sdk import arm, sensors\njoints = sensors.get_arm_joints()\nprint(f\"Current: {joints}\")\ntarget = list(joints)\ntarget[4] += 0.1\narm.move_joints(target)\nprint(\"Done!\")",
    "timeout": 60.0
  }'
```

### 5. Check result

```bash
# Poll until done
curl http://<ROBOT_IP>:8080/code/status

# Get output
curl http://<ROBOT_IP>:8080/code/result
```

### 6. Release the lease

```bash
curl -X POST http://<ROBOT_IP>:8080/lease/release \
  -H "Content-Type: application/json" \
  -d '{"lease_id": "abc123..."}'
```

---

## How It Works

```
Your Agent / Frontend
        │
        ├── GET  /health          — check connectivity (no lease needed)
        ├── GET  /state           — read robot state (no lease needed)
        ├── GET  /code/sdk        — read API docs (no lease needed)
        │
        ├── POST /lease/acquire   — get control
        ├── POST /code/execute    — submit Python code (runs on robot)
        ├── GET  /code/status     — poll execution status
        ├── GET  /code/result     — get stdout/stderr/exit code
        └── POST /lease/release   — release control
```

Code submitted via `/code/execute` runs in an isolated subprocess on the robot with access to `arm`, `base`, `gripper`, `sensors`, and `rewind` modules. Use `print()` for output — it's captured in the result.

---

## Real-Time Streaming

### Robot state (~10 Hz)

Connect to `ws://<ROBOT_IP>:8080/ws/state` for real-time state updates:

```python
import websocket, json

def on_message(ws, message):
    state = json.loads(message)
    print(f"Arm joints: {state['arm']['q']}")
    print(f"Base pose: {state['base']['pose']}")

ws = websocket.WebSocketApp(f"ws://<ROBOT_IP>:8080/ws/state",
                            on_message=on_message)
ws.run_forever()
```

### Camera frames

Connect to `ws://<ROBOT_IP>:8080/ws/cameras` and subscribe:

```json
{"action": "subscribe", "streams": ["color"], "fps": 15, "quality": 80}
```

Or get a single frame:

```bash
curl http://<ROBOT_IP>:8080/cameras/{device_id}/frame -o frame.jpg
```

---

## Complete Example: Pick and Place

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

gripper.activate()
gripper.open()

ee = sensors.get_ee_position()
print(f"Starting at: x={ee[0]:.3f}, y={ee[1]:.3f}, z={ee[2]:.3f}")

# Pick
arm.move_to_pose(x=0.5, y=0.0, z=0.3)
arm.move_delta(dz=-0.12, frame="base")
grasped = gripper.grasp(force=100)
print(f"Grasped: {grasped}")
arm.move_delta(dz=0.15, frame="base")

# Place
arm.move_to_pose(x=0.5, y=0.3, z=0.3)
arm.move_delta(dz=-0.12, frame="base")
gripper.open()
arm.move_delta(dz=0.1, frame="base")

# Home
arm.move_joints([0, -0.785, 0, -2.356, 0, 1.571, 0.785])
print("Done!")
"""

resp = requests.post(f"{URL}/code/execute", headers=headers,
                     json={"code": code, "timeout": 120.0})

# Wait for completion
while True:
    status = requests.get(f"{URL}/code/status").json()
    if not status["is_running"]:
        break
    time.sleep(0.5)

# Print result
result = requests.get(f"{URL}/code/result").json()["result"]
print(f"Status: {result['status']}, Duration: {result['duration']:.1f}s")
print(result["stdout"])

# Release lease
requests.post(f"{URL}/lease/release", json={"lease_id": lease["lease_id"]})
```

---

## Troubleshooting

| Problem | Fix |
|---------|-----|
| `Connection refused` on port 8080 | Agent server not running — contact backend team |
| `{"detail": "backend_unavailable"}` | Hardware service crashed — check `GET /health` |
| `{"detail": "lease_required"}` | Acquire a lease first |
| `{"detail": "lease_invalid"}` | Lease expired or taken — re-acquire |
| Code returns `"status": "timeout"` | Increase timeout, or check for infinite loops |
| Code returns `"status": "failed"` | Check `stderr` in the result for Python errors |
| Gripper not responding | Send `{"action": "activate"}` via code first |
| Robot in error state | Backend team needs to run recovery script |

---

**Full API & SDK docs are always available at:** `curl http://<ROBOT_IP>:8080/code/sdk/markdown`

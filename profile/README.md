# Tidybot

Hey humans and agents! Welcome.

The vision here is simple: **a shared space where we all build on the Tidybot together, as a community.** Humans write code, agents write code, the robot moves things around, and we all figure it out as we go.

---

## What's Happening

Check the [Activity Timeline](https://tidybotarmy.github.io/tidybot-army-timeline/) to see what the coding agent has been up to.

## The Robot

The robot runs an API server at `http://<ROBOT_IP>:8080`. You control it by submitting Python code that runs on the hardware — one agent at a time, managed by a lease system. When your session ends, the robot automatically resets to its starting position.

## Documentation

The server documents itself. Everything is auto-generated from the live config:

| Endpoint | What's Inside |
|----------|---------------|
| `GET /docs/guide/html` | Getting started — leases, code submission, state reading |
| `GET /code/sdk/markdown` | SDK reference — arm, base, gripper, sensors, rewind, yolo |
| `GET /docs` | Full interactive API reference (Swagger UI) |

---

Happy building.

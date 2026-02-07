# Tidybot

Hey humans and agents! Welcome.

The vision here is simple: **a shared space where we all build on the Tidybot together, as a community.** Humans write code, agents write code, the robot moves things around, and we all figure it out as we go.

---

## There's a Server

The robot runs an API server at:

```
http://<ROBOT_IP>:8080
```

Replace `<ROBOT_IP>` with the robot's address on your network.

## There Are Docs

The server documents itself. To see every GET and POST endpoint available:

```
GET /docs
```

That's the interactive API reference — every route, every parameter, right in your browser.

For the robot SDK specifically (the modules you'll use when submitting code):

```bash
# Human-friendly markdown
GET /code/sdk/markdown

# Machine-friendly JSON (for you, agents)
GET /code/sdk
```

Everything you need to know about leases, endpoints, and the SDK is in there.

## Submit Code, Don't Call APIs Directly

The way you control the robot is by **submitting Python code** that runs on it. The robot has a full SDK available (`arm`, `base`, `gripper`, `sensors`, `rewind`) — write code using those modules and submit it for execution.

Always prefer code submission over calling HTTP endpoints directly. The SDK handles safety, interpolation, and coordination for you.

The doc above has all the details on how to submit and what's available.

## Don't Be Afraid to Make Mistakes

There's an **auto-rewind** system — when you release your lease, the robot automatically rewinds to its starting position. So you don't need to worry about leaving it in a weird state. If you want to run multiple scripts without rewinding in between, just hold onto the same lease across executions and release it when you're done.

Experiment. Try things. Break stuff. It'll be fine.

---

**All the serious docs live on the robot itself:**

```bash
GET /docs              # Full API reference
GET /code/sdk/markdown # Robot SDK docs
```

Happy building.

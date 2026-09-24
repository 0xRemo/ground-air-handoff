# ground-air-handoff

A tripod camera detects an RC car and cues a Tello, which flies downrange,
finds the car with its own camera, and follows it. Demo project, public park.

## Environment
- macOS, Apple Silicon. PyTorch device is `mps`, never `cuda`.
- Always invoke Python as `.venv/bin/python`. Do not rely on an activated venv.
- Never install a package without asking me first.

## Hard constraints
- No ROS, PX4, MAVSDK, GPS, homography, ArUco, Docker, web frameworks, databases.
- No abstraction layers, plugin systems, or base classes with one subclass.
- Every module under ~300 lines. Split by responsibility, not by cleverness.
- Never command a real Tello from a test, or from any script not named `fly_*`.
  FakeTello is the default everywhere.

## Architecture
Five local processes on a ZeroMQ PUB/SUB bus.

| Node | Owns |
|---|---|
| `nodes/forwarder.py` | XSUB/XPUB proxy. Starts first. The only thing that binds. |
| `nodes/ground_cam.py` | Tripod camera: detect, track, publish bearing + range cue |
| `nodes/coordinator.py` | Sole owner of mission state. Debounce, transitions, commands. |
| `nodes/drone_node.py` | Tello: flight, video, aerial detection, follow controller |
| `nodes/viewer.py` | Two-pane display. Read-only. |

Every node connects; only the forwarder binds. Any node must tolerate starting
in any order and surviving a restart of any other node.

## Bus conventions
- Multipart: `[topic, json]`, or `[topic, json, jpeg]` for video.
- ZMQ filters on a raw byte prefix, so every topic ends with `|`.
- No retained messages. The coordinator republishes `mission.state|` at 1 Hz.
- `zmq.CONFLATE` on detection and frame subscribers. Never on command or state.

## Verification protocol
Every node runs headless from recorded input. No interactive-only paths.
- Every node takes `--source`: camera index, video file, or image directory.
- Every node writes annotated frames to `out/<node>/` and JSONL to `logs/`.
- I judge correctness by opening those frames. Draw boxes, track IDs,
  confidence, and the current state, large enough to read.
- pytest covers pure logic only: state machine, geometry, message round-trip.

## Conventions
- All tunables in `config.yaml`. No magic numbers in code.
- `shared/messages.py` is the single source of truth for the wire format.
- Timestamps are `time.time()` floats. Angles in degrees, distances in meters.
- Bearing is relative to the camera's optical axis: negative left, positive right.
- Use the `logging` module, not print.

## Workflow
- Plan mode before anything non-trivial.
- One module per session. Don't touch files outside scope.
- Run the tests before telling me something works.

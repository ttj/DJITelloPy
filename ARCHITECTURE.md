# Architecture

Source map checked at `8bf5f13bccef18af26bf9c7397b47fba5e6acbdd` on `master`.
This TTJ fork of DJITelloPy wraps the official Tello SDK text protocol. Upstream authors and package metadata remain attributed in the README/setup file. The lab-specific change permits a client to share a machine with the SDK simulator when local UDP `8889` is occupied, and normalizes hostnames for reply lookup. This is a transport/client library, not a planner, flight guard or qualification of a particular aircraft.

## Read first

- [README.md](README.md) explains upstream installation, API usage and physical-device examples; [README_CN.md](README_CN.md) is the Chinese guide.
- [setup.py](setup.py) declares package `djitellopy` version `2.5.0`, Python 3.6+ and NumPy/OpenCV/PyAV/Pillow dependencies.
- [djitellopy/__init__.py](djitellopy/__init__.py) exports `Tello`, `TelloException`, `BackgroundFrameRead` and `TelloSwarm`.
- The lab's [dependency guide](https://github.com/ttj/vu-autonomy/blob/main/docs/DEPENDENCIES.md) and [runtime requirements](https://github.com/ttj/vu-autonomy/blob/main/requirements-runtime.txt) identify this TTJ fork for the same-machine simulator case. Installing stock PyPI DJITelloPy does not select this fork's patch.

## Module map

| Area | Responsibility and useful seam |
| --- | --- |
| [djitellopy/tello.py](djitellopy/tello.py) | `Tello` owns command/state transport, SDK command wrappers, telemetry accessors and video configuration. `parse_state()` converts semicolon-separated state fields. `BackgroundFrameRead` owns PyAV decoding and latest-frame/queue access. |
| [djitellopy/swarm.py](djitellopy/swarm.py) | `TelloSwarm.fromIps()` / `fromFile()` construct multiple clients. `sequential()` calls a function per client; `parallel()` coordinates worker queues/barriers; `sync()` is an explicit barrier inside a parallel operation. |
| [djitellopy/enforce_types.py](djitellopy/enforce_types.py) | Runtime argument type checks applied by the library decorators. These checks do not validate physical command safety or aircraft capability. |
| [examples/](examples) | Existing live control, mission-pad, video, picture and swarm demonstrations. They exercise external devices/GUI state rather than an offline regression suite. |
| [docs/](docs), [mkdocs.yml](mkdocs.yml), [.readthedocs.yaml](.readthedocs.yaml) | API documentation built through MkDocs/mkdocstrings and upstream Read the Docs configuration. The source config is not an exact-head hosted build receipt for this fork. |

## Command, state and video flow

The first `Tello` constructor initializes a process-wide command socket and background response/state receivers. Clients register in a module-global `drones` dictionary keyed by resolved host address. Command wrappers encode SDK text and send it to the selected host's UDP `8889`. `send_command_with_return()` waits on that host's response list; control/read wrappers interpret responses or raise `TelloException`. `send_rc_control()` clamps four channels and sends a command without waiting for a response.

The state receiver binds UDP `8890`, parses ASCII `key:value` fields through `parse_state()` and replaces the registered host's current state dictionary. Numeric conversions are field-specific; unknown fields remain strings. Telemetry accessors and `get_current_state()` read this asynchronously received state, while query methods issue new SDK commands. A received dictionary is not a fused pose estimate or a freshness/flight-admission decision.

Video is a separate stream. `streamon()` asks the endpoint to stream; `get_frame_read()` starts the cached `BackgroundFrameRead`, whose constructor opens the configured UDP video address through PyAV. The worker decodes frames to arrays; callers select latest-frame or bounded-queue access. A command acknowledgement does not prove video frames or fresh state are arriving.

## TTJ fork boundary

`Tello.__init__()` first resolves `host` to its numeric IPv4 address so responses from `127.0.0.1` match a client created with `localhost`. The shared command socket first tries local `8889`. Only `EADDRINUSE` causes a fallback bind to an ephemeral local port; other bind errors propagate. The remote command destination remains `8889`.

This allows the lab's existing [vu-autonomy/surrogate/tello/server.py](https://github.com/ttj/vu-autonomy/blob/main/surrogate/tello/server.py) to own the server port on the same machine while the client receives replies on its source port. It does not add a simulator, alter the SDK protocol or make the state/video ports automatically conflict-free. Reuse that canonical server and the existing lab controller instead of embedding another server or control policy here.

## Setup and entrypoints

For fork development, use an isolated environment and install this checkout with the README's editable-install workflow. Verify the installed module path and source revision before relying on the fork behavior. The upstream `pip install djitellopy` command selects a package release, not necessarily this TTJ commit. There is no installed console-script entrypoint.

Read `Tello` API docstrings after dependencies are available; importing the package and constructing `Tello` are different operations. Construction performs hostname resolution, binds sockets and starts shared receiver threads. `connect()` sends the SDK `command` request and, by default, waits for state. The README's simple example subsequently takes off and moves the vehicle; it is not an installation smoke test.

For API docs, the existing MkDocs/Read the Docs configuration names the documentation dependencies and source pages. No GitHub Actions workflow or checked-in automated test suite exists at this source snapshot. This guide does not install dependencies, bind ports, change network settings or run examples.

## Traps and change boundaries

- Multiple `Tello` instances share process-wide receivers and a host-keyed registry; they are not independent transport sessions. Swarm support coordinates calls, not collision avoidance or fleet qualification.
- The command-port fallback handles one specific conflict. State receiver port `8890` and video reception have separate ownership; do not infer that all same-machine networking conflicts are solved.
- SDK move/query units and body-axis conventions belong to each method's docstring. Four RC channels are command values, not a world-frame trajectory or measured velocity guarantee.
- `end()` may land an instance marked flying, stop its stream/frame reader and remove its registry entry; destructor cleanup calls it too. It does not provide a general reset of all process-wide receiver state.
- The picture example saves a decoded video frame. The lab's optional raw TelloPy still-image/file-transfer path is a different protocol and separate session, documented in [vu-autonomy/scripts/tello_stills.py](https://github.com/ttj/vu-autonomy/blob/main/scripts/tello_stills.py).

Index searched: lab CODE_MAP/CODE_INDEX, external-project/dependency/runtime requirements and safe sibling references; target README/setup/client/swarm/type checks/examples/docs configuration; current TTJ origin and all-state architecture PR/issue search.

Reusing: `Tello`, `BackgroundFrameRead`, `TelloSwarm`, the fork's existing hostname/bind fallback and canonical `vu-autonomy` SDK server/controller boundaries. New because: this safe TTJ fork lacked a root architecture onboarding guide; no executable capability or existing document is changed.

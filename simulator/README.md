# simulator-rs

## Meet Simulator

This simulator is designed to mimic configuration and behavior of real hardware and API that is being used by the smart locker solutions like CellVault™.
The simulator provides web-based UI to let developers imitate all kinds of interactions between user and hardware interfaces (keypad, push buttons, LCD screen, lockers themselves, etc.).

You're able to create and debug your custom solution using nothing but your computer and then seamlessly move it to a real vault device with minimal or no changes at all.
That is possible because the simulator exposes the same [gRPC](https://grpc.io/) API that is available to you on a real device.

When the simulator is started then an initial setup is done. Initial state for every locked is unlocked and open.
Simulator itself does not provide any business logic exactly like a real hardware. So you will only be able
to close lockers' doors using UI and open them back (unless your configuration uses "Chinese" locks which get locked automatically once door is closed).
All those details are documented below.

In order to enable some useful behavior you need to develop *a business logic app* or use one of [the examples](https://) to have something to start with.

## Quick Start

To run the simulator you need [Docker](https://www.docker.com/) installed on your computer.

First pull the image from public registry:
```shell
docker pull public.ecr.aws/h2j0d2q1/simulator-rs:latest
```

For ease of use let's give it a short name:
```shell
docker tag public.ecr.aws/h2j0d2q1/simulator-rs:latest simulator-rs
```

All preparations are done, we can run the simulator right away:
```shell
docker run -p 5000:5000 -p 7777:7777 -p 4200:4200 simulator-rs 5-6-6-3 true true <username> <password>
```

It will take some time to start, and then you can simply navigate to http://localhost:4200 in your web browser to open the UI.

## Configuration

The command we just ran needs some explanation.

The part `-p 5000:5000 -p 7777:7777 -p 4200:4200` is responsible for port mapping (exposure) configuration.
Once created the docker container will expose ports 5000 (WebSocket Server for communication between UI and the simulator), 7777 (gRPC Server) and 4200 (Web UI).

For more information about port mapping, see [docker documentation](https://docs.docker.com/config/containers/container-networking/).

The part `5-6-6-3 true true` is a configuration of hardware to simulate which stands for `<dimensions> <use_cv_locks> <use_multistate_slave>`:
- `dimensions` property defines how many columns (and how many lockers in each column) a simulated vault should have (e.g. `3-3` means two columns having 3 lockers each);
- `use_cv_locks` is a boolean property which defines a type of locks to be simulated (`true` — CV locks that can be locked or unlocked via API, `false` — "Chinese" locks that can be unlocked via API and lock themselves automatically once a locker door is closed);
- `use_multistate_slave` is a boolean property which enables (if set to `true`) third state for a locker (locked/unlocked + ready_to_open which means that locking mechanism is engaged but can be disengaged by pressing a "lock" button which appears on UI) and also an LED indication (green = unlocked, orange = ready_to_open, red = locked).


## Advanced
The simulator accepts the following optional environment variables:

- COMMS=net - if specified, a TCP server will run on port 6677 and will be used instead of the default serial port
- WS_PORT=xxxx - where xxxx is the websocket port on which to listen. Useful if the default port 5000 is already used

Please note that these environment variables must be passed to the application. If used with docker,
some appropriate docker configuration will be required


End

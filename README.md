# Swift

[![A Python Robotics Package](https://raw.githubusercontent.com/petercorke/robotics-toolbox-python/master/.github/svg/py_collection.min.svg)](https://github.com/petercorke/robotics-toolbox-python)
[![QUT Centre for Robotics Open Source](https://github.com/qcr/qcr.github.io/raw/master/misc/badge.svg)](https://qcr.github.io)

[![PyPI version](https://badge.fury.io/py/swift-sim.svg)](https://badge.fury.io/py/swift-sim)
[![PyPI - Python Version](https://img.shields.io/pypi/pyversions/swift-sim)](https://img.shields.io/pypi/pyversions/swift-sim)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

## Fork notice

This repository is a fork of the original [Swift simulator](https://github.com/jhavl/swift). It keeps the original browser-based robotics visualisation workflow, while adding fixes and features useful for sharing and modern Python environments:

- Git dependencies in related projects are intended to use public HTTPS URLs instead of SSH URLs.
- The websocket server startup has been updated for newer `websockets` releases.
- Swift's bundled frontend is served with cache-busting headers so Python and browser-side protocol changes stay in sync.
- A Python-controlled canvas capture API, `env.capture_frame()`, has been added.
- A programmatic video recording API has been added. It captures Swift canvas frames from Python and optionally encodes them to MP4.

Swift is a light-weight browser-based simulator built on top of the [Robotics Toolbox for Python](https://github.com/petercorke/robotics-toolbox-python). This simulator provides robotics-specific functionality for rapid prototyping of algorithms, research, and education. Built using Python and Javascript, Swift is cross-platform (Linux, MacOS, and Windows) while also leveraging the ubiquity and support of these languages.

Through the [Robotics Toolbox for Python](https://github.com/petercorke/robotics-toolbox-python), Swift can visualise over 30 supplied robot models: well-known contemporary robots from Franka-Emika, Kinova, Universal Robotics, Rethink as well as classical robots such as the Puma 560 and the Stanford arm. Swift is under development and will support mobile robots in the future.

Swift provides:

  * visualisation of mesh objects (Collada and STL files) and primitive shapes;
  * robot visualisation and simulation;
  * recording and saving a video of the simulation;
  * source code which can be read for learning and teaching;

## Installing
### Using pip

Swift is designed to be controlled through the [Robotics Toolbox for Python](https://github.com/petercorke/robotics-toolbox-python). By installing the toolbox through PyPI, swift is installed as a dependency

```shell script
pip3 install roboticstoolbox-python
```

Otherwise, Swift can be install by

```shell script
pip3 install swift-sim
```

Available options are:

- `nb` provides the ability for Swift to be embedded within a Jupyter Notebook
- `vision` implements an RTC communication strategy allowing for visual feedback from Swift and allows Swift to be run on Google Colab
- `recording` installs `imageio-ffmpeg` as a fallback video encoder for programmatic recording

Put the options in a comma-separated list like

```shell script
pip3 install swift-sim[optionlist]
```

### From GitHub

To install the latest version from the original GitHub repository

```shell script
git clone https://github.com/jhavl/swift.git
cd swift
pip3 install -e .
```

To install this fork directly from GitHub

```shell script
pip3 install "swift-sim @ git+https://github.com/chaoyuefei/swift.git@master"
```

With optional recording support

```shell script
pip3 install "swift-sim[recording] @ git+https://github.com/chaoyuefei/swift.git@master"
```

Alternatively, install system `ffmpeg` separately, for example on macOS

```shell script
brew install ffmpeg
```

## Programmatic recording

This fork adds a Python-controlled recording path that avoids browser download prompts and browser-native recorder quirks. The simulator captures the Swift WebGL canvas frame-by-frame and encodes the saved frames to MP4 when possible.

```python
from swift import Swift

# Make and launch the simulator
env = Swift()
env.launch(realtime=True)

# Add robots/shapes here...

# Start recording. One frame is captured immediately, then one frame is
# automatically captured after each env.step(...).
env.start_video_recording("demo.mp4", framerate=40)

for _ in range(100):
    # Update robot state here...
    env.step(0.025)

# Encodes demo.mp4 if ffmpeg is available. If no encoder is found, the frame
# directory is kept and returned instead.
output = env.stop_video_recording()
print(f"Recording saved to {output}")
```

### Encoder dependency

Video encoding is optional:

1. If a system `ffmpeg` executable is available, Swift uses it.
2. Otherwise, if `imageio-ffmpeg` is installed via `swift-sim[recording]`, Swift uses that bundled ffmpeg executable.
3. If neither is available, Swift leaves the captured frames on disk and prints an installation hint instead of failing the simulation.

To show full ffmpeg output for debugging:

```python
env.stop_video_recording(verbose=True)
```

To keep only frames and skip MP4 encoding:

```python
frame_dir = env.stop_video_recording(encode=False)
```

You can also capture a single canvas frame manually:

```python
data_url = env.capture_frame(timeout=5.0)
```

## Code Examples

### Robot Plot
We will load a model of the Franka-Emika Panda robot and plot it. We set the joint angles of the robot into the ready joint configuration qr.

```python
import roboticstoolbox as rp

panda = rp.models.Panda()
panda.plot(q=panda.qr)
```
<p align="center">
 <img src="https://github.com/jhavl/swift/blob/master/.github/figures/panda.png">
</p>

### Resolved-Rate Motion Control
We will load a model of the Franka-Emika Panda robot and make it travel towards a goal pose defined by the variable Tep.

```python
import roboticstoolbox as rtb
import spatialmath as sm
import numpy as np
from swift import Swift


# Make and instance of the Swift simulator and open it
env = Swift()
env.launch(realtime=True)

# Make a panda model and set its joint angles to the ready joint configuration
panda = rtb.models.Panda()
panda.q = panda.qr

# Set a desired and effector pose an an offset from the current end-effector pose
Tep = panda.fkine(panda.q) * sm.SE3.Tx(0.2) * sm.SE3.Ty(0.2) * sm.SE3.Tz(0.45)

# Add the robot to the simulator
env.add(panda)

# Simulate the robot while it has not arrived at the goal
arrived = False
while not arrived:

    # Work out the required end-effector velocity to go towards the goal
    v, arrived = rtb.p_servo(panda.fkine(panda.q), Tep, 1)
    
    # Set the Panda's joint velocities
    panda.qd = np.linalg.pinv(panda.jacobe(panda.q)) @ v
    
    # Step the simulator by 50 milliseconds
    env.step(0.05)
```
<p align="center">
 <img src="./.github/figures/panda.gif">
</p>

### Embed within a Jupyter Notebook
To embed within a Jupyter Notebook Cell, use the `browser="notebook"` option when launching the simulator.

```python
# Try this example within a Jupyter Notebook Cell!
import roboticstoolbox as rtb
import spatialmath as sm
import numpy as np
from swift import Swift

# Make and instance of the Swift simulator and open it
env = Swift()
env.launch(realtime=True, browser="notebook")

# Make a panda model and set its joint angles to the ready joint configuration
panda = rtb.models.Panda()
panda.q = panda.qr

# Set a desired and effector pose an an offset from the current end-effector pose
Tep = panda.fkine(panda.q) * sm.SE3.Tx(0.2) * sm.SE3.Ty(0.2) * sm.SE3.Tz(0.45)

# Add the robot to the simulator
env.add(panda)

# Simulate the robot while it has not arrived at the goal
arrived = False
while not arrived:

    # Work out the required end-effector velocity to go towards the goal
    v, arrived = rtb.p_servo(panda.fkine(panda.q), Tep, 1)
    
    # Set the Panda's joint velocities
    panda.qd = np.linalg.pinv(panda.jacobe(panda.q)) @ v
    
    # Step the simulator by 50 milliseconds
    env.step(0.05)
```

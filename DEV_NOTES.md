# Dev notes: context-aware manual simulator

Branch `smart-manual-dashboard`. Runs against the CARLA 0.10.0 (UE5) Windows
release; the scripts live in `PythonAPI/examples/` and are copied next to the
release to run.

```
python manual_control_driver.py --res 1280x720
```

## New features

**Driver's-seat manual control** (`manual_control_driver.py`)
- Starts in a driver's-eye camera; Backspace cycles the Lincoln MKZ, Dodge
  Charger, Nissan Patrol, Ford taxi and Mini Cooper, each with a tuned eye.

**Second-screen dashboard** (`smart_manual_dashboard/`)
- Opens full screen on monitor 2 (Edge/Chrome app window, local server,
  server-sent events). Shows live speed; the other panels still show defaults.
- `--no-dashboard`, `--dashboard-monitor N`, `--dashboard-port P`;
  `python -m smart_manual_dashboard` runs it without CARLA.

**Logitech G25 wheel with force feedback** (`steering_wheel.py`)
- Steering, throttle, brake and clutch; the keyboard keeps working beside it.
  CARLA has no clutch, so the clutch cuts the throttle.
- 15:1 steering ratio (`--steer-ratio`), like a real sedan. 18:1 was tried and
  felt worse.
- Force feedback: aligning torque from the front tyres' slip (goes light when
  understeering), power steering (light when parking, firmer with speed), road
  rumble and body-roll kicks, a jolt on collisions. On autopilot the wheel
  turns with the car. `--ffb-gain` scales it all.
- Needs G HUB running (900 degrees, FFB strength at max).
  `python steering_wheel.py` prints the live inputs for checking the axes.

## Known gaps

- Dashboard only gets speed so far: nearby vehicles, feature prompts and
  driver profile are still to wire up.
- Shifter and wheel buttons are not mapped (automatic gearbox; Q reverses).
- Close the drive window with Esc, not by killing it: destroying a car while
  its sensors are still attached has crashed the server.

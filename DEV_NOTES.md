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
- 10:1 steering ratio (`--steer-ratio`), picked by driving it with the wheel
  at a true 900 degrees; a real sedan's 15:1 felt sluggish. Set 900 degrees in
  G HUB: the ratio assumes it, and a smaller range makes steering twitchy.
- Force feedback: aligning torque from the front tyres' slip (goes light when
  understeering), power steering (light when parking, firmer with speed), road
  rumble and body-roll kicks, a jolt on collisions. On autopilot the wheel
  turns with the car. `--ffb-gain` scales it all.
- H-shifter as an automatic's selector: 1st = P, 3rd = D, 4th = R, middle or
  any other slot = N. CARLA has no park, so P is neutral with the hand brake
  on and the throttle cut. The dashboard's P/R/N/D selector follows it.
- Needs G HUB running (900 degrees, FFB strength at max).
  `python steering_wheel.py` prints the live inputs for checking the axes.

## Known gaps

- Dashboard only gets speed so far: nearby vehicles, feature prompts and
  driver profile are still to wire up.
- Wheel buttons are not mapped yet.
- If the Traffic Manager fails with a bind error, Windows has reserved port
  8000: pass `--tm-port 8010`.
- Close the drive window with Esc, not by killing it: destroying a car while
  its sensors are still attached has crashed the server.

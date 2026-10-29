# Dev notes: context-aware manual simulator

Branch `smart-manual-dashboard`. Runs against the CARLA 0.10.0 (UE5) Windows
release; the scripts live in `PythonAPI/examples/` and are copied next to the
release to run.

```
python manual_control_driver.py
```

## New features

**Driver's-seat manual control** (`manual_control_driver.py`)
- Starts in a driver's-eye camera, full screen (borderless) on the primary
  monitor; `--windowed` gives a window of `--res`. Backspace cycles the
  Lincoln MKZ, Dodge Charger, Nissan Patrol, Ford taxi and Mini Cooper, each
  with a tuned eye.

**Drive modes and regen**
- Eco / Normal (default) / Sport on the paddles: left = down, right = up.
  Eco caps the throttle at 60% with a soft start and makes the force feedback
  40% lighter; Sport sharpens the throttle.
- Regenerative braking Off / Low / Medium / Maximum, set by the mode (Eco
  Maximum, Normal Low, Sport Off) and adjusted with the wheel's top buttons
  (left down, right up). CARLA has no regen, so it is brake on lift-off:
  0.12 / 0.25 / 0.5 (about 0.5 g), easing off below 13 km/h; none in N or P.

**Cruise control** (plain, not adaptive)
- G25 buttons: 15 on/off, 18 +5, 16 -5, 17 resume. On/off engages at the
  current speed (from 20 km/h, in D) and off forgets the set speed. + and -
  step to the next 5 km/h (47 goes to 50 or 45); on standby they set the
  current speed. Resume returns to the last set speed.
- Throttle only, from a PI loop: it never brakes, so it overshoots downhill.
  The throttle pedal overrides it; braking, the clutch, the hand brake,
  leaving D or autopilot cancels it to standby. No regen while it holds.

**Second-screen dashboard** (`smart_manual_dashboard/`)
- Opens full screen on monitor 2 (Edge/Chrome app window, local server,
  server-sent events). Shows live speed, gear, drive mode, regen, throttle
  (applied vs pedal), brake and cruise control; the adaptive-cruise,
  autopilot and tip panels still show defaults.
- `--no-dashboard`, `--dashboard-monitor N`, `--dashboard-port P`;
  `python -m smart_manual_dashboard` runs it without CARLA.

**Manual tips** (`smart_manual_dashboard/tips.py`)
- A tip slides into the bottom panel with a soft two-note chime (Web Audio,
  no sound file; the browser starts with autoplay allowed) and a glow that
  settles. It outlines the dashboard zones it names (`highlight`) with a
  pulsing ring and a TIP tag, and loops a drawing of the movement to make
  (`gesture`).
- One test tip so far: above 100 km/h, "lift your foot off the
  accelerator", outlining the speed and the throttle bar. It stays at least
  4 s and clears below 97 km/h.
- Zones: speed, throttle, brake, gear, cruise, driveMode, regen. Gestures:
  release-throttle only; button gestures (drive modes and so on) come next.

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

- Dashboard: nearby vehicles, adaptive cruise, autopilot, the driver
  profile and real tips (beyond the test tip) are still to wire up. The tip
  panel's Dismiss and Turn tips off buttons are gone until there is input
  for them.
- The d-pad and the other shifter-unit buttons are not mapped yet.
- If the Traffic Manager fails with a bind error, Windows has reserved port
  8000: pass `--tm-port 8010`.
- Close the drive window with Esc, not by killing it: destroying a car while
  its sensors are still attached has crashed the server.

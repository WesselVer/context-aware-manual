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
  A short press of on/off; a long press is ACC, below.
  The throttle pedal overrides it; braking, the clutch, the hand brake,
  leaving D or autopilot cancels it to standby. No regen while it holds.

**Adaptive cruise control (ACC)**
- Long press (0.6 s) of on/off (15); a short press turns either cruise off.
  Gap buttons: 1 gap -, 2 gap +, time gaps 1.0 / 1.5 / 2.0 / 2.5 s, gap 2
  to start; keeps 5 m at a standstill.
- Set speed to start: 30 km/h, or the current speed when faster (so
  switching it on at 80 does not brake down to 30). It no longer follows
  the road's limit (`get_speed_limit()`; always 30 on Town10HD, 100 on the
  figure-8 highway).
- The car ahead: the nearest vehicle on the ego lane's waypoints up to 100 m
  (straightest way through junctions), checked 20 times a second. It brakes
  to keep the gap and stops behind a stopped car; after 3 s stopped it waits
  for resume or the throttle. Engages from 20 km/h, or slower with a car ahead.
- Traffic: `generate_traffic.py --asynch --tm-port 8010` from the release's
  `PythonAPI/examples`. Without `--asynch` it puts the server into
  synchronous mode, and killing it leaves the server there, frozen.

**Autopilot**
- G25 button 0 (or K) toggles it, in D only. The Traffic Manager drives the
  car around the map at random: 30 km/h to start (+5/-5, 10 to 90 km/h), the
  following distance from the ACC gap buttons (5 m + time gap x speed), Eco
  while it drives. The wheel turns with the car through the FFB spring.
- Takeover: the brake pedal past 10%, the shifter out of D, or the wheel held
  40 degrees off the car's steering for 0.25 s without coming back (the
  spring resists until then). The drive mode from before comes back.
- The dashboard's autopilot card shows DRIVING and pulses.

**Assist chimes** (dashboard, Web Audio)
- ACC on: two rising triangle notes; ACC off or to standby: the same
  falling. Autopilot on: four rising sine notes; off: four falling sine
  notes ending on a held low one with its octave below, to be noticed. Plain cruise is silent.

**EV hum** (dashboard, Web Audio)
- A sine motor whine at 55 Hz + 4 Hz per km/h (about 455 Hz at 100) with a
  quiet 3rd harmonic, louder under throttle or braking, over low-passed noise
  for road and wind that opens up with speed. Fades out at a standstill and
  plays from the dashboard PC's speakers, so `--no-dashboard` has no sound.

**Second-screen dashboard** (`smart_manual_dashboard/`)
- Opens full screen on monitor 2 (Edge/Chrome app window, local server,
  server-sent events). Shows live speed, gear, drive mode, regen, throttle
  (applied vs pedal), brake and cruise control; the adaptive-cruise,
  autopilot and tip panels still show defaults.
- `--no-dashboard`, `--dashboard-monitor N`, `--dashboard-port P`;
  `python -m smart_manual_dashboard` runs it without CARLA.

**Context-aware manual** (`smart_manual_dashboard/tips.py`)
- Off when the drive starts. Keyboard B / E / P starts it for Beginner /
  Experienced / Expert; holding R for 1 s stops it. The header shows
  CONTEXT AWARE ON with the level lit and pulsing, the key under each level.
  (Autopilot moved from P to K, image recording to Shift+R, map-layer
  loading to Ctrl+B / Ctrl+Shift+B.)
- Beginner curriculum, in order: cruise on/off (tap ON / OFF), brake to
  cancel it (after 20 s of cruise or switching it off), ACC (hold ON / OFF),
  autopilot (after 20 s of ACC or switching it off). A tip's turn also comes
  after 120 s without its condition. Something the driver already did
  before its tip is skipped. Experienced and Expert have no tips yet.
- Timing: the first tip can come straight away (pressing the level key is
  the settling-in), then 20 s between tips. Only at a good
  moment (stopped 4 s, steady within 4 km/h going straight for 6 s, or at
  most 1 car within 30 m going straight for 4 s) and not a bad one (a
  junction within 4 s of driving / 25 m, 3+ cars within 30 m, 2+ people
  within 15 m, or for 3 s after reversing, turning > 8 deg/s, indicating or
  braking over 50%). Thresholds are constants at the top of tips.py;
  the measuring is `DrivingContext` in the driver script.
- A tip stays up until the driver has done it, whatever happens after it
  appeared, then shows a 4 s "Done" confirmation with its own chime. The
  "Why now" line says which good moment it came at. Each step is printed to the console as `[context hh:mm:ss]`.
- Drawing: control tips light the button on a drawing of the G25 shifter's
  top (tap: presses in with a ripple; hold: a ring fills) with a leader line
  to its label; the brake tip loops a foot pressing the pedal. While a tip
  shows, its panel grows to 420 px with the drawing at 2x on the left and
  the text on the right; the dashboard above compacts (smaller speed, gear
  and card values, the ACC gap drawing left out). The tip also
  outlines the zones it names (`highlight`) with a pulsing ring and TIP tag.
- With the manual on and no tip, the panel shows the progress dots and what
  it waits for ("Not now: junction ahead", "Next tip after ...").

**Progress replay** (`smart_manual_dashboard/make_showcase.py`)
- `python -m smart_manual_dashboard.make_showcase out.html` builds one
  self-contained page: the real dashboard playing a scripted Beginner drive
  (speed, assists, traffic and junctions in `DRIVE`), with the tip timing
  decided by the real `tips.py`, plus chapters and captions
  (`showcase_template.html`). Rebuild it when the dashboard or tips change.

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

- Dashboard: nearby vehicles and the Experienced and Expert tips are
  still to come; blinkers are keyboard-only (Z/X), so indicating only
  counts as a manoeuvre from the keyboard. The tip
  panel's Dismiss and Turn tips off buttons are gone until there is input
  for them.
- The d-pad and the other shifter-unit buttons are not mapped yet.
- If the Traffic Manager fails with a bind error, Windows has reserved port
  8000: pass `--tm-port 8010`.
- Close the drive window with Esc, not by killing it: destroying a car while
  its sensors are still attached has crashed the server.

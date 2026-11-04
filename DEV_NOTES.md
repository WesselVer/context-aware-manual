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
- Eco / Normal (default) / Dynamic on the paddles: left = down, right = up.
  Eco caps the throttle at 60% with a soft start and makes the force feedback
  40% lighter; Dynamic sharpens the throttle.
- Regenerative braking Off / Low / Medium / Maximum, set by the mode (Eco
  Maximum, Normal Low, Dynamic Off) and adjusted with the wheel's top buttons
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
  Gap on the shifter's d-pad (hat 0): up = gap -, down = gap +; time gaps 1.0 / 1.5 / 2.0 / 2.5 s, gap 2
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
  following distance from the ACC gap on the d-pad (5 m + time gap x speed), Eco
  while it drives. The wheel turns with the car through the FFB spring.
- Throttle override: the accelerator past 5% speeds it up by up to 40 km/h
  at full throttle (a higher Traffic Manager speed) while it keeps steering;
  released, it drops back to its own speed. The dashboard shows Driver
  override.
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
  before its tip is skipped.
- Experienced curriculum (knows P/R/N/D and cruise on/off): change the set
  speed with +5/-5 (once cruise or ACC is on), Dynamic mode on the right
  paddle, RES after braking (once cruise is on standby), the autopilot's
  speed with +5/-5 (once it drives). The cruise and autopilot tips are
  strict (they wait for their state, no 120 s fallback); RES and autopilot
  speed are anytime (they can jump ahead when their moment comes), and so
  is Dynamic mode, so it need not wait for cruise to be switched on. Dynamic
  mode and regen tips wait until the driver drives themself (no assist
  holding the speed), since under an assist they would not feel it.
- A tip to switch an assist on is skipped when that assist is already on.
- Expert curriculum (drives with modes and regen): ACC on (hold ON / OFF),
  then in any order as their moments come: ACC speed with +5/-5 and ACC gap
  on the d-pad (both once ACC is on), regen to Maximum for one-pedal driving
  on the wheel's right button (a drive mode setting regen does not count),
  and overriding the autopilot with the accelerator (while it drives).
- Drawings now cover the wheel too: the paddles behind the rim and the regen
  buttons on the spokes; a tip can light more than one button (+5 and -5).
- Timing: the first tip can come straight away (pressing the level key is
  the settling-in), then 20 s between tips. Only at a good
  moment (stopped 4 s, steady within 4 km/h going straight for 6 s, or at
  most 1 car within 30 m going straight for 4 s) and not a bad one (a
  junction within 4 s of driving / 25 m, 3+ cars within 30 m, 2+ people
  within 15 m, or for 3 s after reversing, turning > 8 deg/s, indicating or
  braking over 50%; the autopilot's own braking does not count).
  Thresholds are constants at the top of tips.py; the measuring is
  `DrivingContext` in the driver script.
- A tip stays up until the driver has done it, whatever happens after it
  appeared, then shows a 4 s "Done" confirmation with its own chime. The
  "Why now" line says which good moment it came at. Each step is printed to the console as `[context hh:mm:ss]`.
- Drawing: control tips light the button on a drawing of the G25 shifter's
  top (tap: presses in with a ripple; hold: a ring fills) with a leader line
  to its label; the brake tip loops a foot pressing the pedal. While a tip
  shows, its panel grows to 420 px with the drawing at 2x on the left and
  the text on the right; the dashboard above compacts (smaller speed, gear
  and card values, the ACC gap drawing turned sideways into one row). The tip also
  outlines the zones it names (`highlight`) with a pulsing ring and TIP tag.
- With the manual on and no tip, the panel shows the progress dots and what
  it waits for ("Not now: junction ahead", "Next tip after ..."). When the
  next tip waits for the driver to use a mode first (cruise on, ACC on, the
  autopilot driving), that mode's card gets a dashed outline and a NEXT TIP
  tag (`expect` in tips.py).

**Progress replay** (`smart_manual_dashboard/make_showcase.py`)
- `python -m smart_manual_dashboard.make_showcase out.html` builds one
  self-contained page: the real dashboard playing a scripted Beginner drive
  (speed, assists, traffic and junctions in `DRIVE`), with the tip timing
  decided by the real `tips.py`, plus chapters and captions
  (`showcase_template.html`). Rebuild it when the dashboard or tips change.

**Driving score for user tests** (`smart_manual_dashboard/drive_log.py`, `report.py`)
- The questionnaires (experience, confidence, knowledge, Van der Laan) are
  in MS Forms; this only scores the driving. One participant:
  ```
  python manual_control_driver.py --participant P07 --speed-limit 50
  python -m smart_manual_dashboard.report P07
  ```
  Everything lands in `study_data/P07/` beside the driver script (not in
  git); `report --all` redoes every report, and each run rewrites
  `study_data/summary.csv`, one row per participant, to join with the
  Forms export on the participant id.
- The first B, E or P (the level from the Forms answers) starts the test
  and its log. F starts the free drive: no tips until F again, or at most
  `--free-drive-seconds` (120), then the score shows as a final tip. F
  replaces the stock script's Ackermann-controller toggle.
- Holding Z for 5 s resets for the next participant: the manual off, the
  assists, drive mode and lights back to their defaults, the same car at a
  new spawn point, and the next test logging to a new `session-<time>`
  folder. A tap of Z is still the left blinker. `--record` also runs
  CARLA's recorder for the test.
- The log (10 Hz CSV plus an events CSV) holds speed, the limit, the
  assist, pedals, steering, lateral offset in the lane, the car ahead
  (gap, headway, TTC), and the tips, features used, takeovers, collisions
  and lane crossings. `drive.json` has the metrics of the guided phase and
  the free drive.
- The score is five parts of 0-100 and their mean, each a plain rule
  (`drive_log.score`): safety (collisions, TTC under 1.5 s, headway under
  1 s), lane keeping (SDLP 20 cm = 100, 50 cm = 0; crossings without a
  blinker; SDLP leaves out 3 s either side of a lane crossing), speed
  (share of time within the limit + 5 km/h), smoothness (braking harder
  than 3.5 m/s2 on the brake pedal; maximum regen on lift-off is about
  3.9 m/s2 and does not count), and features (of the features the tips
  showed in the guided phase, the share used again in the free drive: do
  they remember how? Left out of the total when no tip was shown). Each
  phase is scored at the level it was driven at.
- The report shows the score with the rule behind each part, each metric
  of the guided phase and the free drive with what it measures, and the
  tips' timing.

**Logitech G25 wheel with force feedback** (`steering_wheel.py`)
- Steering, throttle, brake and clutch; the keyboard keeps working beside it.
  CARLA has no clutch, so the clutch cuts the throttle.
- 8:1 steering ratio (`--steer-ratio`), picked by driving it with the wheel
  at a true 900 degrees; a real sedan's 15:1 felt sluggish, 10:1 still a
  little slow, and 9:1 a touch slow on the first test drives. Set 900 degrees in
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
- The d-pad's left and right and the red buttons 1-3 are not mapped yet.
- If the Traffic Manager fails with a bind error, Windows has reserved port
  8000: pass `--tm-port 8010`.
- Close the drive window with Esc, not by killing it: destroying a car while
  its sensors are still attached has crashed the server.
- Town10HD has no speed-limit signs, so CARLA reports 30 km/h everywhere:
  give user tests `--speed-limit` or the speed score counts everything
  over 35 km/h as speeding.
- The drive logging has only run against a simulated drive, not CARLA yet:
  check the lateral offset, the car ahead and the sensor events on the
  first real test.

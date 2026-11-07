# LG Air Pointer: phone as a Magic Remote — design

Date: 2026-11-07 · Status: approved layout, pending plan

## Intent

The owner's LG LJ617T (webOS 3.5, 2017) Magic Remote drains AA cells too fast. They want an Android
app that turns the phone itself into the pointer: aim the top of the phone at the TV and the
on-screen cursor follows, like the Magic Remote's motion pointer. Not a touchpad.

Audience, in order: the owner, then family members, then (later) the public via Play Store.

## Goals

- Phone-as-pointer: cursor follows where the **top of the phone** points (rotation: yaw/pitch),
  not translation of the whole phone. Always active while the app is open and on screen; no
  hold-to-move button.
- Feels as quick as the Magic Remote over home Wi-Fi.
- All buttons the owner uses: power (on and off), volume ±, mute, channel ±, back, home, menu,
  inputs, recents (the webOS recently-used-apps row).
- Text entry into TV search/text boxes by phone keyboard and by voice.
- Phone hardware volume keys change TV volume while the app is in the foreground.
- Zero battery cost when the app is not on screen.
- Works for several phones and more than one TV in the house.

## Anti-goals (fails even if it technically works)

- **Laggy** pointer or buttons.
- **Battery zombie**: anything running, listening or holding sensors/sockets/wakelocks when the app
  is not in the foreground.
- **Pointer that needs whole-arm movement**: sliding the phone left/right/up/down instead of
  pivoting its tip.

## Non-goals (v1)

- Play Store release, store listing, privacy policy (later step; don't block it, don't build for it).
- iPhone, tablets, Wear OS, widgets, notification-shade remote, background service.
- Number pad, playback keys, app launcher grid, screen mirroring.
- Any cloud/server component.

## Constraints

- TVs in the house (both must work in v1):
  - **LG LJ617T** (2017, webOS 3.5), on **Ethernet**. SSAP over plain `ws://<tv-ip>:3000`.
  - **LG 55UT7550** (2024, webOS 24), on **Wi-Fi**. Expects secure `wss://<tv-ip>:3001` with the TV's self-signed
    certificate; plain ws is likely refused. The pointer-input socket it hands back is wss too.
  - Connection: try wss:3001 first, fall back to ws:3000. Trust the TV's certificate on first use
    and pin its fingerprint with the stored TV; a changed certificate means re-pair, not silent accept.
- Control via LG's local second-screen protocol (SSAP). First connection shows an "allow" prompt on the TV; the returned client-key
  is stored per TV and reused so the prompt appears only once per phone.
- LJ617T has "Mobile TV On" enabled by the owner and is wired, so Wake-on-LAN should be reliable.
  The 55UT7550 is on **Wi-Fi**; it needs its "Turn on via Wi-Fi" / "Mobile TV On" setting enabled,
  and wake over Wi-Fi must be verified on it. Power-on
  is a Wake-on-LAN magic packet to the TV's MAC address of the interface it is connected by, captured while connected (e.g. from the TV's connection-manager info) and stored with
  the TV.
- Android only. Owner has Android Studio. Sideload/debug install for the owner and family.
- Same Wi-Fi as the TV; no internet dependency for control (voice recognition may use the phone's
  speech service, which can be online).

## Decided behaviour

### Screen layout (owner approved, "middle ground" between a full remote and an all-click screen)

Portrait, top of phone aimed at TV. Top to bottom, everything visible, no drawer:

1. Small row: **Power · Inputs · Menu · Mute**
2. **Click area**, roughly 55% of the screen, under the thumb. Tap = OK/click. Hold = long-press.
   Vertical swipe = scroll (stands in for the Magic Remote wheel).
3. Row: **Back · Home · Recents**
4. Row: **Volume − / +** pill · **Channel ▼ / ▲** pill
5. Row: **🎤 Speak** · **⌨ Type**

Mockup: [mockups/layout-v2.html](mockups/layout-v2.html) (content fragment; styles come from the brainstorm frame).

### Pointer

- Driven by the gyroscope. Cursor motion follows the phone's pointing direction relative to the
  user, so it behaves the same whether the phone is held flat or tilted toward the TV.
- Small-tremor dead zone so the cursor doesn't creep while held still.
- **Sensitivity setting** (calibration knob) — gyros and hands differ; this must be tunable.
- Movement is sent to the TV's pointer-input channel at a steady high rate; fractional movement is
  carried over, not dropped.
- Phone put down flat and still → pointer pauses until picked up (Claude's call).
- Cursor hiding/reappearing: whatever the TV does natively is fine (Claude's call; earlier chat
  mentioned "reappears centred" — only if the TV provides it).

### Buttons

- Power: if connected → turn TV off; if not reachable → send Wake-on-LAN and connect when it
  answers.
- Inputs: opens a list of the TV's inputs using the names set on the TV; pick one to switch.
- Recents: shows the webOS recently-used-apps row. The exact command must be verified on the real
  TV; fallback is Home (the webOS 3.5 launcher bar shows recents).
- Menu, Back, Home, Mute, Volume, Channel: TV remote keys.
- Light haptic tick on click and button presses (Claude's call).

### Text and voice

- Speak and Type work only when a text box is open on the TV. The TV reports when its keyboard is
  open; while it isn't, both buttons are dimmed (Claude's call). The app does not open a search
  screen itself.
- **Type**: phone keyboard; text goes into the TV box, Enter submits.
- **Speak**: Android speech recognition in the phone's language. Recognised words are shown on the
  phone first; one tap sends them (user catches mistakes before they reach the TV).

### Connection and multiple TVs/phones

- Auto-discover TVs on the Wi-Fi (SSDP, LG second-screen service). Manual IP entry as fallback.
- Remember paired TVs; auto-connect to the last used one on open. Switch TV from a simple picker.
- Each phone pairs on its own (one TV prompt per phone).

### Battery and lifecycle

- Sensors, websockets and any screen-on flag exist only while the app is in the foreground.
  Leaving the app or locking the phone releases everything immediately. Nothing in the background.
- Reopening reconnects fast (target: usable within about a second, with the stored client-key).
- While the app is open, the screen stays on; after ~30 s without use it dims (still usable), and it
  restores on movement or touch (Claude's call).

## Success looks like

- Owner leaves the Magic Remote in a drawer.
- Pointing feels as quick as the Magic Remote on their Wi-Fi.
- No noticeable battery drain attributable to the app when it isn't open.
- A family member can install it and pair it without help beyond the TV's allow prompt.

## Claude's calls (owner may override)

- Working name "Air Pointer"; avoid LG trademarks in the name before any public release.
- Kotlin + Jetpack Compose, minimal dependencies (a websocket client such as OkHttp is acceptable).
- Hardware gyroscope required. Phones without a real one get a clear "not supported" message, not a
  laggy rotation-vector fallback (anti-goal: lag).
- Family install: a release-signed APK shared as a file, with a short "install unknown apps" note.
  Updates = new APK, same signing key. No Play/Firebase distribution in v1.
- Power-on is unverified on both TVs. The first spike tests, on each TV: pointer lag, pairing
  (ws on the LJ617T, wss on the 55UT7550), and Wake-on-LAN from fully off. If it fails, stop and ask the owner (fallbacks: IR blaster on phones that
  have one, or off-only).
- Hardware volume keys mapped to TV volume; Home button added; swipe-to-scroll; put-down pause;
  dim-after-30 s; haptics; dim Speak/Type when no TV text box is open.

## Left to the builder

- Gyro-to-cursor maths (frame handling, gain curve, dead zone values, send rate).
- Exact SSAP/pointer-socket command names for each key, verified on the real TV, including
  long-press and recents.
- Minimum Android version, project structure, storage mechanism for paired TVs.
- Settings screen shape (at least: sensitivity, TV picker/forget TV).
- Testing approach. First milestone should be a pointer-only spike on the real TV to measure lag
  before building the rest.

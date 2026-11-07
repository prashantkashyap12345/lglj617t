# Air Pointer Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** An Android app that turns the phone into a gyro "air pointer" remote for two LG webOS TVs (LJ617T webOS 3.5, 55UT7550 webOS 24), with the remote keys, inputs, and keyboard/voice text entry.

**Architecture:** Pure-Kotlin core (pointer maths, SSAP message building/parsing, pointer-socket frames, SSDP parsing, Wake-on-LAN packets, TV store encoding, certificate pinning) is unit-tested on the JVM. A thin Android layer wires the gyroscope + gravity sensors, OkHttp websockets to the TV, and a single Compose screen. Everything network/sensor-related lives only between `onResume` and `onPause`.

**Tech Stack:** Kotlin 2.0.21, AGP 8.7.3, Gradle 8.11.1, Jetpack Compose (BOM 2024.12.01), activity-compose 1.9.3, OkHttp 4.12.0; tests: JUnit 4.13.2, `org.json:json:20240303` (real org.json on the unit-test classpath), OkHttp `mockwebserver` 4.12.0.

**Spec:** `docs/superpowers/specs/2026-11-07-lg-air-pointer-design.md`

## Global Constraints

- Package / applicationId: `com.prashantkashyap.airpointer`. App label: `Air Pointer`. No "LG" in the app name.
- `minSdk 26`, `targetSdk 35`, `compileSdk 35`, JVM target 17. Portrait only (`android:screenOrientation="portrait"`).
- Runtime dependencies limited to AndroidX/Compose and OkHttp. JSON via platform `org.json`.
- Connection order: `wss://<ip>:3001` first, then `ws://<ip>:3000`; remember which one worked per TV and try it first next time.
- TLS: trust-on-first-use; pin SHA-256 of the TV's leaf certificate; a different certificate later = refuse and require re-pair. Never a silent accept.
- Battery: no Service, no WorkManager, no background threads. Sensors, sockets, `FLAG_KEEP_SCREEN_ON`, multicast lock are acquired in `onResume`/on demand and released in `onPause`.
- Gyroscope required: without `PackageManager.FEATURE_SENSOR_GYROSCOPE` show only "This phone has no gyroscope, so Air Pointer can't work on it."
- Cleartext is allowed (LAN ws:3000 and SSDP description HTTP): `android:usesCleartextTraffic="true"`.
- Manifest permissions: `INTERNET`, `ACCESS_NETWORK_STATE`, `ACCESS_WIFI_STATE`, `CHANGE_WIFI_MULTICAST_STATE`. Nothing else.
- Environment: `JAVA_HOME="/Applications/Android Studio.app/Contents/jbr/Contents/Home"`; `local.properties` → `sdk.dir=/Users/pacific2rim/Library/Android/sdk` (gitignored).

## Review Focus

1. **TV's IP changed (DHCP lease)** — stored IP no longer answers. Expect: rediscover by SSDP, match the stored TV by UUID, update its IP, connect. Test: `refreshIps` in Task 6.
2. **TV off / unreachable on app open** — expect a "TV is off — tap Power" state within ~3 s, never a hang. Test: `connect_unreachable_returnsUnreachableQuickly` in Task 5.
3. **Pairing prompt declined or ignored on the TV** — expect a "Pairing was declined" message and a retry button, not a stuck spinner. Test: `connect_registerError_returnsRejected` in Task 5.
4. **Stored client-key no longer valid** (TV reset, app removed on TV) — TV prompts again and returns a new key; expect the new key to replace the old. Test: `connect_newKey_replacesStoredKey` in Task 5.
5. **Socket dies mid-use** (Wi-Fi blip, TV switched off by its own remote) — expect state `Disconnected` so the UI shows it and reconnects on next resume/tap, rather than buttons silently doing nothing. Test: `serverClose_setsDisconnected` in Task 5.

---

## File Structure

```
settings.gradle.kts, build.gradle.kts, gradle.properties, gradle/wrapper/*, gradlew
app/build.gradle.kts
app/src/main/AndroidManifest.xml
app/src/main/java/com/prashantkashyap/airpointer/
  tv/PointerCommands.kt   pointer-socket text frames + TvKeys constants
  tv/Ssap.kt              SSAP request/register builders, reply parsers
  tv/Manifest.kt          signed client manifest constant (verbatim, attributed)
  tv/PairedTv.kt          PairedTv model + JSON encode/decode + refreshIps
  tv/TvStore.kt           SharedPreferences persistence of PairedTv list + sensitivity
  tv/CertPin.kt           fingerprint(), PinningTrustManager, TvHttp.client()
  tv/TvConnection.kt      websocket session: connect/register, request, subscribe, openPointer
  tv/Discovery.kt         SSDP parse + discover()
  tv/WakeOnLan.kt         magicPacket(), wake()
  motion/PointerMath.kt   gyro+gravity -> integer dx/dy with dead zone and remainder carry
  motion/StillDetector.kt put-down detection
  motion/PointerEngine.kt SensorManager wiring -> PointerMath -> PointerChannel
  RemoteController.kt     one object the UI talks to (connection, keys, inputs, text, power)
  ui/MainActivity.kt      lifecycle, volume keys, keep-screen-on/dim, haptics, gyro gate
  ui/RemoteScreen.kt      the approved layout
  ui/TvPickerScreen.kt    discovered TVs, manual IP/MAC, pair, forget
  ui/TextEntry.kt         Type dialog + Speak confirm dialog
  ui/SettingsSheet.kt     sensitivity slider, TV picker entry
app/src/test/java/com/prashantkashyap/airpointer/...   unit tests mirroring the above
app/src/test/resources/tv-cert-a.pem, tv-cert-b.pem    test certificates
docs/INSTALL.md           family install note
```

---

### Task 1: Project scaffold + pointer-socket frames

**Files:**
- Create: Gradle files listed above, `app/src/main/AndroidManifest.xml`, `ui/MainActivity.kt` (empty Compose screen with the gyro gate), `tv/PointerCommands.kt`
- Modify: `.gitignore` (add `local.properties`, `.gradle/`, `build/`, `app/build/`, `.idea/`, `*.iml`, `keystore.properties`, `*.jks`)
- Test: `app/src/test/java/com/prashantkashyap/airpointer/tv/PointerCommandsTest.kt`

**Interfaces:**
- Produces: `object PointerCommands { fun move(dx: Int, dy: Int): String; fun click(): String; fun scroll(dy: Int): String; fun button(name: String): String }`; `object TvKeys { const val HOME="HOME"; BACK="BACK"; MENU="MENU"; MUTE="MUTE"; VOLUME_UP="VOLUMEUP"; VOLUME_DOWN="VOLUMEDOWN"; CHANNEL_UP="CHANNELUP"; CHANNEL_DOWN="CHANNELDOWN"; RECENT="RECENT"; ENTER="ENTER" }`

- [ ] **Step 1: Generate wrapper and build files.** `gradle wrapper --gradle-version 8.11.1` (`brew install gradle` if missing). Plugins: `com.android.application` 8.7.3, `org.jetbrains.kotlin.android` 2.0.21, `org.jetbrains.kotlin.plugin.compose` 2.0.21. Manifest per Global Constraints, one activity `ui.MainActivity`, launcher intent, portrait.
- [ ] **Step 2: Write the failing test**

```kotlin
@Test fun frames() {
    assertEquals("type:move\ndx:5\ndy:-3\ndown:0\n\n", PointerCommands.move(5, -3))
    assertEquals("type:click\n\n", PointerCommands.click())
    assertEquals("type:scroll\ndx:0\ndy:-2\n\n", PointerCommands.scroll(-2))
    assertEquals("type:button\nname:HOME\n\n", PointerCommands.button(TvKeys.HOME))
}
```

- [ ] **Step 3: Run** `./gradlew :app:testDebugUnitTest --tests '*PointerCommandsTest*'` → FAIL (unresolved reference).
- [ ] **Step 4: Implement `PointerCommands` and `TvKeys`.**
- [ ] **Step 5: Run** the test → PASS; `./gradlew :app:assembleDebug` → BUILD SUCCESSFUL.
- [ ] **Step 6: Commit** `feat: scaffold Android app and pointer-socket frames`

### Task 2: Pointer maths + put-down detection

**Files:**
- Create: `motion/PointerMath.kt`, `motion/StillDetector.kt`
- Test: `motion/PointerMathTest.kt`, `motion/StillDetectorTest.kt`

**Interfaces:**
- Produces: `class PointerMath(var sensitivity: Float = 1f) { fun step(gyro: FloatArray, gravity: FloatArray, dtSec: Float): Pair<Int, Int> }` (gyro rad/s device frame; gravity m/s² device frame as `TYPE_GRAVITY` reports it, pointing up); `class StillDetector { fun update(gyro: FloatArray, gravity: FloatArray, nowMs: Long): Boolean /* true = paused */ }`
- Constants (in `PointerMath` companion): `BASE_GAIN = 2700f` px per radian (~40° sweep = full 1920 px width), `DEAD_ZONE = 0.02f` rad/s, sensitivity clamp `0.5f..2.0f`. Mark `// ponytail: linear gain + dead zone; add an acceleration curve only if fine aiming feels hard on the real TV`.

Algorithm (not determined by the signature):
```
u = normalize(gravity)                 // "up" in device frame
p = (0, 1, 0)                          // phone tip
r = p × u; if |r| < 0.1 use (1,0,0) else normalize(r)   // horizontal "right" axis
yawRate   = gyro · u ; pitchRate = gyro · r
zero each rate whose |rate| < DEAD_ZONE
fx += -yawRate   * BASE_GAIN * sensitivity * dtSec + carryX
fy += -pitchRate * BASE_GAIN * sensitivity * dtSec + carryY
emit (fx.toInt(), fy.toInt()); carry = remainder
```
StillDetector: paused when `|u.z| > 0.97` (flat, either face) and `|gyro| < 0.05` rad/s continuously for 1000 ms; unpauses on the first sample that breaks either condition.

- [ ] **Step 1: Write failing tests**

```kotlin
val upright = floatArrayOf(0f, 0f, 9.81f)                // flat, screen up
val tilted  = floatArrayOf(0f, 6.94f, 6.94f)             // tip raised 45°
@Test fun turnLeft_flat_movesLeft()  { val (dx, dy) = PointerMath().step(floatArrayOf(0f,0f,1f), upright, 0.01f); assertEquals(-27, dx); assertEquals(0, dy) }
@Test fun turnLeft_tilted_sameAsFlat() { val (dx, _) = PointerMath().step(floatArrayOf(0f, 0.7071f, 0.7071f), tilted, 0.01f); assertEquals(-27, dx) }
@Test fun tipUp_movesUp()            { val (_, dy) = PointerMath().step(floatArrayOf(1f,0f,0f), upright, 0.01f); assertEquals(-27, dy) }
@Test fun tremor_isIgnored()         { assertEquals(0 to 0, PointerMath().step(floatArrayOf(0.01f,0.01f,0.01f), upright, 0.01f)) }
@Test fun fractions_carryOver()      { val m = PointerMath(); val slow = floatArrayOf(0f, 0f, 0.03f)   // -0.081 px per step
                                        assertEquals(0, (1..10).sumOf { m.step(slow, upright, 0.001f).first })   // -0.81 so far
                                        assertEquals(-1, (1..10).sumOf { m.step(slow, upright, 0.001f).first }) } // -1.62 total
@Test fun sensitivity_scales()       { val (dx, _) = PointerMath(sensitivity = 2f).step(floatArrayOf(0f,0f,1f), upright, 0.01f); assertEquals(-54, dx) }
@Test fun stillFlatForOneSecond_pauses() { val s = StillDetector(); assertFalse(s.update(FloatArray(3), upright, 0)); assertFalse(s.update(FloatArray(3), upright, 999)); assertTrue(s.update(FloatArray(3), upright, 1000)); assertFalse(s.update(floatArrayOf(0.2f,0f,0f), upright, 1010)) }
@Test fun stillButHeldUpright_doesNotPause() { val s = StillDetector(); s.update(FloatArray(3), tilted, 0); assertFalse(s.update(FloatArray(3), tilted, 2000)) }
```

- [ ] **Step 2: Run** `./gradlew :app:testDebugUnitTest --tests '*motion*'` → FAIL.
- [ ] **Step 3: Implement** both classes per the algorithm.
- [ ] **Step 4: Run** → PASS.
- [ ] **Step 5: Commit** `feat: gravity-aligned gyro pointer maths and put-down detection`

### Task 3: SSAP messages, PairedTv model and store

**Files:**
- Create: `tv/Ssap.kt`, `tv/Manifest.kt`, `tv/PairedTv.kt`, `tv/TvStore.kt`
- Modify: `app/build.gradle.kts` (add `testImplementation("org.json:json:20240303")`)
- Test: `tv/SsapTest.kt`, `tv/PairedTvTest.kt`

**Interfaces:**
- Produces:
  - `object Ssap { fun register(clientKey: String?): JSONObject; fun request(id: String, uri: String, payload: JSONObject = JSONObject()): JSONObject; fun subscribe(id: String, uri: String, payload: JSONObject = JSONObject()): JSONObject; fun parseRegisterReply(msg: JSONObject): RegisterReply }`
  - `sealed interface RegisterReply { data object Prompting; data class Registered(val clientKey: String); data class Error(val message: String) }`
  - URIs as constants in `Ssap`: `POINTER_SOCKET="ssap://com.webos.service.networkinput/getPointerInputSocket"`, `TURN_OFF="ssap://system/turnOff"`, `INPUT_LIST="ssap://tv/getExternalInputList"`, `SWITCH_INPUT="ssap://tv/switchInput"`, `REGISTER_KEYBOARD="ssap://com.webos.service.ime/registerRemoteKeyboard"`, `INSERT_TEXT="ssap://com.webos.service.ime/insertText"`, `SEND_ENTER="ssap://com.webos.service.ime/sendEnterKey"`, `CONNECTION_INFO="ssap://com.webos.service.connectionmanager/getinfo"`
  - `data class PairedTv(val id: String, val name: String, val ip: String, val mac: String?, val clientKey: String?, val certPin: String?, val secure: Boolean?, val lastUsedMs: Long)`; `fun encodeTvs(tvs: List<PairedTv>): String`; `fun decodeTvs(json: String): List<PairedTv>` (bad/empty JSON → empty list)
  - `class TvStore(context: Context) { fun all(): List<PairedTv>; fun upsert(tv: PairedTv); fun forget(id: String); fun lastUsed(): PairedTv?; var sensitivity: Float /* default 1f */ }`
- `Manifest.kt`: `val CLIENT_MANIFEST: JSONObject` built from the signed `manifest` object used by open-source webOS clients (copy verbatim from `aiowebostv`'s `handshake.py`, Apache-2.0; put the source URL and licence in a comment). It must include permissions `CONTROL_POWER`, `CONTROL_INPUT_TEXT`, `CONTROL_MOUSE_AND_KEYBOARD`, `READ_INPUT_DEVICE_LIST`, `CONTROL_INPUT_TV`, `READ_NETWORK_STATE`.

- [ ] **Step 1: Write failing tests**

```kotlin
@Test fun register_withoutKey() { val m = Ssap.register(null); assertEquals("register", m.getString("type")); val p = m.getJSONObject("payload"); assertEquals("PROMPT", p.getString("pairingType")); assertFalse(p.has("client-key")); assertTrue(p.has("manifest")) }
@Test fun register_withKey()    { assertEquals("abc", Ssap.register("abc").getJSONObject("payload").getString("client-key")) }
@Test fun request_shape()       { val m = Ssap.request("7", Ssap.TURN_OFF); assertEquals("request", m.getString("type")); assertEquals("7", m.getString("id")); assertEquals(Ssap.TURN_OFF, m.getString("uri")) }
@Test fun reply_prompting()     { assertEquals(RegisterReply.Prompting, Ssap.parseRegisterReply(JSONObject("""{"type":"response","payload":{"pairingType":"PROMPT","returnValue":true}}"""))) }
@Test fun reply_registered()    { assertEquals(RegisterReply.Registered("k1"), Ssap.parseRegisterReply(JSONObject("""{"type":"registered","payload":{"client-key":"k1"}}"""))) }
@Test fun reply_error()         { assertTrue(Ssap.parseRegisterReply(JSONObject("""{"type":"error","error":"403 denied"}""")) is RegisterReply.Error) }
@Test fun tvs_roundTrip()       { val t = PairedTv("uuid:1","Living room","192.168.1.20","AA:BB:CC:DD:EE:FF","k","sha256/x",true,5); assertEquals(listOf(t), decodeTvs(encodeTvs(listOf(t)))) }
@Test fun tvs_nullsRoundTrip()  { val t = PairedTv("ip:1","TV","1.2.3.4",null,null,null,null,0); assertEquals(listOf(t), decodeTvs(encodeTvs(listOf(t)))) }
@Test fun tvs_garbage_isEmpty() { assertEquals(emptyList<PairedTv>(), decodeTvs("not json")) }
```

- [ ] **Step 2: Run** `--tests '*SsapTest*' --tests '*PairedTvTest*'` → FAIL.
- [ ] **Step 3: Implement** `Ssap`, `Manifest.kt`, `PairedTv` encode/decode, `TvStore` (SharedPreferences `"airpointer"`, keys `"tvs"`, `"sensitivity"`; `lastUsed()` = max `lastUsedMs`).
- [ ] **Step 4: Run** → PASS.
- [ ] **Step 5: Commit** `feat: SSAP messages and paired-TV store`

### Task 4: Certificate pinning (trust on first use)

**Files:**
- Create: `tv/CertPin.kt`, `app/src/test/resources/tv-cert-a.pem`, `tv-cert-b.pem`
- Test: `tv/CertPinTest.kt`

**Interfaces:**
- Produces: `fun fingerprint(cert: X509Certificate): String` → `"sha256/" + Base64(SHA-256(cert.encoded))` (NO_WRAP); `class PinningTrustManager(private val expectedPin: String?, private val onFirstSeen: (String) -> Unit) : X509TrustManager` — `checkServerTrusted`: pin null → call `onFirstSeen(fp)` and accept; pin equal → accept; else throw `CertificateException("TV certificate changed")`; `object TvHttp { fun client(pin: String?, onFirstSeen: (String) -> Unit): OkHttpClient }` — connect/read timeout 3 s, pingInterval 10 s, `sslSocketFactory` from the trust manager, `hostnameVerifier { _, _ -> true }` with comment `// safe: identity comes from the pinned fingerprint, TVs present self-signed certs without our IP`.

- [ ] **Step 1: Create test certs**: `openssl req -x509 -newkey rsa:2048 -nodes -keyout /dev/null -subj /CN=tv-a -days 3650 -out app/src/test/resources/tv-cert-a.pem` and the same with `tv-b`.
- [ ] **Step 2: Write failing tests**

```kotlin
val a = load("tv-cert-a.pem"); val b = load("tv-cert-b.pem")
@Test fun firstUse_acceptsAndReportsPin() { var seen: String? = null; PinningTrustManager(null) { seen = it }.checkServerTrusted(arrayOf(a), "RSA"); assertEquals(fingerprint(a), seen) }
@Test fun samePin_accepts()   { PinningTrustManager(fingerprint(a)) { fail() }.checkServerTrusted(arrayOf(a), "RSA") }
@Test fun changedCert_refuses() { assertThrows(CertificateException::class.java) { PinningTrustManager(fingerprint(a)) {}.checkServerTrusted(arrayOf(b), "RSA") } }
@Test fun fingerprint_format() { assertTrue(fingerprint(a).startsWith("sha256/")); assertNotEquals(fingerprint(a), fingerprint(b)) }
```

- [ ] **Step 3: Run** `--tests '*CertPinTest*'` → FAIL. **Step 4: Implement.** **Step 5: Run** → PASS.
- [ ] **Step 6: Commit** `feat: trust-on-first-use certificate pinning for TV websockets`

### Task 5: TvConnection (websocket session)

**Files:**
- Create: `tv/TvConnection.kt`
- Modify: `app/build.gradle.kts` (add `testImplementation("com.squareup.okhttp3:mockwebserver:4.12.0")`, `testImplementation("org.jetbrains.kotlinx:kotlinx-coroutines-test:1.9.0")`)
- Test: `tv/TvConnectionTest.kt`

**Interfaces:**
- Consumes: `Ssap`, `RegisterReply`, `PointerCommands`, `TvHttp.client`, `PairedTv`.
- Produces:
  - `fun endpointsFor(tv: PairedTv): List<String>` → `wss://ip:3001`, `ws://ip:3000`, swapped when `tv.secure == false`.
  - `sealed interface ConnState { data object Disconnected; data object Connecting; data object WaitingForTvApproval; data object Connected }`
  - `sealed interface ConnectResult { data class Connected(val tv: PairedTv /* updated clientKey, certPin, secure */); data object Unreachable; data object Rejected; data object CertChanged }`
  - `class TvConnection(private val endpoints: List<String>, private val clientFor: (pin: String?, onPin: (String) -> Unit) -> OkHttpClient = TvHttp::client) { val state: StateFlow<ConnState>; suspend fun connect(tv: PairedTv, approvalTimeoutMs: Long = 60_000): ConnectResult; suspend fun request(uri: String, payload: JSONObject = JSONObject()): JSONObject; fun subscribe(uri: String, payload: JSONObject = JSONObject(), onEvent: (JSONObject) -> Unit); suspend fun openPointer(): PointerChannel; fun close() }`
  - `class TvError(message: String) : Exception(message)`
  - `class PointerChannel(private val ws: WebSocket) { fun move(dx: Int, dy: Int); fun click(); fun scroll(dy: Int); fun button(name: String); fun close() }`
- Behaviour: try endpoints in order, first that opens wins (`secure = url.startsWith("wss")`); send `Ssap.register(tv.clientKey)`; `Prompting` → state `WaitingForTvApproval`; `Registered` → state `Connected`, return updated tv; `Error` or approval timeout → `Rejected`; TLS `CertificateException` → `CertChanged` (do not try ws fallback); all endpoints fail to open → `Unreachable`. Request ids are incrementing strings; `request` completes on the reply with the same id, throws `TvError(message)` on `type:"error"` or 3 s timeout. Server close/failure → state `Disconnected`, pending requests fail. `openPointer()` = `request(Ssap.POINTER_SOCKET)` → `payload.socketPath` → open with the same client.

- [ ] **Step 1: Write failing tests** (MockWebServer with `withWebSocketUpgrade`, scripted listener replying to `register`/`request` frames; endpoints = `listOf(server.url("/").toString().replace("http","ws"))`)

```kotlin
@Test fun connect_firstTime_promptsThenRegisters()   // server replies Prompting then Registered("k1"): states pass WaitingForTvApproval → Connected; result tv.clientKey == "k1"
@Test fun connect_withKey_sendsKey()                 // register frame received by server has payload.client-key == "old"
@Test fun connect_newKey_replacesStoredKey()         // tv.clientKey "old", server replies Registered("new") → result tv.clientKey == "new"
@Test fun connect_registerError_returnsRejected()    // server replies {"type":"error","error":"403 user denied"} → Rejected, state Disconnected
@Test fun connect_unreachable_returnsUnreachableQuickly() // endpoints = listOf("ws://10.255.255.1:3000") → Unreachable, elapsed < 4000 ms
@Test fun connect_fallsBackToSecondEndpoint()        // endpoints = [dead ws url, server url] → Connected, result tv.secure == false
@Test fun request_matchesReplyById()                 // two concurrent requests, server answers in reverse order → each gets its own payload
@Test fun request_error_throwsTvError()
@Test fun openPointer_opensSocketPathAndSendsFrames() // server returns socketPath = second mock ws; pointer.move(3,4) arrives as PointerCommands.move(3,4)
@Test fun serverClose_setsDisconnected()
```

- [ ] **Step 2: Run** `--tests '*TvConnectionTest*'` → FAIL. **Step 3: Implement.** **Step 4: Run** → PASS.
- [ ] **Step 5: Commit** `feat: TV websocket session with pairing, requests and pointer channel`

### Task 6: Discovery and Wake-on-LAN

**Files:**
- Create: `tv/Discovery.kt`, `tv/WakeOnLan.kt`
- Modify: `tv/PairedTv.kt` (add `refreshIps`)
- Test: `tv/DiscoveryTest.kt`, `tv/WakeOnLanTest.kt`, `tv/PairedTvTest.kt`

**Interfaces:**
- Produces:
  - `data class SsdpHit(val uuid: String, val ip: String, val location: String)`; `fun parseSsdp(response: String): SsdpHit?` — needs status 200, `ST`/`NT` or `USN` containing `webos-second-screen`; `uuid` = USN up to `::`; ip = host of `LOCATION`; headers case-insensitive.
  - `data class DiscoveredTv(val hit: SsdpHit, val name: String)`; `suspend fun discover(context: Context, timeoutMs: Long = 2500): List<DiscoveredTv>` — holds a `WifiManager.MulticastLock` only for the call; M-SEARCH to `239.255.255.250:1900` with `ST: urn:lge-com:service:webos-second-screen:1`, `MX: 2`; name = `<friendlyName>` from GET `location` (fallback ip); de-duplicate by uuid.
  - `fun refreshIps(stored: List<PairedTv>, hits: List<SsdpHit>): List<PairedTv>` — same `id == uuid`, different ip → copy with new ip.
  - `fun magicPacket(mac: String): ByteArray`; `suspend fun wake(mac: String)` — UDP to `255.255.255.255:9`, broadcast enabled, sent 3× 100 ms apart (`// ponytail: 3 sends cover UDP loss on Wi-Fi`).

- [ ] **Step 1: Write failing tests**

```kotlin
val ssdp = "HTTP/1.1 200 OK\r\nLOCATION: http://192.168.1.20:1817/\r\nST: urn:lge-com:service:webos-second-screen:1\r\nUSN: uuid:abc-123::urn:lge-com:service:webos-second-screen:1\r\n\r\n"
@Test fun parses_lg() { assertEquals(SsdpHit("uuid:abc-123","192.168.1.20","http://192.168.1.20:1817/"), parseSsdp(ssdp)) }
@Test fun parses_lowercaseHeaders() { assertNotNull(parseSsdp(ssdp.lowercase().replace("http/1.1 200 ok","HTTP/1.1 200 OK"))) }
@Test fun ignores_otherDevices() { assertNull(parseSsdp(ssdp.replace("webos-second-screen","dial"))) }
@Test fun refreshIps_updatesMovedTv() { val t = PairedTv("uuid:abc-123","TV","192.168.1.5",null,null,null,null,0); assertEquals("192.168.1.20", refreshIps(listOf(t), listOf(parseSsdp(ssdp)!!)).single().ip) }
@Test fun magic_layout() { val p = magicPacket("AA:BB:CC:DD:EE:FF"); assertEquals(102, p.size); assertTrue(p.take(6).all { it == 0xFF.toByte() }); assertEquals(0xAA.toByte(), p[6]); assertEquals(0xFF.toByte(), p[101]) }
@Test fun magic_dashes() { assertArrayEquals(magicPacket("AA:BB:CC:DD:EE:FF"), magicPacket("aa-bb-cc-dd-ee-ff")) }
@Test fun magic_bad() { assertThrows(IllegalArgumentException::class.java) { magicPacket("AA:BB") } }
```

- [ ] **Step 2: Run** → FAIL. **Step 3: Implement.** **Step 4: Run** → PASS.
- [ ] **Step 5: Commit** `feat: SSDP discovery and Wake-on-LAN`

### Task 7: Pointer spike on both TVs — pairing, pointer, power (CHECKPOINT)

**Files:**
- Create: `motion/PointerEngine.kt`, `RemoteController.kt`, `ui/TvPickerScreen.kt`, `ui/RemoteScreen.kt` (click area + Power only for now)
- Modify: `ui/MainActivity.kt`

**Interfaces:**
- Consumes: Tasks 2–6.
- Produces:
  - `class PointerEngine(context: Context, private val math: PointerMath, private val still: StillDetector, private val sink: (dx: Int, dy: Int) -> Unit) { fun start(); fun stop(); val lastMotionMs: Long }` — registers `TYPE_GYROSCOPE` and `TYPE_GRAVITY` at 10 000 µs; on each gyro event uses latest gravity; skips when paused or `(0,0)`.
  - `class RemoteController(context: Context, private val store: TvStore, scope: CoroutineScope) { val state: StateFlow<ConnState>; val currentTv: StateFlow<PairedTv?>; suspend fun connect(tv: PairedTv): ConnectResult; fun pointer(): PointerChannel?; fun key(name: String); fun click(); fun scroll(dy: Int); suspend fun power(); fun stop() }`
  - `connect`: on success store the updated tv with `lastUsedMs = now`, then `request(Ssap.CONNECTION_INFO)` and save `wiredInfo.macAddress` if present else `wifiInfo.macAddress` (ignore failure), then `openPointer()`. On `Unreachable` → run `discover` + `refreshIps`, store, retry once.
  - `power()`: connected → `request(Ssap.TURN_OFF)`; not connected and `mac != null` → `wake(mac)`, then retry `connect` every 1 s for up to 15 s; no mac → state message "Turn the TV on once with its remote so Air Pointer can learn its address".
- MainActivity: gyro gate; `onResume` → `FLAG_KEEP_SCREEN_ON`, connect `store.lastUsed()` (or show picker), `engine.start()`; `onPause` → `engine.stop()`, `controller.stop()` (closes sockets), clear flag. Picker: discovered list + "Enter IP" + optional MAC field; tapping shows "Accept the prompt on your TV" while `WaitingForTvApproval`; `Rejected` → "Pairing was declined on the TV" + Retry; `CertChanged` → "This TV's security certificate changed. Forget and pair again?" with Forget button; `Unreachable` → "TV is off — tap Power".

- [ ] **Step 1: Implement** the files above.
- [ ] **Step 2: Run** `./gradlew :app:testDebugUnitTest :app:assembleDebug` → all PASS, BUILD SUCCESSFUL.
- [ ] **Step 3: Install** `adb install -r app/build/outputs/apk/debug/app-debug.apk` on the owner's phone.
- [ ] **Step 4: CHECKPOINT with the owner, on each TV (LJ617T and 55UT7550). Record results in `docs/spike-results.md`:**
  1. Pairing: which endpoint connected (expect ws:3000 on LJ617T, wss:3001 on 55UT7550); prompt appeared once; reopening the app does not prompt again and is usable within ~1 s.
  2. Pointer lag: owner compares sweep/stop with the Magic Remote; owner verdict "as quick" / "noticeably slower". Note `dx/dy` rate sent (log frames per second).
  3. Pointing feel: pivoting the tip moves the cursor; flat and tilted hold behave the same; phone laid flat stops the cursor.
  4. Power off via app; then power on via app from fully off (wait 2 min after off). MAC captured automatically? (y/n).
  5. Leave the app; check `adb shell dumpsys sensorservice | grep -i airpointer` shows no active registration.
  **Stop and ask the owner** if lag is "noticeably slower", pairing fails on either TV, or power-on fails on either TV (fallbacks per spec: IR blaster or off-only). Do not start Task 8 until the owner says go.
- [ ] **Step 5: Commit** `feat: pointer spike — pairing, gyro pointer, power` (+ `docs/spike-results.md`)

### Task 8: Full remote layout, keys, inputs, volume keys, haptics

**Files:**
- Modify: `ui/RemoteScreen.kt`, `ui/MainActivity.kt`, `RemoteController.kt`
- Test: `RemoteControllerKeysTest.kt` (pure helper below)

**Interfaces:**
- Produces: `data class TvInput(val id: String, val label: String)`; `fun parseInputs(payload: JSONObject): List<TvInput>` (from `payload.devices[].id/label`); `RemoteController.inputs(): List<TvInput>`, `RemoteController.switchInput(id: String)` (payload `{"inputId": id}`).
- Layout exactly as spec "Screen layout": row Power·Inputs·Menu·Mute; click area (weight so it takes ~55% of height); Back·Home·Recents; Volume −/+ pill and Channel ▼/▲ pill; Speak·Type (disabled placeholders until Task 9).
- Click area: tap → `click()`; vertical drag → `scroll(-dragPx / 8)` (`// ponytail: 8 px per scroll step, tune on the TV`); long-press → see checkpoint.
- Recents button → `key(TvKeys.RECENT)`.
- `MainActivity.onKeyDown`: `KEYCODE_VOLUME_UP/DOWN` → `key(VOLUME_UP/DOWN)`, return true (only while foreground, which is the only time the activity receives keys).
- Haptics: `LocalHapticFeedback.current.performHapticFeedback(HapticFeedbackType.TextHandleMove)` on click and each button.
- Inputs button → bottom sheet listing `inputs()` labels; tap → `switchInput`.

- [ ] **Step 1: Write failing test**

```kotlin
@Test fun parseInputs_usesTvLabels() {
    val p = JSONObject("""{"devices":[{"id":"HDMI_1","label":"PlayStation"},{"id":"HDMI_2","label":"HDMI 2"}]}""")
    assertEquals(listOf(TvInput("HDMI_1","PlayStation"), TvInput("HDMI_2","HDMI 2")), parseInputs(p))
}
@Test fun parseInputs_missing_isEmpty() { assertEquals(emptyList<TvInput>(), parseInputs(JSONObject("{}"))) }
```

- [ ] **Step 2: Run** → FAIL. **Step 3: Implement** helper + UI. **Step 4: Run** tests + `assembleDebug` → PASS.
- [ ] **Step 5: CHECKPOINT on both TVs:** every button does what its label says. If `RECENT` does nothing on a TV, change Recents to `HOME` for that case (`// ponytail: RECENT unsupported on <model>`). If `MENU` doesn't open settings, try `QMENU`. Long-press: try sending `PointerCommands.button(TvKeys.ENTER)` held ≥ 800 ms via repeated frames; if neither TV reacts, drop long-press and tell the owner. Record outcomes in `docs/spike-results.md`.
- [ ] **Step 6: Commit** `feat: full remote layout, inputs, volume keys, haptics`

### Task 9: Type and Speak

**Files:**
- Create: `ui/TextEntry.kt`
- Modify: `RemoteController.kt`, `ui/RemoteScreen.kt`
- Test: `RemoteControllerTextTest.kt`

**Interfaces:**
- Produces: `fun keyboardFocused(event: JSONObject): Boolean` — true when `payload.currentWidget.focus == true` (missing → false); `RemoteController.keyboardOpen: StateFlow<Boolean>` fed by `subscribe(Ssap.REGISTER_KEYBOARD)` after connect; `RemoteController.sendText(text: String)` → `request(Ssap.INSERT_TEXT, {"text": text, "replace": 0})` then `request(Ssap.SEND_ENTER)`.
- Speak/Type enabled only while `keyboardOpen`. Type → dialog with a `TextField`, Send button. Speak → `RecognizerIntent.ACTION_RECOGNIZE_SPEECH` via `rememberLauncherForActivityResult` (`EXTRA_LANGUAGE_MODEL_FREE_FORM`, phone default language); result shown in the same dialog, editable, one tap Send. No `RECORD_AUDIO` permission (the system recogniser owns the mic).

- [ ] **Step 1: Write failing tests**

```kotlin
@Test fun focus_true()  { assertTrue(keyboardFocused(JSONObject("""{"payload":{"currentWidget":{"focus":true}}}"""))) }
@Test fun focus_false() { assertFalse(keyboardFocused(JSONObject("""{"payload":{"currentWidget":{"focus":false}}}"""))) }
@Test fun focus_missing() { assertFalse(keyboardFocused(JSONObject("""{"payload":{}}"""))) }
```

- [ ] **Step 2: Run** → FAIL. **Step 3: Implement.** **Step 4: Run** → PASS.
- [ ] **Step 5: CHECKPOINT on both TVs:** open YouTube search; Speak/Type light up; typed and spoken text arrives and submits; buttons dim when the TV keyboard closes. If the webOS 3.5 TV never reports focus, enable the buttons whenever connected on that TV and note it in `docs/spike-results.md`.
- [ ] **Step 6: Commit** `feat: keyboard and voice text entry`

### Task 10: Dim when idle, settings, multiple TVs

**Files:**
- Create: `ui/SettingsSheet.kt`
- Modify: `ui/MainActivity.kt`, `ui/RemoteScreen.kt`, `ui/TvPickerScreen.kt`

**Interfaces:**
- Consumes: `TvStore.sensitivity`, `TvStore.forget`, `PointerEngine.lastMotionMs`.
- Dim: a 1 s ticker while resumed; if no touch and no pointer motion for 30 000 ms → `window.attributes.screenBrightness = 0.1f`; any touch or motion → `BRIGHTNESS_OVERRIDE_NONE`. Ticker cancelled in `onPause`.
- Settings (gear icon in the top row's corner, small): sensitivity slider 0.5–2.0 step 0.1 writing `TvStore.sensitivity` and `PointerMath.sensitivity` live; "Switch TV" → picker; picker rows have Forget.

- [ ] **Step 1: Implement.** **Step 2: Run** `./gradlew :app:testDebugUnitTest :app:assembleDebug` → PASS.
- [ ] **Step 3: CHECKPOINT:** idle 30 s dims and recovers; sensitivity change felt immediately; switch between the two TVs without re-pairing; with the app in background for 10 min, battery usage screen shows no Air Pointer activity.
- [ ] **Step 4: Commit** `feat: idle dimming, sensitivity setting, TV switching`

### Task 11: Release build for family

**Files:**
- Modify: `app/build.gradle.kts` (release `signingConfig` read from `keystore.properties` keys `storeFile`, `storePassword`, `keyAlias`, `keyPassword`; skip signing config when the file is absent)
- Create: `docs/INSTALL.md`

- [ ] **Step 1: Owner creates the key** (owner runs, keeps it safe, never committed): `keytool -genkeypair -v -keystore ~/airpointer-release.jks -alias airpointer -keyalg RSA -keysize 2048 -validity 10000`, and writes `keystore.properties`.
- [ ] **Step 2: Build** `./gradlew :app:assembleRelease` → `app/build/outputs/apk/release/app-release.apk`; `apksigner verify --print-certs` shows the airpointer cert.
- [ ] **Step 3: Write `docs/INSTALL.md`** (≤ 15 lines, plain language): send the APK file; tap it; allow "Install unknown apps" for the app that opened it; open Air Pointer; pick your TV; press Allow on the TV. Updates: install the newer APK over the old one (same key keeps settings and pairing).
- [ ] **Step 4: Verify** `git status` shows no `.jks` or `keystore.properties`.
- [ ] **Step 5: Commit** `build: release signing and family install guide`

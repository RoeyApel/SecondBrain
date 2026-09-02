		**Data that has no source anywhere in the system today**

- **`batteryPercentage`** — not tracked for cameras at all (no battery data is stored or read from any camera).
- **`speed`** — cameras are fixed/mounted, so there's no concept of a camera "moving." This field only makes sense once drones are added.
- **`footprint`** (the polygon showing what area a sensor covers) — not being computed in this first version. Would need to be added later.

**Data that's only sometimes available**

- **`coverage` (pan/tilt/field-of-view)** — only comes through for certain camera brands (the SOAP-based ones), and only while that camera is actively connected. Other camera brands (ONVIF, KELA) won't report this at all right now.
- **`heading`** — only available when `coverage` is available (it's derived from the same live feed). For cameras without live coverage data, this will be missing too.
- **`maxRange_m`** — depends on a terrain-calculation cache; if that service is cold or unreachable, this will be missing for that sensor rather than failing the whole response.

**Problems with the example response itself**

- **`stationID`** — the way we're planning to figure out which station a camera belongs to relies on an assumption about how spotters and stations are linked in the data, which hasn't been confirmed against the actual product/domain logic yet. If the assumption is wrong, this field could silently show `null` or the wrong station for some sensors.

**Operational caveat**

- If the service handling live camera data runs multiple copies (for scale), a camera's live coverage data may only be visible to whichever copy is actually connected to it — so under scale, some sensors' `coverage` could intermittently disappear from the response even though the camera is fine. Acceptable for now, but worth knowing about.
# `GET /api/v1/sensors/state` on Lillit

  

## Context

  

Lillit is the machine-to-machine bridge between an external mission system ("safepass") and

the Jupiter platform. It currently exposes exactly one business route,

`POST /api/v1/safepass/missions` ([mission.controller.ts](apps/lillit/src/mission/mission.controller.ts)),

which takes control of a sensor on the mission system's behalf.

  

The mission system now needs the inverse: a periodic snapshot of **all sensors and their most

current state** — identity, type, status, mounting station, geographic position, and the area

each sensor is currently covering — so it can render them on its own map. This plan adds

`GET /api/v1/sensors/state` to serve that snapshot.

  

"Sensors" today means **cameras in the Jupiter DB**. The response contract is deliberately

shaped to accommodate future sensor types (drones/EVO) without a breaking change.

  

### The core constraint driving this design

  

`apps/lillit` has **no database access** — [package.json](apps/lillit/package.json) has no

Prisma dependency. It is an HTTP-only service with an axios client pointed at device-manager.

Everything it reports must be fetched over HTTP from another service.

  

And the data is genuinely split across two services:

  

| Data | Owner | Why |

|---|---|---|

| id, isPTZ, status, activeSpotter, position | `apps/backend` | Owns Prisma / PostGIS |

| terrain-limited max range | `apps/backend` | Owns the Redis-cached DTM viewshed |

| live pan / tilt / hFOV | `apps/device-manager-service` | Only place the live SOAP status report lives |

  

Neither service currently exposes a read route for this.

[device-manager.controller.ts](apps/device-manager-service/src/device-manager/device-manager.controller.ts)

is command-only — its single data-returning GET is `/:cameraId/streams`.

  

**Chosen approach (hybrid):** add one read route to each service, and merge in lillit.

  

---

  

## Agreed decisions

  

| Question | Decision |

|---|---|

| Data source | Hybrid — new read route on both backend and device-manager, merged in lillit |

| `stationID` | Derived via the active spotter: `CameraConn.activeSpotterId` → `Station.activeSpotterId` |

| `footprint` | **Omitted** for this iteration |

| `maxRange_m` | Included, via the existing Redis-backed `CameraViewshedService` |

| `status` | DB values passed through as-is (`ACTIVE` / `SEMI_ACTIVE` / `INACTIVE`), plus `BUSY` when `activeSpotterId != null` |

| `sensorType` | Enum `{ PTZ, EVO }`; always `PTZ` for now. `EVO` reserved for future drones |

| Missing data | Omit optional keys entirely; explicit `null` only for required-but-nullable (`stationID`) |

| Partial failure | **Fail the whole request** (502) if either upstream call fails |

  

---

  

## Assumptions to confirm at review

  

1. **`position`, not `telemetry`.** The example payload is internally inconsistent — the first

   sensor uses `"telemetry"` with *string* values, the second uses `"position"` with *numbers*.

   The specification table lists **`position` (object)** with `float` children, so the table is

   treated as authoritative: the key is `position` and all values are JSON numbers.

2. **`FAKE`-vendor (simulator) cameras are included.** Backend's existing `findCamerasForInit`

   filters `vendor != 'FAKE'`; the new query intentionally does not, so local/Tilt development

   returns data.

3. **Soft-deleted cameras excluded** (`deleted_at IS NULL`), matching every other camera query.

  

---

  

## Known data gaps (documented, not worked around)

  

These fields from the specification have **no source anywhere in the repo** and will simply be

absent from the response:

  

- **`batteryPercentage`** — no battery column on `Camera`/`CameraConn`, no battery in any DTO.

  Battery exists only in the RaaS drone protocol schema, which is not persisted.

- **`speed`** — cameras are static; there is no velocity concept for them.

- **`coverage` for ONVIF and KELA cameras** — both

  [onvif-camera-device-client.ts](apps/device-manager-service/src/devices-manager/camera-device-client/onvif-camera-device-client.ts)

  and `kela-camera-device-client.ts` **stub out** `handleDoDeviceStatusReport`, and no ONVIF

  `GetStatus` is ever issued. Live pan/tilt/hFOV exist **only for SOAP vendors**

  (`AVIV`, `BARKAN`, `NETZ`, `TZUKIT`) and only while a camera client is connected.

- **`heading`** — for a static camera the only meaningful heading is its LOS azimuth, which is

  the same value as `coverage.pan_deg`. It is emitted only when that value is available.

  

Two operational caveats worth raising in the PR:

  

- Live LOS broadcasting is gated by env `BROADCAST_CAMERA_LOS_ENABLED` (currently `false` in

  `.env.example` and in `terraform/device-manager/`). The new device-manager route reads the

  in-memory status report **directly**, so it is *not* affected by that flag — but this should

  be stated explicitly so nobody assumes otherwise.

- Device-manager's camera state is **per-process in-memory**. With more than one replica, the

  new route only sees cameras owned by the replica that answers. `CameraOwnershipRedisService`

  exists precisely because ownership is split across replicas. Acceptable at current scale;

  flag it.

  

---

  

## Response contract

  

```jsonc

{

  "timestamp": "04082026 08:43:08",

  "sensors": [

    {

      "sensorID": "e4d9b21a-...",       // Camera.id

      "sensorType": "PTZ",              // enum PTZ | EVO

      "status": "BUSY",                 // ACTIVE | SEMI_ACTIVE | INACTIVE | BUSY

      "stationID": null,                // string | null  (always present)

      "position": {                     // omitted only if the camera has no position

        "latitude": 32.1025,

        "longitude": 34.858,

        "altitude": 15.0,

        "heading": 180.0               // omitted when no live LOS

      },

      "coverage": {                     // omitted entirely for non-SOAP / disconnected cameras

        "pan_deg": 180.0,

        "tilt_deg": -10.0,

        "hfov_deg": 30.0,               // omitted if the primary optical sensor reports no FOV

        "maxRange_m": 2000              // omitted if the viewshed cache has no entry

      }

    }

  ]

}

```

  

`timestamp` is produced by the existing

[`formatMissionTimestamp`](apps/lillit/src/mission/mission-timestamp.util.ts) — it already emits

`DDMMYYYY hh:mm:ss` in UTC. Reuse it as-is; do not write a second formatter.

  

---

  

## Implementation

  

### Step 0 — shared lib: expose the client registry

  

`DevicesManagerService` holds `private readonly clients: Map<string, DeviceClient>` but has no

way to enumerate it.

  

**File:** [libs/backend-common-src/sensor-standard/src/devices-manager/devices-manager.service.ts](libs/backend-common-src/sensor-standard/src/devices-manager/devices-manager.service.ts)

  

Add a read-only accessor next to the existing `getClientByDeviceId`:

  

```ts

public getAllClients(): DeviceClient[] {

  return Array.from(this.clients.values());

}

```

  

Purely additive, so no existing consumer changes. Per `CLAUDE.md`, rebuild dependents:

`turbo build --filter=...@libraries/backend-common`.

  

---

  

### Step 1 — device-manager: live coverage per camera

  

Rather than let callers reach into the status-report internals, put the derivation on the

camera client where the north reference and the primary optical sensor already live.

  

**File:** [apps/device-manager-service/src/devices-manager/camera-device-client/camera-device-client.ts](apps/device-manager-service/src/devices-manager/camera-device-client/camera-device-client.ts)

  

Add two null-safe getters:

  

```ts

getCurrentLineOfSight(): CameraLineOfSight | null

getCurrentHfovDegrees(): number | null

```

  

- `getCurrentLineOfSight` reuses the exact arithmetic already in

  [`broadcastCameraLos`](apps/device-manager-service/src/device-manager/sensor-status-broadcaster.service.ts#L228):

  read `PedestalStatus.Azimuth._` / `.Elevation._` (mils), add

  `configurationManager.getCameraNorthRef()`, convert with `milsToDegrees` / `normalizeDegrees`

  from `@libraries/backend-dto`. Returns `null` if the pedestal report or north ref is absent.

  **Extract this into one shared private helper** so `broadcastCameraLos` and the new getter

  cannot drift apart.

- `getCurrentHfovDegrees` reads

  `getPrimarySensor()?.OpticalStatus?.FieldOfViewStatusType?.Value` and converts with

  `milsToDegrees` — the wire format is confirmed `<Value Units="Mils">230</Value>` (see the

  SOAP fixtures in `device-manager-camera-test.const.ts`). Guard on `Units === 'Mils'`; return

  `null` for an absent value or an unrecognised unit rather than silently mis-scaling.

  

**New folder:** `apps/device-manager-service/src/sensors/`

  

- `sensors.controller.ts` — `@ApiTags('sensors')`, `@MicroServiceController('sensors')`,

  `@Get('live-state')`. A dedicated base path (not the root `@MicroServiceController('')`

  controller) avoids any route-matching ambiguity with the existing `/:cameraId/*` patterns.

- `sensors.service.ts` — iterate `devicesService.getAllClients()`, keep

  `instanceof CameraDeviceClient`, map each to

  `{ sensorId, panDeg, tiltDeg, hfovDeg }` with the null-safe getters. Cameras with no LOS are

  omitted from the array entirely (absence means "no live data", which lillit handles).

- `sensors.module.ts` — register in `app.module.ts`.

  

Route: `GET /api/device-manager/sensors/live-state` → `SensorLiveStateDto[]`.

  

---

  

### Step 2 — backend: persistent sensor state + max range

  

**File:** [apps/backend/src/camera/cameras.controller.ts](apps/backend/src/camera/cameras.controller.ts)

  

Add one route to the **existing** `CamerasController`, which already injects both

`CamerasService` and `CameraViewshedService` — so **no new module wiring and no new Redis code

is required**:

  

```ts

@PublicRoute()

@UseGuards(MicroServiceKeyGuard)

@Get('sensor-state')

async getSensorState(@Query() query: SensorStateQueryDto): Promise<CameraSensorStateDto[]>

```

  

Follow the microservice-key pattern already used by `get-cameras-for-tasks` /

`by-ids` / `get-by-id` on this controller — lillit holds a microservice key, not a user JWT, so

the RBAC-guarded `GET /cameras` is not reachable from it.

  

**Query parameter:** `los` — a compact `cameraId:azimuthDeg` list

(`?los=<uuid>:180.0,<uuid>:45.5`), letting backend resolve `maxRange_m` for each camera's

*current* pan direction in the same round-trip. Validate with class-validator (`@IsOptional`,

`@Matches`) and parse defensively — malformed pairs are skipped, never fatal.

  

**File:** [apps/backend/src/camera/cameras.service.ts](apps/backend/src/camera/cameras.service.ts)

+ [cameras.sql.ts](apps/backend/src/camera/cameras.sql.ts)

  

New `getCameraSensorState(losByCameraId: Map<string, number>)`:

  

1. One `prisma.namedQueryRaw` (same pattern as `findCamerasForInit`) joining

   `jupiter.cameras` → `jupiter.cameras_connections` → `jupiter.stations`:

  

   ```sql

   SELECT

     c.id, c.is_ptz AS "isPTZ",

     ST_AsGeoJSON(c.position)::json AS position,

     cc.status, cc.active_spotter_id AS "activeSpotterId",

     s.id AS "stationId"

   FROM jupiter.cameras c

   LEFT JOIN jupiter.cameras_connections cc ON cc.camera_id = c.id

   LEFT JOIN jupiter.stations s

     ON s.active_spotter_id = cc.active_spotter_id

    AND s.deleted_at IS NULL

   WHERE c.deleted_at IS NULL

   ```

  

   `Station.activeSpotterId` is `@unique`, so the join yields at most one station per camera.

   `LEFT JOIN` on `cameras_connections` matters — a camera with no connection row must still

   appear (it maps to `INACTIVE`).

  

2. For each camera that has an entry in `losByCameraId`, call the existing

   `cameraViewshedService.getVisibleRangeKm(id, azimuth)` and convert km → m. Run these

   concurrently with `Promise.all`. The method already returns `null` on any failure, so a cold

   Redis or an unreachable GeoServer degrades to an absent `maxRange_m` rather than an error.

  

Status is mapped in **backend**, close to the data:

`activeSpotterId != null` → `BUSY`; otherwise the raw `CameraConn.status`; a missing connection

row → `INACTIVE`.

  

---

  

### Step 3 — lillit: clients, module, and mapping

  

**Refactor first:** move `apps/lillit/src/mission/deviceManagerClient/` up to

`apps/lillit/src/deviceManagerClient/`. It is about to have a second consumer, so it no longer

belongs inside `mission/`. Update the import in

[mission.module.ts](apps/lillit/src/mission/mission.module.ts).

  

**New:** `apps/lillit/src/backendClient/` — `backendClient.module.ts` + `backendClient.service.ts`.

Copy the structure of

[device-manager-service's BackendClientModule](apps/device-manager-service/src/backendClient/backendClient.module.ts)

verbatim: `HttpModule.registerAsync` with

``baseURL: `${BACKEND_SERVICE_URL}/api` ``, `timeout: 30000`, the `x-microservice-key` header,

plus the `AuthInterceptor` / `CaCertService` `onModuleInit` block that lillit's existing

`DeviceManagerClientModule` already uses.

  

Add `getCameraSensorState(los)` to the device-manager client and to the new backend client,

each wrapping failures in the existing

[`DeviceManagerError`](apps/lillit/src/errors/device-manager.error.ts) shape / a matching

`BackendError`, logged via the structured

[`DeviceManagerErrorLog`](apps/lillit/src/types/device-manager-error-log.ts) pattern.

  

**New folder:** `apps/lillit/src/sensors/`

  

- `sensors.controller.ts`

  

  ```ts

  @ApiTags('sensors')

  @MicroServiceController('sensors')

  export class SensorsController {

    @Get('state')

    @ApiOperation({ summary: 'Get the current state of all sensors' })

    @ApiHeader({ name: 'x-microservice-key', required: true })

    @ApiResponse({ status: 200, type: SensorsStateResponseDto })

    async getSensorsState(): Promise<SensorsStateResponseDto>

  }

  ```

  

  With `app.setGlobalPrefix('api/v1')` in [main.ts](apps/lillit/src/main.ts#L81) this resolves to

  **`GET /api/v1/sensors/state`**. `@MicroServiceController` supplies

  `MicroServiceKeyGuard` + `PublicRoute`, matching `MissionController`.

  

- `sensors.service.ts` — sequential, because the second call depends on the first:

  

  1. `deviceManagerClient.getLiveState()` → live pan/tilt/hFOV. Throw on failure.

  2. Build the `los` string from step 1, then `backendClient.getCameraSensorState(los)`.

     Throw on failure.

  3. Merge via the mapper, wrap with `formatMissionTimestamp(new Date())`.

  

  Per the agreed decision, either upstream failure fails the whole request. Map both to a

  `BadGatewayException` so the caller gets a `502` with a clear message rather than a `500`.

  

- `sensor-state.mapper.ts` — a pure function

  `toSensorState(backend: CameraSensorStateDto, live?: SensorLiveStateDto): SensorStateDto`.

  Keeping this pure and free of I/O is what makes the omit/null rules cheap to unit-test

  exhaustively.

  

- `sensors.module.ts` — imports `DeviceManagerClientModule` + `BackendClientModule`; register in

  [app.module.ts](apps/lillit/src/app.module.ts).

  

**DTOs** go in `libs/backend-dto/src/lillit/dto/` alongside

`take-sensor-control-request.dto.ts`, exported through that folder's `index.ts`. Match the

established house style: **classes with `@ApiProperty` + class-validator decorators** (lillit

uses class-validator, not Zod), TS enums for `SensorType` and `SensorStatus`, and every optional

field marked `@ApiPropertyOptional` + `@IsOptional`.

  

---

  

## Config

  

Add to `.env.example` (`BACKEND_SERVICE_URL` already exists at line 20 — confirm lillit reads it):

  

```

# consumed by apps/lillit

BACKEND_SERVICE_URL="http://localhost:3000"

```

  

Per `CLAUDE.md`'s env-var workflow, also wire it into the Tilt config.

**Note a pre-existing gap:** `tiltfiles/platform/env/Lillit.Tiltfile` and

`tiltfiles/lillit/EnvVars.Tiltfile` currently inject *only* OTel variables — neither

`DEVICE_MANAGER_SERVICE_URL` nor `MICROSERVICE_KEY` is set there, and

`scripts/in-a-box/docker-compose-dev.yml` omits them for `lillit` too. The new

`BACKEND_SERVICE_URL` must be added, and the two existing omissions should be fixed at the same

time or the endpoint will not work under Tilt.

  

---

  

## Tests

  

Naming per `CLAUDE.md`: `*.unit-spec.ts` / `*.module-spec.ts` (plain `*.spec.ts` is ESLint-banned).

  

| Test | Covers |

|---|---|

| `sensor-state.mapper.unit-spec.ts` (lillit) | The omit/null matrix: optional keys absent when unknown, `stationID` explicitly `null`, `coverage` omitted wholesale when no live entry, `hfov_deg` omitted independently of pan/tilt |

| `sensors.service.unit-spec.ts` (lillit) | Merge by `sensorID`; a camera present in backend but absent from device-manager yields no `coverage`; either client throwing produces `BadGatewayException` |

| `sensors.controller.module-spec.ts` (lillit) | Route resolves at `/api/v1/sensors/state`; 401 without `x-microservice-key`; response shape and the `DDMMYYYY hh:mm:ss` timestamp regex |

| `mission-timestamp.util.unit-spec.ts` (lillit) | **Currently missing entirely.** The format is now load-bearing for two endpoints — add UTC/padding/month-offset coverage |

| `sensors.service.unit-spec.ts` (device-manager) | Non-camera clients filtered out; mils→degrees for LOS and hFOV; `null` when pedestal/north-ref/FOV absent |

| `cameras.service.unit-spec.ts` (backend, extend) | `BUSY` beats raw status; missing `cameras_connections` row → `INACTIVE`; station resolved via active spotter; `maxRange_m` omitted when `getVisibleRangeKm` returns `null` |

  

Run: `turbo test:unit --filter=lillit --filter=backend --filter=device-manager-service`

and `npm run typecheck`.

  

> **Heads-up, pre-existing:** `mission.service.unit-spec.ts` and

> `mission.controller.module-spec.ts` are stale — they assert the old

> `POST /mission/take-sensor-control` contract and old DTO field names, and would fail

> `typecheck:spec` against today's source. Out of scope here, but it will surface when the suite

> is run; worth a separate ticket.

  

---

  

## Verification

  

1. `tilt up -- --yanai`, then trigger `redis`, `unleash`, `mapproxy`, `geoserver`

   (per the local-startup runbook — `geoserver` is required for viewshed/`maxRange_m`).

2. Seed cameras (the existing Lillit seed script from commit `2657ba5c8`).

3. Hit the new upstreams directly first, to isolate failures:

   ```bash

   curl -H "x-microservice-key: $MICROSERVICE_KEY" \

     http://localhost:3060/api/device-manager/sensors/live-state

   curl -H "x-microservice-key: $MICROSERVICE_KEY" \

     "http://localhost:3000/api/cameras/sensor-state?los=<uuid>:180"

   ```

4. Hit the real endpoint:

   ```bash

   curl -H "x-microservice-key: $MICROSERVICE_KEY" \

     http://localhost:3080/api/v1/sensors/state | jq

   ```

   Assert: `timestamp` matches `^\d{8} \d{2}:\d{2}:\d{2}$`; every sensor has

   `sensorID` / `sensorType` / `status` / `stationID`; no `batteryPercentage` or `speed` keys

   appear anywhere.

5. **Live-state path** (needs a SOAP-vendor camera): use `onvif-simulator` or a `FAKE`-vendor

   camera, take control via `POST /api/v1/safepass/missions`, then re-hit `/sensors/state` and

   confirm `coverage` appears with plausible `pan_deg` (0–360) and `tilt_deg`.

6. **`BUSY` path:** with control held, confirm that camera reports `status: "BUSY"` and a

   non-null `stationID` (the spotter's station); release control and confirm it reverts.

7. **Degradation:** stop `geoserver`, confirm `maxRange_m` disappears but the request still

   returns `200`. Stop `device-manager`, confirm the endpoint returns `502` (the agreed

   fail-whole-request behaviour) rather than a partial payload.

8. Swagger: `http://localhost:3080/api/docs` — confirm the `sensors` tag documents the route

   with correct optional-field markings.
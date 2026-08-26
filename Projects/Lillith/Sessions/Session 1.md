## Part 1 — How camera control actually works today, end to end

The system has three services in the loop:

- **[apps/frontend](vscode-webview://0duokdd0ugkicnq5cs5u01psunduuhufjpks5u9prqio0fbphvid/apps/frontend)** — the operator's ("spotter's") browser.
- **[apps/backend](vscode-webview://0duokdd0ugkicnq5cs5u01psunduuhufjpks5u9prqio0fbphvid/apps/backend)** — owns Postgres (camera catalog, `CameraConn` rows, users), and is the source of truth for who's supposed to control what.
- **[apps/device-manager-service](vscode-webview://0duokdd0ugkicnq5cs5u01psunduuhufjpks5u9prqio0fbphvid/apps/device-manager-service)** — the only service that holds live TCP/ONVIF-SOAP connections to the actual camera hardware, plus an in-memory + Redis-backed ownership map.

There are two parallel entry points that do the _same_ thing — a REST call and a WebSocket event — because the frontend needs both a one-shot "take control" call and a persistent socket for live commands/telemetry.

### Step 0 — who's allowed to call this at all

This matters a lot for part 2, so I checked it carefully:

- [device-manager.controller.ts:50-52](vscode-webview://0duokdd0ugkicnq5cs5u01psunduuhufjpks5u9prqio0fbphvid/apps/device-manager-service/src/device-manager/device-manager.controller.ts:50) is decorated with `@MicroServiceController('')`, which expands to `@UseGuards(MicroServiceKeyGuard)` + `@PublicRoute()` ([microservice-controller.decorator.ts](vscode-webview://0duokdd0ugkicnq5cs5u01psunduuhufjpks5u9prqio0fbphvid/libs/backend-common-src/nest-core/src/decorators/microservice-controller.decorator.ts)). `PublicRoute()` means Nest's normal per-user JWT guard is **skipped entirely** for this controller. The only thing device-manager-service itself checks on an incoming HTTP request is a static shared secret in the `x-microservice-key` header, MD5-hashed and compared to `HASHED_MICROSERVICE_KEY` ([microservice-key.guard.ts](vscode-webview://0duokdd0ugkicnq5cs5u01psunduuhufjpks5u9prqio0fbphvid/libs/backend-common-src/nest-core/src/guards/microservice-key.guard.ts:27)).
- The frontend's own axios client (`deviceManagerApiClient`) does **not** attach that key — it attaches the user's `Authorization: Bearer <jwt>` instead ([createAxiosInstance.ts:22](vscode-webview://0duokdd0ugkicnq5cs5u01psunduuhufjpks5u9prqio0fbphvid/libs/frontend-shared/src/utils/createAxiosInstance.ts:22)). So real end-user JWT verification for the REST path happens _upstream_ of this Nest app — in the private API Gateway sitting in front of it (`terraform/device-manager/private-rest-api.tf`, VPC-endpoint-only, cross-account). By the time a request reaches `DeviceManagerController`, it is trusted, and whatever `userId` is in the JSON body is taken at face value.
- The **WebSocket** path is stronger: `WebsocketGateway.handleConnection` runs `AuthJwtGuard.wsAuthenticate(client)` ([websocket.gateway.ts:184-241](vscode-webview://0duokdd0ugkicnq5cs5u01psunduuhufjpks5u9prqio0fbphvid/libs/backend-common-src/ws/src/websocket.gateway.ts:184)), which verifies a real JWT and populates `client.data.user.id`. Every WS handler then reads the id from the verified session (`getSocketSpotterId`), never from the message payload.

So: **REST = trusts the body's `userId`, gated by a shared internal secret. WebSocket = derives the id from a verified JWT session.** Keep that asymmetry in mind for part 2.

### Step 1 — the take-control request

Frontend calls `POST /:cameraId/command/connect` with `ConnectCommandDto { userId, isFakeVendor, firstName, lastName, cameraId }` ([cameras.http-protocol.ts:26-36](vscode-webview://0duokdd0ugkicnq5cs5u01psunduuhufjpks5u9prqio0fbphvid/apps/frontend/src/api/cameras/cameras.http-protocol.ts:26)), or emits `CAMERA_CONNECT` over the socket with the same shape minus identity ([device-manager.gateway.ts:186-222](vscode-webview://0duokdd0ugkicnq5cs5u01psunduuhufjpks5u9prqio0fbphvid/apps/device-manager-service/src/device-manager/device-manager.gateway.ts:186)).

Controller validates `cameraId`/`userId` are present, calls `deviceManagerService.cameraToControl(...)`, then validates the _response_ shape against a JSON Schema (`CameraTakeControlDtoSchema`) before returning 201 — a guard against a malformed downstream camera-driver response leaking out ([device-manager.controller.ts:530-559](vscode-webview://0duokdd0ugkicnq5cs5u01psunduuhufjpks5u9prqio0fbphvid/apps/device-manager-service/src/device-manager/device-manager.controller.ts:530)).

### Step 2 — `cameraToControl` ([device-manager.service.ts:1257-1302](vscode-webview://0duokdd0ugkicnq5cs5u01psunduuhufjpks5u9prqio0fbphvid/apps/device-manager-service/src/device-manager/device-manager.service.ts:1257))

1. **Per-camera mutex.** Acquires `camerasConnectionsMutexes.get(cameraId)` so two simultaneous take-control attempts on the same camera can't race.
2. **`controlCamera`** checks `getActiveUserIdByCameraId(cameraId)` — a reverse scan of the in-memory ownership map. If someone else already holds it, throws `400 Camera already controlled by <id>`. This is pure first-come-first-served; there's no request/approval queue.
3. Notes if _this_ userId already held a _different_ camera (`prevControlledCameraId`) — a spotter can only actively hold one camera.
4. **`updateControllingSpotter(userId, cameraId)`** does five things:
    - Writes the new owner to Postgres via `backendClient.updateControllingSpotter` (durable source of truth, `CameraConn.activeSpotterId`).
    - Updates the local in-process `Map<userId, cameraId>` (L1 cache).
    - Fire-and-forget writes to Redis (`cameraOwnershipRedis.set`), then does `websocketGateway.serverSideEmit(INVALIDATION_EVENT, userId)` — because device-manager-service runs multiple replicas behind an NLB, and each pod has its _own_ L1 map. The Socket.IO server-side broadcast tells every other replica to evict that userId from its local cache so a stale read never wins a race during a rolling deploy.
    - Tells backend to disconnect this spotter from any drone they might currently be piloting (`disconnectFromDrone`) — camera and drone control are mutually exclusive per spotter.
    - Writes an audit row (`handleConnectionOpLog`, `CONNECT_TO_CAMERA`).
5. If there was a previous camera, clears its command mutexes/lasing state and stops its status broadcast, returning a `cameraDisconnected` payload.
6. **`getConnectionData`** builds the sensor payload returned to the caller: real optical/pedestal/status data pulled live from the ONVIF camera client, or a hardcoded fake payload if `isFakeVendor` (used for demo/simulated cameras like the ones in your recent Lillit commits).
7. **`startStatusBroadcasting`** kicks off a periodic push of live telemetry to the WebSocket room `user:<userId>` (skipped for fake vendor).
8. Releases the mutex, returns `{ cameraDisconnected, cameraConnected, connData }`.

### Step 3 — every subsequent command

Move/zoom/focus/lase/calibrate/FOV/channel-switch/power/north/go-to/polarity all funnel through `validateUserCameraControl(cameraId, userId)` first ([device-manager.service.ts:1110-1121](vscode-webview://0duokdd0ugkicnq5cs5u01psunduuhufjpks5u9prqio0fbphvid/apps/device-manager-service/src/device-manager/device-manager.service.ts:1110)):

```
cameraControlId = userIdToCameraControlId.get(userId)  // falls back to Redis on L1 miss
if (!cameraControlId || cameraControlId !== cameraId) throw NotFoundException
```

That's the _entire_ authorization check for every camera movement — does this opaque `userId` string currently own this `cameraId` in the map. Nothing about roles, nothing about hardware permissions.

### Step 4 — release

Either an explicit `POST /:cameraId/command/disconnect` / WS `CAMERA_DISCONNECT`, or an implicit steal (whoever calls take-control first while unowned wins), or a `switchControllingSpotter` (shift-change: re-point ownership from one id to another atomically without dropping the physical camera connection).

---

## Part 2 — assigning control to a pre-registered computer instead of a logged-in user

The key fact from Part 1 that makes this tractable: **`userId` in the ownership model is just an opaque string key.** `userIdToCameraControlId`, `validateUserCameraControl`, `updateControllingSpotter`, `switchControllingSpotter`, `removeControllingSpotter` — none of them care whether that string corresponds to a real interactive session. They just compare strings and read/write a map + a Postgres column + Redis. Nothing _forces_ it to be a logged-in user today.

That means the mechanical part of what you're describing is small. The part that actually needs design is the **authorization decision** — deciding _who_ is allowed to hand a live PTZ/laser-capable camera to a given `computerId`, since device-manager-service itself currently enforces almost nothing beyond "did the caller have the shared microservice key."

### Recommended shape

**1. Treat `computerId` as just another controller id, but make the DTO honest about it.** Add a discriminator rather than silently overloading `userId`/`firstName`/`lastName` with fake values for a machine:

```ts
export class MissionConnectCommandDto extends CommandBaseDto {
  // userId here = the pre-assigned computerId
  displayName: string;      // replaces firstName/lastName — a computer has no name
  missionId?: string;
  isFakeVendor: boolean;
  cameraId: string;
}
```

Reuse `CommandBaseDto.userId` as the slot that holds the `computerId` — every downstream function already keys off that field name, so you don't have to touch `validateUserCameraControl`, the Redis ownership service, or any of the per-command handlers.

**2. New endpoint, thin wrapper around existing internals.** `POST /:cameraId/command/mission-connect` in [device-manager.controller.ts](vscode-webview://0duokdd0ugkicnq5cs5u01psunduuhufjpks5u9prqio0fbphvid/apps/device-manager-service/src/device-manager/device-manager.controller.ts), calling something like:

```ts
async cameraToMissionControl(cameraId, computerId, displayName, isFakeVendor) {
  return this.controlCamera(cameraId, computerId, displayName, /* no lastName */ '');
  // ...same updateControllingSpotter / getConnectionData / startStatusBroadcasting flow
}
```

You don't need to fork `cameraToControl` — just give it a `displayName` instead of `firstName`+`lastName` so `activeSpotterDisplayName` in the response isn't `"undefined undefined"`.

**3. Who's allowed to call it — this is the real question.** Since "not logged in" means there's no JWT to check, the caller of this new endpoint won't be the browser — it'll be another backend service acting on the mission's behalf (whatever owns your mission/orchestration logic). That caller reaches device-manager-service the same way any internal service does today: inside the private VPC, carrying the shared microservice key. Device-manager-service doesn't need a new auth mechanism for _that_ — it's the same trust boundary every other endpoint here already has.

What's missing is the layer **above** device-manager-service: something has to verify that (a) the human/process requesting "give camera X to computer Y" is actually authorized to do that, and (b) `computerId` is a real pre-registered station and not an arbitrary string. Device-manager-service has no concept of a "Computer" entity today (I checked — no such model exists yet), so that registry and its authorization check need to live wherever your mission logic does (`apps/backend` or a new service), _not_ be assumed away here. Right now, anyone who can reach this endpoint with the shared key can hijack any camera by supplying any `computerId` string — that's fine as long as the calling service is the only thing that can reach it and it does real authorization before calling in, but it's worth being explicit about, since it's a live PTZ/laser device.

**4. Telemetry after control is granted.** `startStatusBroadcasting` pushes to WS room `user:<id>`. A computer that never opens an authenticated socket under that id won't receive those pushes — they'll just silently go nowhere (no error). If the computer only needs to _send_ commands (move/goTo/etc.), that's fine as-is since those are plain REST calls gated the same way. If it also needs live sensor status, you'd need either a pull-based status endpoint, or a machine-token variant of `AuthJwtGuard.wsAuthenticate` that lets a service identity open a socket resolving to `client.data.user.id === computerId`. 

**5. Release/handoff is already free.** `switchControllingSpotter(humanId, computerId)` and `removeControllingSpotter(computerId)` work unchanged — they're already generic over the id type, so handing control from a person to a pre-assigned computer (or back) needs no new code.

---

Want me to actually implement this — new DTO, controller endpoint, and service method — or do you want to first pin down where the "Computer" registry and its authorization check will live (backend vs. a new mission service) before I touch device-manager-service?

Good question — there's actually a terminology trap here worth clearing up first: **there are two completely separate "connections" in this system, and the WebSocket is only one of them.**

- **Connection A**: the operator's browser ↔ `device-manager-service` (this is the Socket.IO WebSocket — the "websockets part" you're asking about).
- **Connection B**: `device-manager-service` ↔ the physical camera hardware (ONVIF/SOAP over raw TCP, handled by `CameraDeviceClient`, nothing to do with sockets or browsers at all).

The WebSocket never "connects to a computer" in the sense of dialing out to a station — it's the reverse: a client (browser, or in your mission-request idea, a machine) dials **in** to device-manager-service, and the server has to figure out **who just connected** and **which camera that identity currently owns**.

### How the socket is authenticated and identified

When a client opens the socket, it doesn't send a token as a header or query param — Socket.IO has a dedicated `auth` payload sent at handshake time, and that's what this app uses:

```ts
protected getWsToken(socket: Socket) {
  const token = socket.handshake.auth.token;   // client passed this in io(url, { auth: { token } })
  const aadToken = socket.handshake.auth.aadToken;
  return { token, aadToken };
}
```

([auth.guard.ts:174-180](vscode-webview://0duokdd0ugkicnq5cs5u01psunduuhufjpks5u9prqio0fbphvid/libs/backend-common-src/nest-core/src/guards/auth.guard.ts:174))

`WebsocketGateway.handleConnection` runs on every new socket ([websocket.gateway.ts:184](vscode-webview://0duokdd0ugkicnq5cs5u01psunduuhufjpks5u9prqio0fbphvid/libs/backend-common-src/ws/src/websocket.gateway.ts:184)):

```ts
async handleConnection(client: Socket) {
  const isAuthenticated = await this.onUserLogin(client);   // -> auth.wsAuthenticate(client)
  if (!isAuthenticated) return;

  const userId = client.data['user']?.id;
  if (userId) {
    this.trackConnectedSocket(client.id, userId);
    void client.join(`user:${userId}`);        // <-- this is the key line
  }
  ...
}
```

`wsAuthenticate` verifies the JWT the same way the REST auth guard would, and — critically — **stamps the verified identity onto the socket itself**: `socket.data['user'] = userData`. From that point on, this specific TCP/WebSocket connection _is_ that user, for its entire lifetime. There's no per-message re-authentication; the server just trusts `client.data.user` because it was set once, at handshake, from a verified token.

Then `client.join('user:' + userId)` puts the socket into a Socket.IO **room** named after that user's id. Rooms are just Socket.IO's built-in multicast grouping — a socket can be in many rooms, a room can have many sockets (e.g. the same user logged in from two tabs — that's exactly what `connectedUserSocketCounts` tracks, supporting multiple simultaneous sockets per user).

This room is how the server pushes things _to_ a specific client later, without needing to remember a raw socket reference:

```ts
public emitToSpecificUser(clientOrUserId: Socket | string, eventName: string, message: any) {
  ...
  this.server.to(`user:${clientOrUserId}`).emit(eventName, message);
}
```

([websocket.gateway.ts:397-417](vscode-webview://0duokdd0ugkicnq5cs5u01psunduuhufjpks5u9prqio0fbphvid/libs/backend-common-src/ws/src/websocket.gateway.ts:397))

So "which computer does it send data to" is answered by: _whichever sockets are currently joined to room `user:<id>`_ — could be zero, one, or several.

### How it knows which _camera_ an incoming event is about

This is the part that's easy to miss: **WS event payloads (move, zoom, focus...) don't carry a `cameraId` at all.** Look at [device-manager.gateway.ts:283-308](vscode-webview://0duokdd0ugkicnq5cs5u01psunduuhufjpks5u9prqio0fbphvid/apps/device-manager-service/src/device-manager/device-manager.gateway.ts:283):

```ts
private async cameraStartMoving(cameraMoveData: MoveCommandDto, callback, client: Socket) {
  const spotterId = this.getSocketSpotterId(client);        // from client.data.user.id
  const cameraId = await this.getUserControlledCameraId(client);   // <-- derived, not sent
  ...
  await this.deviceManagerService.startMoving(cameraId, spotterId, ...);
}
```

`getUserControlledCameraId` does:

```ts
private async getUserControlledCameraId(client: Socket): Promise<string> {
  const spotterId = this.getSocketSpotterId(client);
  const cameraId = await this.deviceManagerService.getControlledCameraIdByUserId(spotterId);
  if (!cameraId) throw new Error(`Camera not found for user ${spotterId}`);
  return cameraId;
}
```

([device-manager.gateway.ts:270-281](vscode-webview://0duokdd0ugkicnq5cs5u01psunduuhufjpks5u9prqio0fbphvid/apps/device-manager-service/src/device-manager/device-manager.gateway.ts:270))

So the sequence for, say, a "move camera" command is:

1. Socket arrives with `client.data.user.id` already set (from handshake auth — happened once, way earlier).
2. Server looks that id up in `userIdToCameraControlId` (the same in-memory/Redis ownership map from Part 1's `cameraToControl` flow) — this is the map that got populated when the user _took control_ of a camera in the first place.
3. Whatever `cameraId` comes back from that lookup is the camera the move command gets applied to.

In other words: **the camera identity is entirely a side effect of "which user does this socket belong to, and which camera did that user last successfully take control of."** There is no explicit "connect this socket to camera X" step for ongoing commands — only the initial `CAMERA_CONNECT` event carries a `cameraId` (that's the one call that's allowed to establish new ownership; [device-manager.gateway.ts:186-222](vscode-webview://0duokdd0ugkicnq5cs5u01psunduuhufjpks5u9prqio0fbphvid/apps/device-manager-service/src/device-manager/device-manager.gateway.ts:186)). Every command after that rides on the ownership map, keyed purely by the socket's authenticated user id.

That's also why the exact same `validateUserCameraControl(cameraId, userId)` check from the REST path reappears here in spirit — except over WS the `userId` can't be spoofed by the payload (it's pinned to the verified socket), whereas over REST it's just whatever string the request body says.

### Tying this back to your "computer, not logged in" idea

If a pre-registered computer wanted to receive live pushes (sensor status, `CAMERA_CONNECTED`/`CAMERA_DISCONNECTED` events) the same way a browser does, it would need to do exactly what a browser does: open its own Socket.IO connection and pass **something** in `handshake.auth.token` that `wsAuthenticate` can verify down to `client.data.user.id === computerId`. That's a machine-credential problem (e.g. a client-credentials/service JWT minted for that computer) rather than a routing problem — the routing (rooms, `getUserControlledCameraId`) already works generically off whatever id ends up on `client.data.user.id`, human or machine, with zero changes needed. The only piece that doesn't exist yet is a way to authenticate a non-interactive computer at the socket handshake in the first place.

I dug into the actual JWT verification code that `wsAuthenticate` relies on, and it changes the picture: **you can't just point lillit's socket at device-manager-service with an arbitrary token today — the auth guard hard-requires a human-shaped payload.**

### What's actually enforced today

`AuthJwtGuard.getUserData` ([auth.guard.ts:28-54](vscode-webview://0duokdd0ugkicnq5cs5u01psunduuhufjpks5u9prqio0fbphvid/libs/backend-common-src/nest-core/src/guards/auth.guard.ts:28)) runs on every verified token, WS or REST, and throws unless the payload has:

- `username`
- `role` — must map to an existing `UserRolesEnum` Prisma value (a human role)
- `firstName`, `lastName`, `hamalId` — all required, or it 401s with `"Token is missing required user claims"`

So there is currently no way for a non-human "computer" identity to pass through `wsAuthenticate` at all — not because of a missing credential, but because the shape of what counts as a valid identity is baked into this shared guard (`libs/backend-common-src`), used by **every** service in the monorepo, not just device-manager-service. That's the real blocker, more than "how do I open a socket."

### The flow you described, concretely

```
Computer B  --HTTP-->  lillit  --Socket.IO client-->  device-manager-service
                                    (authenticated AS computerId A)
```

Lillit needs to act as a **Socket.IO client** (not a server) here — that's a new dependency (`socket.io-client`) and a new piece of code in `apps/lillit`, since right now lillit is pure scaffolding with no logic. Mechanically, once it has _some_ token that `wsAuthenticate` will accept:

```ts
import { io, Socket } from 'socket.io-client';
import { DEVICE_MANAGER_WS_PATH } from '@libraries/backend-dto';

const socket: Socket = io(deviceManagerWsUrl, {
  path: DEVICE_MANAGER_WS_PATH,
  auth: { token: machineTokenForComputerA },
  transports: ['websocket'],
});
```

That's it on the wiring side — once the handshake succeeds, `client.data.user.id` becomes A's id, the gateway joins it to room `user:A`, and every mechanism I described last message (ownership map, `getUserControlledCameraId`, status broadcast) works completely unchanged, because none of it cares whether the socket "belongs to" a browser or a NestJS process. Worth noting: losing this socket does **not** auto-release camera ownership — nothing in device-manager-service registers a disconnect callback that revokes control, so a lillit reconnect/restart wouldn't kick A off the camera, just interrupt live telemetry push until it reconnects.

### The actual decision: how does that token get minted?

**Option 1 — extend the real identity model.** Add a machine-appropriate `UserRolesEnum` value (e.g. `COMPUTER`/`STATION`) in the Prisma schema, have your auth-service issue a properly-signed RS256 token for a registered computer record (with placeholder or real `firstName`/`lastName`/`hamalId`), and lillit fetches that token before connecting. Pro: zero special-casing anywhere downstream — device-manager-service, rooms, ownership, `emitToSpecificUser`, all just work as-is, and you get a real auditable machine identity. Con: touches a shared library used by every microservice's auth, plus the Prisma schema and auth-service — bigger blast radius, needs careful review per your CLAUDE.md rules on shared-lib and migration changes.

**Option 2 — a parallel, narrower machine-auth path.** Add a second WS authentication branch (only reachable by a caller holding a shared secret, mirroring the existing `MicroServiceKeyGuard` REST pattern) that skips `JWT.verify`/`getUserData` entirely and directly sets `client.data.user = { id: computerId, role: 'COMPUTER' }` when lillit connects. Pro: fully isolated to device-manager-service (or a tiny shared helper), no schema/shared-guard changes, much smaller diff. Con: weaker identity guarantee — it proves "this is lillit" (holder of a shared secret), not cryptographically "this is specifically computer A" — same shape of trust the REST `x-microservice-key` already has today, so it's consistent with an existing pattern rather than a new weaker one, but it's still worth being deliberate about since this is the thing granting live PTZ/laser control.

I'd lean toward **Option 2** for a first cut — it's contained, matches the trust model your REST endpoints already use, and doesn't require touching shared auth code that every other service depends on. Option 1 is the "more correct" long-term shape if computers need real per-identity accountability (audit trails showing _which_ station, not just "lillit vouched for someone").

Want me to implement Option 2 — the new WS auth branch in device-manager-service plus the lillit-side socket client — or do you want to go the Option 1 route instead?

Right now nothing is actually pegging the CPU (sampled live over 2s — top consumer was VS Code itself at under 1%), but there are a few things worth knowing about:

**VS Code itself is heavy:** 21 `Code` processes totaling ~3.5GB RAM. That's normal if you have multiple windows/workspaces or a lot of extensions running their own extension-host processes, but it's worth checking `Help > Open Process Explorer` inside VS Code to see if one extension host is bloated.

**Non-coding background apps eating resources:**

- `BlueStacksServices` (Android emulator) — two service processes, ~1.6GB cumulative CPU time between them, running even though presumably not in active use.
- `NahimicSvc64`/`NahimicSvc32` (audio-enhancement software bundled with some laptops) — a well-known culprit for random CPU spikes and input lag on Windows; often safe to disable if you don't use its audio profiles.
- `Microsoft.CmdPal.UI` (Windows Command Palette/PowerToys) — high cumulative CPU (that's total time since launch, not a current spike, but still notable).
- `GoogleDriveFS` — actively syncing, ~180MB.

**The one most likely to actually matter for VS Code specifically:** Windows Defender (`MsMpEng`, 324MB resident). I couldn't check your exclusion list without admin rights, but real-time scanning of a large monorepo like this one — lots of small files, `node_modules`, `.git`, Turbo/Nest build output — is one of the most common causes of sluggish file search, git status, and TypeScript language-server responsiveness on Windows. If this repo's root and `node_modules` aren't already excluded from real-time protection, that's worth adding (Windows Security → Virus & threat protection → Exclusions) — I can't add it myself without elevated permissions, and it's the kind of system-wide change you'd want to do yourself anyway.

**Nothing project-related is currently running** — no `docker`, `tilt`, or lingering `turbo`/`nest` watch processes were found, just 2 stray `node` processes (~184MB combined) and no active dev environment. So if VS Code feels slow right now, it's not this repo's tooling causing it — it's most likely the Defender scanning + the sheer number of open Code windows/extension hosts.
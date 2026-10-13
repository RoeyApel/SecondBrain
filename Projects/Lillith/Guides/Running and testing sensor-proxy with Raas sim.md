### Testing the Lillith API locally

**Before the first run**

1. In the RaaS repo, check out the version the proxy’s SDK uses (currently `dev.88`), then build:
    
    ```
    git checkout v1.0.0-dev.88 && pnpm install && pnpm build:types && pnpm --filter @100-platform/raas-shared-core build
    ```
    
2. Still in the RaaS repo, run `pnpm certs:sim`. In apps/server/.env, set `CERTIFICATES_PATH` to the full path of `apps/simulator/certs`.
3. In apps/proxy/.env, set `BASE_URL=http://localhost:3000`, `API_KEY=api-key-example`, and `CALLBACK_URL=http://localhost:<PORT>`, where `<PORT>` is the proxy’s own port (not 3000).
4. In apps/proxy/config/assets.yml, add the drones (don’t commit this file):
    
    ```
    drones:  - { type: EYESATOP, hostname: localhost:8543, serial: SIM-001, name: sim-alpha }
    ```
    
    For a quick test without the simulator, use `{ type: TEST, hostname: 10.0.0.5:9000, name: drone-1 }` instead.
    

**Run**, each in its own terminal:

1. `docker start proxy-redis`
2. RaaS repo: `pnpm dev:server`
3. RaaS repo: `pnpm dev:sim`. Then start the drones:
    
    ```
    curl -k -X POST https://localhost:8443/admin/instances/start-all
    ```
    
4. Proxy (`apps/proxy`): `$env:LOG_LEVELS='log,warn,error,debug'; pnpm start`. Copy each drone’s id from the `Registered drone successfully ... (id: ...)` log line.

**Test** at `http://localhost:<PORT>/api/docs` (Swagger), or with curl:

```
curl -X POST http://localhost:<PORT>/api/v1/safepass/missions -H "Content-Type: application/json" -d '{"hamalID":"Hamal_A","stationID":"Station_A","assetID":"<ID>","timestamp":"2026-10-13T10:00:00.000Z","missionType":"Pair"}'
```

```
curl -X POST "http://localhost:<PORT>/api/v1/safepass/assets/<ID>/commands?command=move" -H "Content-Type: application/json" -d '{"hamalID":"Hamal_A","stationID":"Station_A","timestamp":"2026-10-13T10:00:00.000Z","pan":30,"tilt":-10,"zoom":2}'
```

**What to look for**

- Responses are `200` with `SUCCESS`, or `FAILED` plus a reason.
- The simulator log shows `Gimbal rotated` and `Camera zoom` for each command that went through.
- The proxy log shows `Received drone telemetry update` about once a second per drone.

**Unit tests** (no other programs needed):

```
pnpm --filter @platform/sensor-proxy test lillith
```
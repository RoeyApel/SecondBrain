## Overview

Three processes run side by side: Redis in Docker, the RaaS server, and sensor-proxy (which also serves the lillith API). Start them in that order.

|   |   |   |   |
|---|---|---|---|
|Process|Repo|Port|Started with|
|Redis|(Docker image `redis:7-alpine`)|6379|Docker Desktop|
|RaaS server|`100-platform-raas`|3001|`pnpm dev:server`|
|sensor-proxy + lillith API|`100-platform-sensor-proxy`|4000|`pnpm start`|

Prerequisites: Node.js 22+, Docker Desktop, pnpm through corepack, and both repos cloned under `C:\Users\roiap\repos`.

## One-time setup

Do these once per machine, and again whenever the proxy's RaaS SDK version changes.

1. Make pnpm 10 the global default. RaaS has no pinned pnpm version, and corepack's default (pnpm 12) fails on Windows.
    - `corepack install -g pnpm@10.11.0`
    - Check: `pnpm --version` prints `10.11.0`.
2. Put RaaS on the same version as the proxy's SDK. The proxy uses `@100-platform/raas-sdk` 1.0.0-dev.81; find the current one in `apps/proxy/package.json`.
    - `git -C C:/Users/roiap/repos/100-platform-raas checkout v1.0.0-dev.81`
3. Install and build RaaS. From `100-platform-raas`:
    - `pnpm install`
    - `pnpm build` (builds types, shared-core and the SDK; `build:types` alone is not enough)
4. Set the RaaS port. In `100-platform-raas/apps/server/.env`:
    - `RAAS_PORT=3001`
    - Keep `NODE_ENV=development`, which lets RaaS accept a `localhost` callback URL.
    - Note `RAAS_API_KEY` (default `api-key-example`); the proxy needs the same value.
5. Create the proxy's env file at `100-platform-sensor-proxy/apps/proxy/.env` (not the repo root). Copy `deploy/secrets/proxy.env.example`, then set:

```
PORT=4000
BASE_URL=http://localhost:3001
API_KEY=api-key-example
CALLBACK_URL=http://localhost:4000
REDIS_HOST=localhost
REDIS_PORT=6379
```

- `API_KEY` must equal `RAAS_API_KEY` in the RaaS `.env`.
- Leave `/api/raas` off `BASE_URL`; the SDK adds it.
- Leave `REDIS_PASSWORD` unset for the plain Redis container.
- `LORA_BRIDGE_URL` must exist but is only called when lora detections arrive.
- Leave `NODE_ENV` unset or `development` so Swagger is served.

6. Install the proxy's dependencies. From `100-platform-sensor-proxy`: `pnpm install`
7. Create the Redis container:
    - `docker run -d --name redis -p 6379:6379 redis:7-alpine`

## Every time you run it

Start Redis, then use two terminals, in this order.

1. Start Docker Desktop, wait until the engine is running, then start Redis:
    - `docker start redis`
2. Terminal 1, in `100-platform-raas`: `pnpm dev:server`
    - Wait for `RaaS Server running on port: 3001`.
3. Terminal 2, in `100-platform-sensor-proxy`: `pnpm start`
    - Wait for `sensor proxy is listening on port: 4000`.

To stop, press Ctrl+C in each terminal, then `docker stop redis`.

After pulling new commits: run `pnpm install` in both repos, and `pnpm build` in RaaS. If the proxy's SDK version changed, repeat one-time setup step 2 first.

## Checking it works

The quickest check is lillith's Swagger UI at http://localhost:4000/api/docs, where every route has a "Try it out" button.

|   |   |   |
|---|---|---|
|Check|URL or command|Expected result|
|Proxy health|http://localhost:4000/health|200|
|Sensor state|`GET http://localhost:4000/api/v1/sensors/state`|`{"timestamp":"DDMMYYYY HH:mm:ss","sensors":[]}`|
|Configuration sync|`GET http://localhost:4000/api/v1/configuration/sync`|Same shape as sensor state|
|Execute a mission|`POST http://localhost:4000/api/v1/safepass/missions`|201, `"status":"FAILED"`, `"reason":"SENSOR_UNAVAILABLE"`|
|Invalid mission body|Same POST with an unknown field or a date like `31022026 00:00:00`|400|

Empty sensors and FAILED missions are expected until lillith is connected to RaaS.

A valid mission body, in PowerShell (use `curl.exe`, not `curl`):

```
curl.exe -X POST http://localhost:4000/api/v1/safepass/missions -H "Content-Type: application/json" -d '{"timestamp":"20082026 00:00:00","missionID":"f6b8f98f-23b0-4c47-a19f-fc74eaee8fa5","missionType":"Pair","operatorContext":{"hamalID":"Hamal_A","stationID":"Station_A","sensorID":"5e5f4f2e-6d3a-4a1a-9f3e-4b8f3f2a9c11"},"position":{"latitude":1,"longitude":2,"altitude":3,"heading":4,"speed":5}}'
```

To test lillith alone, with no Redis, RaaS or `.env`: `pnpm --filter @platform/sensor-proxy test lillith`

## Troubleshooting

Every error below came up during setup; the fix is in the right-hand column.

|   |   |   |
|---|---|---|
|Error|Cause|Fix|
|`Could not run the pnpm binary at ...\pnpm\12.6.0\pnpm-native.exe`|RaaS pins no pnpm version, so corepack falls back to pnpm 12, which fails on Windows|`corepack install -g pnpm@10.11.0`|
|`Cannot find module '@100-platform/raas-shared-core'` (plus `auth.service.ts` type errors)|Only `raas-types` was built|`pnpm build` in the RaaS root, then restart `pnpm dev:server`|
|`failed to connect to the docker API at npipe:////./pipe/dockerDesktopLinuxEngine`|Docker Desktop is not running|Start Docker Desktop and wait for the engine|
|`Configuration key "LORA_BRIDGE_URL" does not exist`|The proxy can't find its `.env`. It reads `apps/proxy/.env`, not the repo root|Move the file to `apps/proxy/.env`|
|Port 3000 already in use|RaaS and the proxy both default to 3000|`RAAS_PORT=3001` in RaaS, `PORT=4000` in the proxy|
|Proxy fails to register or subscribe with RaaS|Wrong `BASE_URL` or `API_KEY`, or RaaS on a different version|Check `BASE_URL=http://localhost:3001`, matching API keys, and RaaS on the SDK's version tag|
|`docker start redis` says no such container|The container was created without `--name redis`|Start it from Docker Desktop, or remove it and re-run `docker run -d --name redis -p 6379:6379 redis:7-alpine`|
|`/api/docs` returns 404|`NODE_ENV=production` in the proxy `.env`|Unset it or set `development`|
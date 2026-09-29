# Port the lillith API into sensor-proxy (`apps/proxy/src/lillith-api/`)

## Context
The lillith SafePass-facing API (`100-platform-lilith`, branch `feat/create-lillith-service`, `apps/lillith`) moves into the sensor-proxy app. It will run inside the proxy's NestJS process rather than as a separate service. The lilith repo is left untouched.

The code is **fully aligned** to proxy conventions:
- NestJS 11, CommonJS, Jest, zod
- the proxy's root eslint/prettier and tsconfig

Lillith's feature modules are imported straight into the proxy's `AppModule`, with no wrapper module.

Work happens on branch `feat/add-lillith-api` (already checked out). The output is **local changes only**: no commit or push.

### Decisions (made by the user)
| Topic | Decision |
|---|---|
| Source | `feat/create-lillith-service` (stubbed RaaS). Plain copy, no git history. |
| Placement | `apps/proxy/src/lillith-api/`. Modules imported directly in `apps/proxy/src/app.module.ts`. |
| Routes | Keep `/api/v1/...` for lillith routes only: `GET /api/v1/configuration/sync`, `GET /api/v1/sensors/state`, `POST /api/v1/safepass/missions`. Proxy routes unchanged. |
| Health/metrics | Drop lillith's `/api/v1/health` and `/api/v1/metrics`. Use the proxy's `/health` and `/metrics`. |
| Request metric | Drop `http_requests_total` along with its interceptor and filter, and don't use `@willsoto/nestjs-prometheus`. |
| Logging | Drop pino. Use the Nest `Logger`, which goes to the proxy's OTEL bridge. |
| Auth | Drop `HASHED_MICROSERVICE_KEY`, the guard and the `MicroserviceController` decorator. Routes are unauthenticated for now. |
| Env/config | Drop lillith's env validation (`PORT`, `LOG_LEVEL`, `NODE_ENV` class). No new env vars. |
| Validation | Convert to zod. Request bodies use `.strict()`, which rejects unknown fields with 400, nested objects included. |
| Swagger | Keep it. Docs cover **lillith routes only**, at `/api/docs`, served only when `NODE_ENV !== 'production'`. That means local dev only: the Dockerfile and Helm values set production. |
| Swagger + zod | Use `nestjs-zod` 5.x, which is compatible with Nest 11, @nestjs/swagger 11 and zod 3.25. One zod schema produces both the validated DTO class and its docs. |
| RaaS | Keep the `TODO(raas)` stubs exactly as they are. |
| Tests | Jest, proxy naming (`src/tests/lillith-*.unit-test.ts`). The e2e test becomes a `*.component-test.ts`. |
| Docs | Add `lillith-api/README.md` only. |

## Target layout
```
apps/proxy/src/lillith-api/
  configuration/  configuration.{module,controller,service}.ts, index.ts
  sensors/        sensors.{module,controller,service}.ts, index.ts
  mission/        mission.{module,controller,service}.ts, index.ts
    handlers/     mission-handler.interface.ts, take-sensor-control-mission.handler.ts
  types/
    enums/        (the 9 enums, copied verbatim, same subfolders missions/ and sensors/)
    schemas/      zod schemas + createZodDto classes (replace dtos/)
    index.ts
  utils/safepass-timestamp.util.ts
  swagger.ts      setupLillithSwagger(app) + LILLITH_API_PREFIX / SWAGGER_PATH
  README.md
```

## Steps

### 1. Dependencies (`apps/proxy/package.json`)
- Add to `dependencies`: `@nestjs/swagger@^11` and `nestjs-zod@^5`.
- Add to `devDependencies`: `supertest` and `@types/supertest`.
- Update `pnpm-lock.yaml` with `pnpm install`. The private `@100-platform/*` packages are already installed; if the install needs `NPM_TOKEN`, ask the user.
- If nestjs-zod turns out not to work with zod 3.25 at implementation time, fall back to `zod-to-json-schema` / manual schema registration, and tell the user.

### 2. Copy and convert the code
**Imports:** drop the `.js` suffixes and use proxy style (`consistent-type-imports`, explicit return types).

**Enums** (`types/enums/**`): copy verbatim, including the Hebrew values.

**`utils/safepass-timestamp.util.ts`:** copy verbatim.

**DTOs to `types/schemas/`.** Each class-validator DTO becomes a zod schema plus `export class XDto extends createZodDto(XSchema) {}`, with each field's `.describe(...)` carrying the old `@ApiProperty` description.

- **Request, `ExecuteMissionRequestSchema`:**
  - `timestamp`: `z.string().regex(SAFEPASS_TIMESTAMP_REGEX, 'timestamp must match format DDMMYYYY HH:mm:ss')`
  - `missionID`: `z.string().uuid()`
  - `missionType`: `z.nativeEnum(MissionType)`
  - `operatorContext`: a strict object of `hamalID` / `stationID` (nativeEnum) and `sensorID` (uuid)
  - `position`: a strict object of `latitude`, `longitude`, `altitude`, `heading` and `speed`, all numbers
  - `.strict()` at every level
- **Responses:**
  - `ConfigurationSyncResponse` / `SensorConfig`
  - `SensorsStateResponse` / `SensorState` / `Position` / `Coverage` / `FootprintVertex`
  - `ExecuteMissionResponse`
  - These are used only for Swagger typing and not validated at runtime, as before.
  - Keep optional vs nullable exact. `footprint` is `array | null`, `speed` is `number | null`, and optional keys use `.optional()`, because the contract distinguishes "absent" from "null".

**Services and handlers:** `configuration.service.ts`, `sensors.service.ts`, `mission.service.ts` (the MISSION_HANDLERS registry, with `NotImplementedException` for unknown types) and `take-sensor-control-mission.handler.ts` are logic-identical copies.

**Controllers:**
- Replace `@MicroserviceController(x)` with `@Controller(x)`.
- Keep `@ApiTags`, `@ApiOperation` and `@ApiOkResponse` / `@ApiCreatedResponse` / `@ApiBadRequestResponse`.
- Add `@UsePipes(ZodValidationPipe)` from `nestjs-zod` at controller level, so validation stays scoped to lillith. The proxy's own `reports/zod-validation.pipe.ts` is untouched.
- The mission route stays `POST safepass/missions` and returns 201.

**Modules:** as in lillith; the proxy's `index.ts` barrels export the module classes.

**Not ported:** `main.ts`, `app.module.ts`, `app.setup.ts`, `config/`, `common/auth/`, `common/logging/`, `common/metrics/`, `health/`, `.env.example`, `Dockerfile`, eslint/prettier/vitest/nest-cli configs.

### 3. Wire into the proxy
**`apps/proxy/src/app.module.ts`:**
- Import `ConfigurationModule`, `SensorsModule` and `MissionModule`.
- Add `RouterModule.register([{ path: 'api/v1', children: [ConfigurationModule, SensorsModule, MissionModule] }])` so only these get the prefix.

**`apps/proxy/src/lillith-api/swagger.ts`:**
- `setupLillithSwagger(app)` builds the `DocumentBuilder` (title "Lillith API", same description).
- It calls `SwaggerModule.createDocument(app, config, { include: [ConfigurationModule, SensorsModule, MissionModule] })`, then nestjs-zod's `cleanupOpenApiDoc(...)`, then `SwaggerModule.setup('api/docs', ...)`.

**`apps/proxy/src/main.ts`:**
- After `create`, add `if (process.env.NODE_ENV !== 'production') { setupLillithSwagger(app); logger.log(...) }`.

### 4. Tests (Jest, `apps/proxy/src/tests/`)
**Unit tests.** Port lillith's specs, converting `vi.fn` to `jest.fn`:
- `lillith-safepass-timestamp.unit-test.ts`
- `lillith-configuration.service.unit-test.ts`
- `lillith-sensors.service.unit-test.ts`
- `lillith-mission.service.unit-test.ts`
- `lillith-take-sensor-control-mission.handler.unit-test.ts`
- Add `lillith-schemas.unit-test.ts` to cover what class-validator checked before: timestamp regex, uuid, enums, strictness.

Drop the specs for removed code: guard, metrics filter and env validation.

**Controller specs.** They used `test/create-test-app.ts`, so they are folded into the component test.

**`lillith-api.component-test.ts`.** It boots `Test.createTestingModule` with the three modules and the same `RouterModule` registration (not the full `AppModule`, which needs Redis), then checks with supertest:
- `GET /api/v1/configuration/sync` and `/api/v1/sensors/state` return 200, with the timestamp format and `sensors: []`
- a valid `POST /api/v1/safepass/missions` returns 201 `FAILED/SENSOR_UNAVAILABLE`
- an unknown field, a bad uuid or a bad timestamp returns 400
- `/api/v1/health` returns 404

### 5. Docs
Add `apps/proxy/src/lillith-api/README.md`, adapted from lillith's README:
- the API table without auth, health or metrics
- the SafePass timestamp format
- the Swagger location (`http://localhost:3000/api/docs`, non-production only)
- the `TODO(raas)` status
- that it runs inside sensor-proxy on the proxy port

## Verification
Run these from the sensor-proxy root:
- `pnpm typecheck`
- `pnpm lint`
- `pnpm format:check`
- `pnpm test:unit`
- `pnpm --filter @platform/sensor-proxy test -- lillith`, which includes the component test
- `pnpm build`, then check that `dist/` has no `lillith-api` artifacts referencing removed deps.

For a manual smoke test (optional, needs Redis per the proxy README), run `pnpm start`, then:
- `curl localhost:3000/api/v1/sensors/state`
- `curl localhost:3000/health` should be unchanged
- open `localhost:3000/api/docs`, which should show only the 3 lillith routes

Finally, confirm `git status` in `C:\Users\roiap\repos\100-platform-lilith` is unchanged.
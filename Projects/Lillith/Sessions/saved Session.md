# Take-Sensor-Control Mission Request Endpoint

## Context

External applications ("hsifa tkifa", "mars", etc.) need a way to request that our
platform take control of a sensor (currently always a camera) on behalf of a
mission. They will POST a mission payload to our backend; the backend must
translate that into a `POST /:cameraId/command/connect` call against
`device-manager-service` and report back whether control was granted.

`position` is accepted and validated today but intentionally unused (reserved
for a future feature). The only sensor type today is a camera, so
`operatorContext.sensorID` **is** the camera ID, and `operatorContext.stationID`
**is** the user ID (both simplifications the requirements owner confirmed are
correct for now).

Decisions confirmed with the user:
- Build a **new, standalone** device-manager client module for this feature
  (not an extension of the existing `apps/backend/src/deviceManagerClient/`),
  mirroring its structure/conventions but scoped under `apps/backend/src/mission/`
  so the plain class names (`DeviceManagerClientModule`, `DeviceManagerClientService`,
  `DeviceManagerError`) don't need a `Mission`-prefixed alias — the folder
  path provides the scoping instead.
- The endpoint is **service-to-service** (called by other applications, not a
  logged-in browser user), so it uses the existing `@MicroServiceController` +
  `MicroServiceKeyGuard` (`x-microservice-key` header) pattern — the same
  mechanism `device-manager-service` uses to call back into the backend.
- Route: `POST /mission/take-sensor-control`.
- `isFakeVendor`, `firstName`, `lastName` are hardcoded placeholders:
  `false`, `'Mission'`, `'Request'`.
- Response is a small ack DTO, not the raw device-manager response.
- **Lillit is this feature's owning team** (a new dev team, alongside existing
  teams `rotem` and `yanai` — there's already a standalone `apps/lillit`
  service). `libs/backend-dto/src/` namespaces DTOs per team (`platform/`,
  `rotem/`, `yanai/`, each with their own `dto/`, `enums/`, `types/`
  subfolders and a barrel `index.ts`, all re-exported from
  `libs/backend-dto/src/index.ts`). No `lillit/` namespace exists yet in
  `backend-dto` (`yanai/` currently exists only as a placeholder —
  `// Reserved for Yanai project DTOs... export {};` — showing the pattern to
  follow once a namespace has real content). This plan creates
  `libs/backend-dto/src/lillit/` following that same structure, and puts the
  new mission-request DTOs there instead of in `platform/`, since they're
  Lillit-specific and not shared platform concepts.

## Reference material (already read)

- `apps/backend/src/deviceManagerClient/deviceManagerClient.service.ts` — the
  pattern to mirror: one method per action, try/catch wrapping
  `httpService.axiosRef.<verb>()`, structured `logger.error(...)`, rethrow as a
  custom `Error` subclass carrying an `ERROR_CODES` code.
- `apps/backend/src/deviceManagerClient/deviceManagerClient.module.ts` — the
  `HttpModule.registerAsync` boilerplate (baseURL from
  `DEVICE_MANAGER_SERVICE_URL`, `x-microservice-key` header from
  `MICROSERVICE_KEY`) plus `onModuleInit` wiring of `AuthInterceptor` (SigV4)
  and `CaCertService` (custom CA / https agent). All existing client paths are
  relative to a baseURL that already ends in `/api/device-manager`.
- `apps/backend/src/deviceManagerClient/device-manager.controller.ts` +
  `device-manager.controller.module.ts` — example of an inbound
  `@MicroServiceController('device-manager')` controller/module pairing.
- `libs/backend-common-src/nest-core/src/decorators/microservice-controller.decorator.ts`
  and `.../guards/microservice-key.guard.ts` — `MicroServiceController(path)` =
  `@Controller({path})` + `@UseGuards(MicroServiceKeyGuard)` + `@PublicRoute()`;
  the guard hashes the incoming `x-microservice-key` header (md5) and compares
  to `process.env.HASHED_MICROSERVICE_KEY`.
- `libs/backend-dto/src/platform/dto/camera.ts` — `ConnectCommandDto`
  (`userId`, `isFakeVendor`, `firstName`, `lastName`, `cameraId`) is the exact
  request body device-manager's connect endpoint expects; `CameraTakeControlDto`
  is its response type. Both are reused as-is — no need to redefine them.
- `libs/backend-dto/src/platform/dto/index.ts` and
  `.../constants/errorMessages.ts` — barrel export and `ERROR_CODES` map
  conventions to extend.
- `apps/backend/src/camera/cameras.service.ts:420-443` — convention for
  translating a client-layer error into an HTTP response: catch the custom
  error type, log it, throw `HttpException({ statusCode, message, error },
  HttpStatus.FAILED_DEPENDENCY)`.
- Validation conventions confirmed: `class-validator`/`class-transformer`
  (not Zod — Zod is env-only in this repo), global `ValidationPipe({
  whitelist: true, transform: true })` in `apps/backend/src/main.ts`. IDs use
  `@IsUUID()`, ISO timestamp strings use `@IsDateString()`, nested objects use
  `@ValidateNested()` + `@Type(() => Class)`.

## Implementation

### 1. New DTOs — `libs/backend-dto/src/lillit/`

Create the namespace following the `rotem/` layout (kebab-case filenames,
`.dto.ts` suffix, one barrel `index.ts` per subfolder, one namespace
`index.ts`):

- `libs/backend-dto/src/lillit/dto/take-sensor-control-mission-request.dto.ts`
- `libs/backend-dto/src/lillit/dto/index.ts`
- `libs/backend-dto/src/lillit/index.ts` (`export * from './dto';`, matching
  `rotem/index.ts`'s `export * from './dto'; export * from './enums'; ...`
  minus the subfolders this feature doesn't need yet)

Field names in the request DTO match the external JSON contract exactly
(including the `stationID`/`sensorID` casing), since `class-transformer`
binds by property name and this is what callers will actually send:

```ts
// take-sensor-control-mission-request.dto.ts
import { IsDateString, IsNumber, IsString, IsUUID, ValidateNested } from 'class-validator';
import { Type } from 'class-transformer';
import { ApiProperty } from '@nestjs/swagger';

export class MissionPositionDto {
  @ApiProperty() @IsNumber() latitude: number;
  @ApiProperty() @IsNumber() longitude: number;
  @ApiProperty() @IsNumber() altitude: number;
}

export class MissionOperatorContextDto {
  @ApiProperty() @IsUUID() stationID: string;
  @ApiProperty() @IsUUID() sensorID: string;
}

export class TakeSensorControlMissionRequestDto {
  @ApiProperty() @IsUUID() missionId: string;
  @ApiProperty() @IsString() missionName: string;
  @ApiProperty({ type: () => MissionPositionDto })
  @ValidateNested() @Type(() => MissionPositionDto)
  position: MissionPositionDto;
  @ApiProperty({ type: () => MissionOperatorContextDto })
  @ValidateNested() @Type(() => MissionOperatorContextDto)
  operatorContext: MissionOperatorContextDto;
  @ApiProperty() @IsDateString() timestamp: string;
}

export class TakeSensorControlMissionResponseDto {
  @ApiProperty() missionId: string;
  @ApiProperty() cameraId: string;
  @ApiProperty() controlGranted: boolean;
}
```

Replace the placeholder `libs/backend-dto/src/yanai/index.ts`-style content
of the new `libs/backend-dto/src/lillit/index.ts` with real re-exports, and
add `export * from './lillit';` to `libs/backend-dto/src/index.ts` (alongside
the existing `platform`/`rotem`/`yanai` exports).

Add a new error code to `libs/backend-dto/src/platform/constants/errorMessages.ts`
(kept in `platform/` since `ERROR_CODES` is a shared cross-team map):
`DEVICE_MANAGER_ERROR: 'DEVICE_MANAGER_ERROR'`.

### 2. New client module — `apps/backend/src/mission/deviceManagerClient/`

Mirrors `apps/backend/src/deviceManagerClient/` one-to-one, but scoped inside
the `mission/` feature folder (so it's clearly this feature's own client, not
a change to the existing shared one), split into module/service/errors/logs
files:

- `deviceManagerClient.module.ts` — `DeviceManagerClientModule`: same
  `HttpModule.registerAsync` boilerplate as the existing
  `apps/backend/src/deviceManagerClient/deviceManagerClient.module.ts`
  (baseURL from `DEVICE_MANAGER_SERVICE_URL`, `x-microservice-key` header from
  `MICROSERVICE_KEY`, 30s timeout), same `onModuleInit` wiring of
  `AuthInterceptor` and `CaCertService`. Exports `DeviceManagerClientService`.
- `errors/device-manager.error.ts`:
  ```ts
  export class DeviceManagerError extends Error {
    public readonly code = ERROR_CODES.DEVICE_MANAGER_ERROR;
    constructor(error?: Error) {
      super(`Failed to perform device manager action: ${error?.message ?? 'Unknown error'}`);
      this.name = 'DeviceManagerError';
    }
  }
  ```
- `logs/device-manager-logs.ts` — typed shape for the structured log object,
  so the service doesn't build an untyped inline literal:
  ```ts
  export interface DeviceManagerErrorLog {
    msg: string;
    cameraId: string;
    operation: string;
    error: string;
  }
  ```
- `deviceManagerClient.service.ts`:
  ```ts
  @Injectable()
  export class DeviceManagerClientService {
    constructor(private readonly httpService: HttpService, private readonly logger: Logger) {}

    async sendConnectCommand(
      cameraId: string,
      payload: ConnectCommandDto,
    ): Promise<CameraTakeControlDto> {
      try {
        const { data } = await this.httpService.axiosRef.post<CameraTakeControlDto>(
          `/${cameraId}/command/connect`,
          payload,
        );
        return data;
      } catch (error) {
        const log: DeviceManagerErrorLog = {
          msg: 'Error sending take-control connect command',
          cameraId,
          operation: 'sendConnectCommand',
          error: error instanceof Error ? (error.stack ?? error.message) : String(error),
        };
        this.logger.error(log);
        throw new DeviceManagerError(error);
      }
    }
  }
  ```

### 3. New feature module — `apps/backend/src/mission/`

- `mission.module.ts` — imports `DeviceManagerClientModule` (from
  `./deviceManagerClient/deviceManagerClient.module`), declares
  `MissionController`, provides `MissionService` + `Logger`.
- `mission.service.ts`:
  ```ts
  @Injectable()
  export class MissionService {
    constructor(
      private readonly deviceManagerClientService: DeviceManagerClientService,
      private readonly logger: Logger,
    ) {}

    async takeSensorControl(
      request: TakeSensorControlMissionRequestDto,
    ): Promise<TakeSensorControlMissionResponseDto> {
      const { missionId, operatorContext } = request;
      const cameraId = operatorContext.sensorID; // sensor === camera today
      const userId = operatorContext.stationID;  // station === user today

      const connectPayload: ConnectCommandDto = {
        userId,
        cameraId,
        isFakeVendor: false,
        firstName: 'Mission',
        lastName: 'Request',
      };

      try {
        await this.deviceManagerClientService.sendConnectCommand(cameraId, connectPayload);
      } catch (error) {
        if (error instanceof DeviceManagerError) {
          this.logger.error({ msg: 'Failed to take sensor control', missionId, cameraId, error: error.message });
          throw new HttpException(
            { statusCode: HttpStatus.FAILED_DEPENDENCY, message: `Failed to take control of sensor ${cameraId} for mission ${missionId}`, error: error.message },
            HttpStatus.FAILED_DEPENDENCY,
          );
        }
        throw error;
      }

      return { missionId, cameraId, controlGranted: true };
    }
  }
  ```

  Note: `DeviceManagerClientService`/`DeviceManagerError` here are the new
  `mission/deviceManagerClient/` classes (imported from their relative paths),
  distinct from the identically-named classes in the existing top-level
  `apps/backend/src/deviceManagerClient/`. `mission.service.ts` only ever
  imports the local one, so there's no ambiguity at the import site — but
  don't add both to the same file's imports.
- `mission.controller.ts`:
  ```ts
  @ApiTags('mission')
  @MicroServiceController('mission')
  export class MissionController {
    constructor(private readonly missionService: MissionService) {}

    @ApiOperation({ summary: 'Take control of a mission sensor (camera)' })
    @ApiBody({ type: TakeSensorControlMissionRequestDto })
    @ApiResponse({ status: 201, type: TakeSensorControlMissionResponseDto })
    @Post('take-sensor-control')
    async takeSensorControl(
      @Body() request: TakeSensorControlMissionRequestDto,
    ): Promise<TakeSensorControlMissionResponseDto> {
      return this.missionService.takeSensorControl(request);
    }
  }
  ```

  Note: `position` is accepted (and validated) on the DTO but deliberately
  unused in `MissionService` for now — add a one-line comment noting it's
  reserved for a future feature, per the requirement.

### 4. Wire into `apps/backend/src/app.module.ts`

Add `MissionModule` to the `imports` array alongside `DeviceManagerControllerModule`/`CamerasModule`.

### 5. Tests

- `mission.controller.module-spec.ts` — verifies the route delegates to
  `MissionService` and that `MicroServiceKeyGuard` rejects requests missing/
  with an invalid `x-microservice-key`.
- `mission.service.unit-spec.ts` — verifies `sensorID`→`cameraId` and
  `stationID`→`userId` mapping, the hardcoded placeholder values sent to
  `sendConnectCommand`, the success response shape, and that a
  `DeviceManagerError` is translated into a `FAILED_DEPENDENCY`
  `HttpException`.
- `apps/backend/src/mission/deviceManagerClient/deviceManagerClient.service.unit-spec.ts` —
  mirrors existing specs for the top-level `DeviceManagerClientService`
  methods (mock `HttpService.axiosRef.post`, assert success passthrough and
  that a failure is logged via the typed `DeviceManagerErrorLog` shape and
  rethrown as `DeviceManagerError`).

## Verification

1. `turbo build --filter=@libs/backend-dto` then `turbo build --filter=backend`
   to confirm the new DTOs/module compile and are exported correctly.
2. `turbo test:unit --filter=backend` (or `turbo test:unit --filter=@libs/backend-dto`
   for the DTO package) to run the new specs above.
3. `turbo lint --filter=backend` to confirm formatting/naming conventions pass.
4. Manual smoke test via `tilt up -- --yanai` (or `npm run dev`): POST to
   `http://localhost:<backend-port>/mission/take-sensor-control` with a valid
   `x-microservice-key` header and a sample mission JSON, against a running
   `device-manager-service` with a known `cameraId`/`userId`; confirm a 201
   with `{ missionId, cameraId, controlGranted: true }`, and confirm a missing/
   wrong `x-microservice-key` yields 401, and a bad `cameraId` yields a 424
   `FAILED_DEPENDENCY` from `device-manager-service`.
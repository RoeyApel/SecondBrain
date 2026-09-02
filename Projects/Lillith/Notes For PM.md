# Notes / Questions

## 1. Camera heading value

For cameras is live `pan_deg` can be used for the heading value? For now at least?

`heading` value is a fixed value that is supposed to be set in the DB after installation of the camera (not currently found in jupiter DB).

## 2. Status inconsistency

**In the system**, the camera statuses are:

| Status        | Meaning                                              |
| ------------- | ---------------------------------------------------- |
| `ACTIVE`      | Camera connected                                     |
| `INACTIVE`    | Camera isn't connected OR any fetal connection fail  |
| `SEMI_ACTIVE` | Connected but there are problems with the connection |

**In the spec**, the statuses are:

- `ACTIVE`
- `INACTIVE`
- `FAIL`
- `BUSY`

## 3. Footprint data

	Can't get footprint data.

## 4. Sensor type

Sensor type is `EVO` or `PTZ`. Shouldn't it be camera or drone instead?

> What is `EVO`?

## 5. Coverage is not available for every camera

- Live aim readings only exist for cameras on the older control protocol (`AVIV`, `BARKAN`, `NETZ`, `TZUKIT`) and only while that camera is actively connected.
- **ONVIF and KELA cameras never report aim data at all**, so they will have no `coverage` object.

Is it ok?

## 6. maxRange_m source

`maxRange_m` value is gotten currently from the geo server service.

Is it a problem? Should we get the geo server service as part of lillit just for that value?

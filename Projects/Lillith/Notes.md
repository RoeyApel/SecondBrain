```json
{
  "timestamp": "26082026 10:30:00",
  "missionID": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "missionType": "Pair",
  "operatorContext": {
    "hamalID": "hamal-1",
    "stationID": "some-station-id",
    "sensorID": "3fa85f64-5717-4562-b3fc-2c963f66afa7"
  },
  "telemetry": {
    "latitude": 32.0853,
    "longitude": 34.7818,
    "altitude": 100,
    "heading": 90,
    "speed": 12.5
  }
}
```

## Suggestions

1. change name position --> telemetry (more fitting name)
## Questions: 

1. why have a mission type field instead of having different routes for different missions
2. currently no way to tell what type of sensor is sent (no sensorType field)


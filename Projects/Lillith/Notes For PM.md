notes/questions:
1. for cameras is live pan_deg can be used for the heading value? for now at least?. heading value is a fixed value that is supposed to be set in the db after installation of the camera (not currently found in jupiter db). 
2. status inconsistency 
- in the system the camera statuses are: ACTIVE, INACTIVE (camera isn't connected OR any fetal connection fail), SEMI_ACTIVE (connected but there are problems with the connection)
- in the spec the statuses are: ACTIVE, INACTIVE, FAIL, BUSY 
3. can't get footprint data
4. sensor type is EVO or PTZ. shouldn't it be camera or drone instead. (what is EVO?)
5. Coverage is not available for every camera:
- Live aim readings only exist for cameras on the older control protocol (`AVIV`, `BARKAN`, `NETZ`, `TZUKIT`) and only while that camera is actively connected. **ONVIF and KELA cameras never report aim data at all**, so they will have no `coverage` object. Is it ok?
6. maxRange_m value is gotten currently from the geo server service. Is it a problem? Should we get the geo server service as part of lillit just for that value?  

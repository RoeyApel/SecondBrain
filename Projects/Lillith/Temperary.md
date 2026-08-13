{ 
"missionId": "M-8899-TEST", 
"userId": "dc68541d-08ec-4aa0-9737-0a77e1f07d0d", 
"cameraId": "86d7d08b-7933-4d20-9494-ffffe0679569" 
} 
# prev safepass env

--------------------------------------------
PORT=4000,

MICROSERVICE_KEY=efd6831182f3706bb91a4197d29ca9ee5a36bac2cf9f92c9be1a0ef705c29fee

TARGET_URL=http://localhost:3060/api/device-manager/ef55153c-d60c-4a25-9e87-942258617397/command/connect

-------------------------------------------------------------------------
# safepass env now

-------------
PORT=4000

// taken from secrets in monorepo
MICROSERVICE_KEY=efd6831182f3706bb91a4197d29ca9ee5a36bac2cf9f92c9be1a0ef705c29fee

DEVICE_MANAGER_BASE_URL=http://localhost:3060/api/device-manager

OPERATOR_FIRST_NAME=SafePass

OPERATOR_LAST_NAME=Operator

IS_FAKE_VENDOR=false

------------------------
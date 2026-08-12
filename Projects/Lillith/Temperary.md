{
    "missionId": "M-8899-TEST",
    "userId": "f443f715-f55e-4975-b733-c1a5573af4a9",
    "cameraId": "84dd51e0-2538-43e3-82e2-dbdbaadece45"
  }
84dd51e0-2538-43e3-82e2-dbdbaadece45 camera
f443f715-f55e-4975-b733-c1a5573af4a9 user
# prev safepass env

--------------------------------------------
PORT=4000,

MICROSERVICE_KEY=efd6831182f3706bb91a4197d29ca9ee5a36bac2cf9f92c9be1a0ef705c29fee

TARGET_URL=http://localhost:3060/api/device-manager/ef55153c-d60c-4a25-9e87-942258617397/command/connect

-------------------------------------------------------------------------
# safepass env now

-------------
PORT=4000

MICROSERVICE_KEY=efd6831182f3706bb91a4197d29ca9ee5a36bac2cf9f92c9be1a0ef705c29fee

DEVICE_MANAGER_BASE_URL=http://localhost:3060/api/device-manager

OPERATOR_FIRST_NAME=SafePass

OPERATOR_LAST_NAME=Operator

IS_FAKE_VENDOR=false

------------------------
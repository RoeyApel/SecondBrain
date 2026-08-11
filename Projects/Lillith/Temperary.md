{
    "missionId": "M-8899-TEST",
    "userId": "1bdc1b62-da88-4621-806d-182771df9bfd",
    "cameraId": "a87edfca-1651-4c2d-92dc-1f60ecdd7645"
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

MICROSERVICE_KEY=efd6831182f3706bb91a4197d29ca9ee5a36bac2cf9f92c9be1a0ef705c29fee

DEVICE_MANAGER_BASE_URL=http://localhost:3060/api/device-manager

OPERATOR_FIRST_NAME=SafePass

OPERATOR_LAST_NAME=Operator

IS_FAKE_VENDOR=false

------------------------
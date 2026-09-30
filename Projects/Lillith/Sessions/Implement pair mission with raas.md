## Prompt
-----------
I want to implement the pair mission (take sensor control mission. route safepass/missions).

very important info is that in raas and proxy context asset is the word for devices like drones and sensors and in lillith context an asset is called a sensor and is a word for devices like drones and cameras. we stick to proxy wording but not changing the missions request fields from sensorid to assetId for now because that change was not approved yet by the manager.

and raas doesn't yet working with ptz cameras but will in the future.

so for now the pair mission is with a simple drone.

pair meaning:

1. connect to the right sensor using raas client and then acquire control, then saving the handle for later to run commands on it (find a good way to do it so that with stationId I can get the current handle for the current asset that the station is controlling).
    

2. after successful acquiring of control, the client (frontend) with the stationId that acquired control on the asset, now can control it.
    
3. for the client to be able to control the asset you need to implement a api endpoints for all the commands of the asset. for now just move and zoom.
    

all of the code should be best practice, clean null safe type safe and the standards should be similar to the current project standards when the standards are good.

ask a lot of question to know exactly what I meant don't assume. I want to make all the non obvious decision.

my explanation is a bit of a mess so improve it of course in the actual plan.

--------

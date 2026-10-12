## Prompt
-----------
I want to implement the pair mission (take asset control mission. route safepass/missions).

and raas doesn't yet working with ptz cameras but will in the future.

so for now the pair mission is with a simple drone.

pair meaning:

1. connect to the right asset using raas client and then acquire control, then saving the handle for later to run commands on it (with a seperate endpoint). look at how assets are being strored in other places in the repo tell me about it and ask me how I would like to implement it in this case (give me good well thought out suggestions). 

2. after successful acquiring of control, the client (frontend) with the stationId that acquired control on the asset, now can control it.
    
3. for the client to be able to control the asset you need to implement a api endpoint for all the commands of the asset, the endpoint has a query parameter called command for the command and all the for now just move and zoom.
    

all of the code should be best practice, clean null safe type safe and the standards should be similar to the current project standards when the standards are good.

ask a lot of question to know exactly what I meant don't assume. I want to make all the non obvious decision.

my explanation is a bit of a mess so improve it of course in the actual plan.

--------

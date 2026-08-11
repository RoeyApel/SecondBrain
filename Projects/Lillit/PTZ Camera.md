# Connection

1. Go to SADP to find the camera's IP and port.
2. Connect to the router.
3. Log in with:
   - Username: `admin`
   - Password: `Maccabi14`
4. Enter the IP as the URL in Chrome to access the camera config (log in with the username and password above).
5. Make sure ONVIF is enabled, and verify the admin user is created with the credentials above.
6. In `https://localhost/management`, add the camera using the config shown in the three images below (IP always from SADP).
7. Tilt services needed: `frontend_local` (`-rotem`, or regular if Yanai), `backend_local`, `device-manager-service_local`, `video-server`.

![[Pasted image 20260810120350.png]]
![[Pasted image 20260810133633.png]]
![[Pasted image 20260810133848.png]]

## Root cause

The bug is an **identity-namespace split between two independent "who controls this camera" write paths** in `device-manager`:

1. **Socket path (used by the UI's own Take Control click)** — `apps/frontend-rotem` → `CameraItem.tsx:175` calls `controlCamera(...)`, which emits `CAMERA_CONNECT` over the browser's already-authenticated socket. On the server, `device-manager.gateway.ts`'s `cameraConnect` handler **ignores any `userId` in the payload** and always uses `getSocketSpotterId(client)` — i.e. `client.data.user.id`, the real logged-in user. All later `move`/`zoom`/`focus` socket commands are authorized the same way (`device-manager.gateway.ts:260-268`), so this path is self-consistent by construction.
    
2. **REST path (Postman → lillit → device-manager)** — `mission.service.ts:27-33` builds `connectPayload.userId = operatorContext.stationID` and POSTs it to `device-manager`'s `/{sensorId}/command/connect` (`deviceManagerClient.service.ts:17-46`). That REST controller _does_ take `userId` from the body and writes `userIdToCameraControlId.set(stationID, cameraId)` directly (`device-manager.service.ts:~1199`).
    

These two paths write into the **same map** but only agree when `stationID` in the mission JSON equals the `id` of the user account whose browser socket is live. In this seed data `user.id === station.id` by design (`seed-lillit.ts:56-78`, comment: "user id = station id"), so if you Postman a `stationID` that isn't the exact account you're logged into the UI as, device-manager registers control under a different key than your socket is authenticated as. Your next `move`/`zoom`/`focus` from the UI then hits `getUserControlledCameraId` → `getControlledCameraIdByUserId(yourRealId)` → not found → `"Camera not found for user ${spotterId}"`, caught per-handler and logged as the errors you're seeing, with `callback({ success: false })` and no socket event emitted.

## Why the UI still _shows_ "you have control"

The camera's `activeSpotterId` (pushed/polled from the backend) gets set to the `stationID` mission payload set. `DeviceBadge.tsx:34` shows "בשליטה" (in control) whenever `activeSpotterId === user?.id`. If you happened to test with a `stationID` that matches your logged-in user id, the badge looks correct — but the frontend's own local `isControllingCamera` flag never got set, because the frontend's `controlCamera` (the socket call that actually registers the _live_ controller) was never invoked for this externally-initiated take-control.

**This exact scenario is already anticipated in your own uncommitted working tree.** In `CameraItem.tsx` you replaced a hardcoded debug-only effect with a generalized one meant to fix precisely this — but it's currently **commented out**:

```tsx
// useEffect(() => {
//   if (!isControllingCamera && activeSpotterId && activeSpotterId === user?.id) {
//     void (async () => {
//       await controlCamera({ name, channels, deviceId, vendor, user });
//       setIsCameraInMaintenance(false);
//     })();
//   }
// }, [...]);
```

With it disabled, an externally-connected camera (mission/SafePass) never gets its socket-side control re-issued by the frontend, so `isControllingCamera` stays `false` and device-manager never registers your socket's real user id as the controller — hence no move/zoom/focus events.

## Plan

1. **Uncomment that `useEffect`** in [CameraItem.tsx](vscode-webview://0oqv6b7nv0tpri0fic1gvopa9la7rss27q263hd6m4ij2uors27h/apps/frontend-rotem/src/rotem/overview/deviceItem/CameraItem.tsx#L89-L110). When the backend reports `activeSpotterId === user.id` but the frontend's local `isControllingCamera` is still `false`, it will auto re-issue `controlCamera` over the authenticated socket — which registers control under the _same_ id (`client.data.user.id`) that subsequent move/zoom/focus commands are checked against, closing the gap.
2. **Verify with matching IDs**: retest the Postman mission call using the `stationID`/`sensorID` of the account and camera you're actually logged into the UI as (the seeded Lillit station user, where `user.id === station.id`), so the mismatch isn't masking whether the fix worked.
3. **Confirm no re-render loop**: check that once `controlCamera` resolves and `isControllingCamera` flips `true`, the effect's dependency array stabilizes (it should, since the guard condition becomes false).
4. **Longer term** (the comment already flags this): replace this poll-and-resync client hack with a proper server push — device-manager should notify the actual owning client via socket when a REST-originated connect happens, rather than relying on the frontend re-issuing its own connect. Track as follow-up, not blocking this fix.

Want me to go ahead and uncomment/re-enable that effect now, or would you like to test the ID-matching theory in Postman first?
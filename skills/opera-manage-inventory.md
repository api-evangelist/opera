---
name: Register an Opera Ads app and placement
description: Create and manage publisher inventory (apps and ad placements) via the Opera Ads Inventory Management API.
api: openapi/opera-ads-openapi.yml
operations: [createApp, listApps, updateApp, createPlacement, listPlacements, updatePlacement]
---

# Register an Opera Ads app and placement

Onboard publisher inventory by creating an app, then adding ad placements to it.

## Auth
- Add header `Authorization: Bearer <token>` and `Content-Type: application/json`.
- Base host: `https://ofp.adx.opera.com/openapi/inventory/v1`.

## Steps
1. Create the app with `createApp` (`POST /app/create`) — supply `name`, `deviceType`, `osType`,
   `trafficType`, `pkgName` or `domainName`, `iabCategory`, and `coppaCompliant`. The response returns
   `appId`, `publisherId`, and `displayId`.
2. Confirm with `listApps` (`POST /app/list`), filtering by `deviceType`/`osType`/`trafficType`/`status`.
3. Add a placement with `createPlacement` (`POST /placement/create`) — supply `appId`, `name`,
   `floorPrice`, `sdkAdFormat`, and refresh/reward settings. The response returns the placement `sid`.
4. Adjust later with `updateApp` (`POST /app/update`, requires `appId`) or `updatePlacement`
   (`POST /placement/update`, requires `sid`) — both support partial updates.
5. Review placements with `listPlacements` (`POST /placement/list`).

## Rules
- All operations are POST with JSON bodies; updates are incremental (send only changed fields).
- Handle HTTP 403 (insufficient scope) and 429 (rate limit) per errors/opera-problem-types.yml.

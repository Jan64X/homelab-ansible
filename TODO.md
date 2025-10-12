- Sharkey role has directory permissions change, check later

TASK [sharkey : Ensure files directory exists with correct permissions] ************************************************
changed: [sharkey]

- Add restarting container stack for observability when configs changed

TASK [navidrome : Add NAS share to fstab for backups (noauto)] *********************************************************
[WARNING]: Ignore the 'boot' due to 'opts' contains 'noauto'.
ok: [navidrome]

- Unifi Controller role has directory permissions change, check later

TASK [unificontroller : Ensure UniFi data directory exists with correct permissions] ***********************************
changed: [unificontroller]

- Add a task/role to update all docker compose stacks?

- Dang I realised I left the CDN role html template personalized, gotta fix that too

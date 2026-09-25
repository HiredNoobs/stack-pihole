# stack-pihole

Pihole w/Unbound deployment for k8s.

One replica runs on each node labelled `pihole_core_pihole` (set in iac-homelab), the replica count is worked out at deploy time. Each pod runs:

- `pihole` - settings come from the `pihole-settings` ConfigMap (`k8s/settings.yaml` and `stack.env`) as `FTLCONF_` env vars, so they are read only in the web UI. Changing them restarts the pods.
- `unbound` - Pi-hole's upstream on `127.0.0.1#5335`, config from `configs/*.conf`.
- `sync-lists` - keeps the block lists and allow/deny domains in line with `configs/lists/` through the local Pi-hole API. Changes are picked up without a restart and anything not in the files is removed. The pod isn't ready until the first sync (including the gravity update) has finished.

There is no volume, the block lists are downloaded again when a pod starts and query history doesn't survive a restart.

`stack-kube-vip` announces `$PIHOLE_LB_IP`, without it DNS is only reachable inside the cluster.

| Service | Type | Used for |
| --- | --- | --- |
| `pihole-dns` | LoadBalancer (`$PIHOLE_LB_IP`, kube-vip) | DNS for the LAN |
| `pihole-web` | ClusterIP | Web UI/API on any replica, e.g. `pihole.$DOMAIN` via nginx |
| `pihole` | Headless | Web UI/API per replica, e.g. `pihole-0.pihole.pihole-core.svc` for homepage widgets or `pihole1.$DOMAIN` via nginx |

`secrets.env` can set:

- `PIHOLE_WEB_ADMIN_PASS` - web UI/API password.
- `PIHOLE_API_APP_PWHASH` - optional app password hash for API clients like homepage, shared by every replica. Generate an app password in the web UI and copy `webserver.api.app_pwhash` from `pihole.toml`.

## Deployments

- `core/production` - the `production.core` cluster.
- `core/development` - the `development.core` cluster (to be built), `PIHOLE_LB_IP` still needs setting.

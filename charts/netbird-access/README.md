# netbird-access

Exposes in-cluster services to NetBird peers via the NetBird Kubernetes operator. Generic and
**list-driven** — each cluster supplies its own resources (typically via `jx-values`). Part of DEV-123.

## What this chart manages (GitOps)
- `NetworkRouter` (`router.name`) — the operator turns this CR into the routing-peer pod, registered under `dnsZone`.
- `NetworkResource` ×N — one per `resources[]` entry; **no group on the resource** (access is a per-resource policy).
- Optional `clusterIpServices[]` — plain ClusterIP wrappers (for in-cluster NodePort/headless services, or to scope ports).
- Optional `externalServices[]` — selector-less Service + Endpoints (for off-cluster targets, e.g. an external mongos).

```yaml
dnsZone: <region>.dev.netbird.<domain>      # supplied per cluster (jx-values)
router: { name: jx-staging-router }
resources:
  - { name: redis,      serviceRef: redis-cluster }      # point straight at an existing ClusterIP
  - { name: clickhouse, serviceRef: clickhouse-netbird } # or at a wrapper below
clusterIpServices:                                       # only if you need a wrapper / port-scope
  - { name: clickhouse-netbird, selector: {...}, ports: [ {name: http, port: 8123}, {name: tcp, port: 9000} ] }
externalServices:                                        # only for off-cluster targets
  - { name: mongo-netbird, portName: mongo, externalHost: 51.68.95.180, port: 27017 }
```

> **When do you need a wrapper?** Point `serviceRef` straight at an existing Service when it's an
> in-cluster **ClusterIP with the ports you want**. Add a `clusterIpServices` wrapper to **scope ports**
> (a `NetworkResource` exposes the *whole* Service) or for NodePort/headless services. Add an
> `externalServices` wrapper only for **off-cluster** endpoints (no in-cluster pods/endpoints).

**This chart does NOT create the access Groups or any SetupKeys** — see the access model below for why.

## Access model (SSO-driven, per database)
A user's Keycloak realm roles ride in the `groups` claim (netbird client's realm-role→groups mapper).
NetBird **JWT group-sync** creates a NetBird group for each claim value and auto-adds the peer on login.

> ### ⚠️ Groups must be JWT-issued, NOT operator-created
> NetBird only adds peers to groups it **issued itself (`issued=jwt`)**. A group created by the
> operator (or the API) is `issued=api`, and **JWT sync silently refuses to populate it** — peers
> never join and SSO access fails with no error. So the access groups `netbird-clickhouse` /
> `netbird-mongo` must be **created by JWT sync** (from the same-named realm roles), which means the
> operator must NOT create same-named groups. That's why this chart no longer ships a `Group` (or
> `SetupKey`, whose `autoGroups` would need an operator group). The resources carry **no group** and
> access is granted purely by policy `destinationResource`.

**Grant a user access** = add them to the lopz-rbac group (one-line PR), then they `netbird up`:
- ClickHouse → `/anyip-staging/netbird/clickhouse` (role `netbird-clickhouse`)
- MongoDB → `/anyip-staging/netbird/mongo` (role `netbird-mongo`)
Removing the role revokes on next login (full-sync).

## ⚠️ Manual steps the operator CANNOT do (out-of-band, one-time per env)

### 1. Enable JWT group-sync (account setting)
```bash
export NB_TOKEN=$(kubectl -n netbird get secret netbird-mgmt-api-key -o jsonpath='{.data.NB_API_KEY}' | base64 -d)
API=https://vpn.lop3z.one/api
ACC=$(curl -s -H "Authorization: Token $NB_TOKEN" "$API/accounts" | jq -r '.[0].id')
curl -s -X PUT -H "Authorization: Token $NB_TOKEN" -H "Content-Type: application/json" \
  -d '{"settings":{"jwt_groups_enabled":true,"jwt_groups_claim_name":"groups",
       "jwt_allow_groups":[],"groups_propagation_enabled":true}}' \
  "$API/accounts/$ACC"
```
> **`jwt_allow_groups` MUST be empty `[]`.** It is a **login gate**, not a propagation filter — set
> it to anything and every user whose token lacks those groups gets **401 on the portal** (this was
> the original "other users get 401" bug). Least-privilege is enforced by the **policies** below,
> not by this setting. With it empty, NetBird mirrors every realm role as a group (cosmetic clutter).

### 2. Per-resource access policies (REQUIRED — nothing is reachable without them)
The operator can't create access policies. Use **one policy per resource** — NetBird does **not**
reliably distribute two `destinationResource` rules in a single policy (only the first resource gets
routed; symptom: one DB works, the other times out). The policy **source is the jwt-issued group**,
which only exists **after** the first user with that role logs in.
```bash
# group ids (jwt-issued, appear after first SSO login): .../groups
# resource ids: .../networks/<network-id>/resources
CH_GRP=...; CH_RES=...; MONGO_GRP=...; MONGO_RES=...
curl -s -X POST -H "Authorization: Token $NB_TOKEN" -H "Content-Type: application/json" \
  -d "{\"name\":\"netbird-db-access\",\"enabled\":true,\"rules\":[
        {\"name\":\"clickhouse\",\"enabled\":true,\"action\":\"accept\",\"bidirectional\":true,\"protocol\":\"all\",
         \"sources\":[\"$CH_GRP\"],\"destinationResource\":{\"id\":\"$CH_RES\",\"type\":\"host\"}}]}" "$API/policies"
curl -s -X POST -H "Authorization: Token $NB_TOKEN" -H "Content-Type: application/json" \
  -d "{\"name\":\"netbird-mongo-access\",\"enabled\":true,\"rules\":[
        {\"name\":\"mongos\",\"enabled\":true,\"action\":\"accept\",\"bidirectional\":true,\"protocol\":\"all\",
         \"sources\":[\"$MONGO_GRP\"],\"destinationResource\":{\"id\":\"$MONGO_RES\",\"type\":\"host\"}}]}" "$API/policies"
```

### 3. DNS zone
`NetworkRouter.dnsZoneRef` (`sbg5.dev.netbird.lop3z.one`) must exist in NetBird DNS Zones first
(operator looks it up by name, never creates it).

### 4. Do NOT add client peers to the `networkrouter-*` group
A peer in the routing-peer group is treated as a **router** and gets **no client route** (symptom:
`netbird networks list` empty, traffic leaves via the default interface).

### 5. (Optional) DNS names instead of IPs — name-based, VPN-only access
To let peers reach DBs by **name** (resolvable only on the VPN, not public), add **domain** network
resources — the operator can't (`NetworkResource` is IP-only), so create them via API. The in-cluster
router resolves the FQDN via CoreDNS, so there's no kube-dns/subnet exposure. One domain resource +
one policy per DB (same one-policy-per-resource rule as step 2):
```bash
NETID=$(curl -s -H "Authorization: Token $NB_TOKEN" "$API/networks" | jq -r '.[0].id')
curl -s -X POST -H "Authorization: Token $NB_TOKEN" -H "Content-Type: application/json" \
  -d '{"name":"clickhouse-dns","address":"clickhouse-netbird.<ns>.svc.cluster.local","enabled":true}' \
  "$API/networks/$NETID/resources"   # repeat for mongo-netbird.<ns>.svc.cluster.local
# then POST a policy per resource: source = the jwt group, destinationResource = {id, type:"domain"}
```
Then connect by name: `mongosh "mongodb://mongo-netbird.<ns>.svc.cluster.local:27017/?directConnection=true"`.
**Do NOT use `external-dns` for this** — that publishes a *public* DNS record; NetBird split-DNS keeps
the name VPN-only.

> Current live policy set (SBG5 dev): `netbird-clickhouse-access` + `netbird-mongo-access` (IP, step 2)
> and `netbird-clickhouse-dns` + `netbird-mongo-dns` (domain, step 5).
>
> Future improvement: manage the policies + account setting via the NetBird Terraform provider or an
> idempotent Job, to remove these manual steps.

## Bootstrap order (fresh env)
1. Merge this chart → operator creates the router + resources (no groups).
2. Enable JWT group-sync (step 1) with `jwt_allow_groups: []`.
3. Assign the `netbird-clickhouse` / `netbird-mongo` roles to a user in lopz-rbac; have them
   `netbird up` once → JWT sync creates the jwt groups and joins their peer.
4. Read the jwt group ids + resource ids, create the two policies (step 2).

## Connect
```bash
# ClickHouse (peer in netbird-clickhouse)
curl "http://<clickhouse-route>:8123/?user=default&password=..." --data-binary "SELECT version()"
# MongoDB mongos (peer in netbird-mongo)
mongosh "mongodb://<mongos-route>:27017/" --username <user> --authenticationDatabase admin
```
Routes/addresses: `netbird networks list` (or NetBird portal → Networks).

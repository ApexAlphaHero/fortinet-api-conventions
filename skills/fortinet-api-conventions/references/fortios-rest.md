# FortiOS REST (direct to a FortiGate)

## Authentication and reachability

- Per-gate **api-user token** over `Authorization: Bearer`; HTTPS on the gate's admin port
  (443 or the configured `admin-sport`). Pin or verify the certificate deliberately — most gates
  run a self-signed or factory cert; make verification a per-credential switch, never silently off.
- A fleet where every gate carries its OWN api-user cannot be polled with one shared token.
  Resolve the credential per device (device credential → integration default), and never let a
  credential's configured host override the device address: a wrong host mis-attributes one
  device's data to twenty.
- Under FortiManager proxy transport with no device token, REST reads are impossible; fall
  back to ICMP for reachability and to FMG's `/dvmdb` for state, and say so in the UI rather than
  letting a "rest_api" method silently collect nothing.

## Admin session login (username + password)

When a token is not available and an admin login is, sign in the way the 7.4+ GUI does — NOT
through `/logincheck`. Seen on FortiGate 61F, FortiOS 7.6.7 build3704:

- **`POST /logincheck` is dead for this.** It answers **200 with the (gzipped) login page** and
  sets **no cookie**, whatever the password. A client that reads "200 + no error" as success, or
  waits for a `ccsrftoken` cookie, simply never gets a session.
- **Login:** `POST /api/v2/authentication`, JSON `{ "username": "...", "password": "..." }`.
  The field is `password` — `secretkey` (the old form field) answers `LOGIN_FAILED`.
- **The verdict is the BODY, never the cookies.** `{"status":6,"status_message":"LOGIN_SUCCESS"}`
  on success; `{"status":-1,"status_message":"LOGIN_FAILED"}` on a bad password — and the gate
  sets `session_key_<port>_<hash>` and `ccsrf_token_<port>_<hash>` cookies **on the failure too**.
  Any other `status_message` is a prompt (two-factor, pre-login disclaimer, password change) a
  script cannot answer: stop, don't retry — each attempt counts toward the admin lockout.
- **The CSRF cookie is `ccsrf_token_<port>_<hash>`** (older builds: `ccsrftoken` /
  `ccsrftoken_<port>_<id>`) — match `/^ccsrf_?token/`, strip surrounding quotes, echo it as
  `X-CSRFTOKEN` on writes. The gate also names it, unauthenticated, in
  `GET /api/v2/service/login-config` → `ccsrf_token_cookie_name`.
- **Logout:** `DELETE /api/v2/authentication` → `HTTP_AUTHD_LOGOUT_SUCCESS` (did not require the
  CSRF header). Pre-login state: `GET /api/v2/authentication-status` → `{ login_locked_out }`,
  `GET /api/v2/authentication` → `{ authenticated }` — both unauthenticated, neither costs an
  attempt.
- An older build without the endpoint answers it with 401/404 or non-JSON: only a JSON
  `status_message` decides anything, so fall back to `/logincheck` (form `ajax=1`, `username`,
  `secretkey`; success = a `ccsrftoken` cookie) only then.
- FortiOS gzips HTML answers even when the request sent no `Accept-Encoding`; inflate on
  `content-encoding: gzip` (or the `1f 8b` magic) before reading a body.

## Firmware upgrade (FortiGate)

`POST /api/v2/monitor/system/firmware/upgrade` with `source=upload`. Seen on the same 61F
(7.6.7 build3704 → 8.0.1 build245, 99.7 MB image), through a bearer token AND through the admin
session above (with `X-CSRFTOKEN`):

- **Multipart works:** a form field `source=upload` then the image in a `file` part, streamed
  with a precomputed Content-Length — ~7 s for 99.7 MB on a LAN. (The documented alternative,
  JSON `file_content` base64, inflates the body by 4/3; not needed here, and unproven.)
- The answer comes ~8 s after the last byte; the gate **stops answering ~30 s later**, then the
  web server comes back **before** the REST API — the first `monitor/system/status` after the
  reboot got a dropped socket, the next one (20 s later) the new version. Total ~4.5–5 min.
  Budget a retry loop for the version check, not a single read.
- `GET /api/v2/monitor/system/firmware` → `results.current` `{version, major, minor, patch, build,
  platform-id}` is a cheap pre-check that the token can read the firmware tree; `platform-id`
  (`FGT61F`) is the image platform the gate expects.
- **Image identity.** A FortiGate `.out` is a **gzip stream whose embedded file NAME is the header
  token**, e.g. `FGT61F-8.00-FW-build0245-260909-patch01-F-260421`, inside the first 512 bytes:
  platform (`FGT61F` = the serial's first six characters), `8.00` + `patch01` = 8.0.1, build 245 —
  the same token shape the FortiSwitch image header carries, with the same `FW` marker.
- Check `GET /api/v2/cmdb/system/ha` → `mode` before flashing: anything but `standalone` means
  the upgrade reboots every cluster member.
- The REST upload does **not** enforce Fortinet's supported upgrade path; a 7.6 → 8.0 jump was
  accepted. Choosing a supported step is the caller's job.

## Paths

| Need | Path | Notes |
|---|---|---|
| system status, hostname, serial, version | `/api/v2/monitor/system/status` | serial here is the chassis; HA cluster members differ |
| interfaces + addresses | `/api/v2/cmdb/system/interface`, `/api/v2/monitor/system/interface` | cmdb = configured, monitor = live counters. **monitor omits every `type tunnel` interface** (IPsec phase1-interfaces incl. ADVPN overlays, GRE, VXLAN) — read their name and `ip` from cmdb or they vanish from an inventory built on monitor |
| DHCP servers / scopes | `/api/v2/cmdb/system/dhcp/server` | `reserved-address[]` lives INSIDE each server object |
| DHCP leases | `/api/v2/monitor/system/dhcp` | per-scope lease table; supports `?scope=` |
| ARP table | `/api/v2/monitor/network/arp` | L3 neighbor cache (IP, MAC, interface) |
| VIPs | `/api/v2/cmdb/firewall/vip` | device-owned addresses; never "assignable" |
| managed switches | `/api/v2/monitor/switch-controller/managed-switch/status`, `/api/v2/cmdb/switch-controller/managed-switch` | monitor may omit fields via FMG proxy |
| managed APs | `/api/v2/monitor/wifi/managed_ap`, `/api/v2/cmdb/wireless-controller/wtp` | radio[] and client counts are here even when the AP itself has SNMP off |
| wireless clients | `/api/v2/monitor/wifi/client` | |
| address groups (quarantine) | `/api/v2/cmdb/firewall/addrgrp`, `/api/v2/cmdb/firewall/address` | MAC-typed addresses |
| SD-WAN health | `/api/v2/monitor/virtual-wan/health-check`, `/api/v2/monitor/virtual-wan/members`, `/api/v2/cmdb/system/sdwan` | the selected route is not always exposed; label inferred values as inferred |
| DHCP lease release | `/api/v2/monitor/system/dhcp/revoke` | body `{ "ip": [...] }` |

Always pass `vdom=` where the gate has VDOMs; many monitor endpoints default to the
management VDOM only.

## Access-profile groups gate writes per CMDB tree

A REST API admin's access profile is granted per **group**, not per object, and the group
a tree belongs to is not always the one its menu location suggests. The trap that bites:

- `/api/v2/cmdb/system/global` (the device `alias`, hostname and other global settings) is
  in the **System** group (`sysgrp`). `system/interface` — interface descriptions and
  addresses — is in **Network** (`netgrp`). A profile with Network → Configuration Read-Write
  and System read-only lets an interface-description PUT through and refuses the alias PUT
  with a 403. A feature that writes both then looks half-working rather than broken.
- System Read-Write also covers administrators, access profiles and every global setting.
  There is no narrower grant that reaches `system/global`, so a token that needs it is
  admin-grade; scope it with `trusthost`.
- The same split applies when the writer reaches the gate directly while a FortiManager
  manages it: the gate's own access profile authorizes the call, not FMG's admin profile.
- Prove a grant before depending on it: `GET <path>?action=schema` returns the table's
  definition including its `access_group`, which names the profile group for that build.
  A plain GET proves read access only; the write can still be refused.

## CMDB vs monitor

`cmdb` is configuration (what the operator set); `monitor` is state (what is happening). A
reservation is cmdb; a lease is monitor. A managed switch's port descriptions are cmdb; its
port link state is monitor. Keep the two apart in your model — a "who owns this address"
question is answered by cmdb, a "who is on this address right now" question by monitor.

## Admission and connectivity are two different fields

On both managed-device tables the controller reports **two independent axes**, and reading
either one as "is this device healthy" is wrong.

| | admission (has the operator authorized it) | connectivity (is the session up right now) |
|---|---|---|
| `managed-switch/status` | `state` — `"Authorized"` / `"Unauthorized"` | `status` — `"Connected"` / `"Disconnected"` |
| `wifi/managed_ap` | `state` — `"authorized"` / `"discovered"` / … | `status` — `"online"` / `"connected"` / `"offline"` / `"discovered"` |

A device is routinely authorized-but-offline (powered down, cable pulled) or
discovered-but-connected (plugged in, never admitted). Note the case difference between the
two tables, and that `"discovered"` appears as a value on BOTH axes of the AP row meaning
different things.

**The AP connectivity vocabulary varies by firmware**: most releases report `"online"`, some
report `"connected"`. Match both or a healthy fleet reads as entirely offline. The switch
table has no such variance — `"Connected"` is the only healthy value.

**A device absent from the table is not the same as a device reported down.** It ages out
briefly during a config push and after a controller reboot, so "not listed" is genuinely
unknown; treating it as down produces a burst of false alarms on every push. For the same
reason the `cmdb` roster is the stable answer to "which devices does this controller manage"
while the `monitor` status table is only the answer to "what is their state right now" — a
switch can be missing from the second while present in the first.

## IPsec monitor: tunnels and the peers hanging off them

`GET /api/v2/monitor/vpn/ipsec` returns one entry per phase-1 the IKE daemon is servicing
**and one per dynamic child** of a phase-1. A child carries `parent: "<phase-1 name>"` and is
named `<phase-1>_<n>`. Confirmed on FortiOS 7.6.7 (ADVPN hub) and 8.0.1 (ADVPN spoke).

- On a **hub**, a dial-up template (`vpn.ipsec/phase1-interface` `type: "dynamic"`) gets a
  child per connected peer — each ADVPN spoke, dial-up site or remote-access client. The
  template entry itself has an empty `proxyid` and `rgwy: "0.0.0.0"`; its byte counters are
  the aggregate. Children report `type: "dialup"`.
- On a **spoke**, a static tunnel with `auto-discovery-receiver` + `net-device` enabled gets a
  child per on-demand ADVPN shortcut. A quiet spoke has none — absence is normal.
- **The `n` in `<phase-1>_<n>` is a reusable slot** (`dialup_index`), not an identity: the
  same name can be a different peer after a reconnect. Key a peer on its address / identity.

Child fields worth knowing:

| field | meaning |
|---|---|
| `username` | the peer's IKE identity (its `localid`). When the peer sends none, FortiOS echoes its address here — the same value as `rgwy`. |
| `rgwy` | the peer's underlay / public address |
| `tun_id` | the peer's overlay (or mode-cfg-assigned) address inside the tunnel |
| `creation_time` | **seconds since the SA came up** (an uptime), not an epoch |
| `incoming_bytes` / `outgoing_bytes` | phase-1 totals; the per-selector counters are under `proxyid[]` |

Classify a child from its PARENT's cmdb entry, not from the child: `auto-discovery-sender:
enable` on a dynamic template → ADVPN spoke; `xauthtype` ≠ disable, `eap: enable`, or
`wizard-type: dialup-forticlient*` → remote-access user; a child of a `static` tunnel only
exists as an ADVPN shortcut. Remote-access children are expected to carry the authenticated
user in an `xauth_user`-style field — not yet confirmed on hardware, so read it defensively.

`GET /api/v2/monitor/vpn/ssl` lists SSL-VPN sessions (`user_name`, `remote_host`,
`subsessions[]` with the assigned `aip` and byte counts). It still answers on 7.6.7 with an
empty list after SSL-VPN tunnel mode was removed (7.6.3+), so an empty answer there is not a
read failure.

## Management access

`allowaccess` on a `system/interface` object lists the protocols that interface accepts
(`ping https ssh snmp http …`). Read it, do not infer it from open ports. A managed FortiSwitch
has no per-device `allowaccess`; the controller's
`switch-controller security-policy local-access` object carries `internal-allowaccess`
(and `mgmt-allowaccess` for the out-of-band port) for the whole fleet the controller manages.
When neither can be read, the honest value is **unknown**, not "no access".

## Quarantine by address group

MAC-based quarantine works by adding a MAC-typed `firewall/address` to a quarantine
`firewall/addrgrp` that a deny policy references, on **every gate that has recently seen the
device** (sightings from DHCP/endpoint tables). Push per gate, report per gate, and only flip
your own state when at least one gate accepted. Release removes the addresses; a failed
device-side removal is reported, never allowed to block the release. Verify by re-reading the
group later — drift happens when someone edits the gate by hand.

## Small-branch models

Branch-class gates (60F/61F/91G class) sometimes answer monitor endpoints slowly or partially
under load; use longer per-request timeouts for them, never fewer retries, and prefer the
FMG-side roster for reachability rather than hammering the gate.

## Per-device change tracking

Firmware (`os_ver`/`build`), a client's switch port, its AP, and its gateway are the four
changes operators want as events. Compute them from an end-of-run baseline against the stored
row, not phase by phase — coarse-then-corrected phases within one run ping-pong otherwise.

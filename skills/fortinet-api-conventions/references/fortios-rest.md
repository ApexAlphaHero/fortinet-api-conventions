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
- **The HTTPS certificate can change more than once after the upgrade reboot.** Seen on a 601E
  (FG6H1E, 7.6.7 build3704 → 7.6.8 build3744) running the default self-signed `FortiGate`
  certificate. The pre-upgrade cert was served until the reboot. A second cert answered when it
  came back. ~14 min later a THIRD one answered, and stayed: its NotBefore fell inside the boot,
  before the second was first seen, so it was minted then but not served first. A 61F upgraded the same hour on the same
  build changed once. A gate presenting a CA-issued cert it was given does not regenerate it. If
  you pin the leaf, accept every change for a while after an upgrade you ran, not only the first.
  Same address, same serial; nothing in the cert says which one is "final".

### Upgrading an HA cluster (a-p, FGCP)

Seen on a 61F pair, 7.6.7 build3704 → 7.6.8 build3744, `upgrade-mode uninterruptible`,
`override disable`, `session-pickup enable`, a reserved HA management interface on each member:

- **Upload once, to the primary**, with the same `firmware/upgrade` multipart call as a
  standalone gate. FortiOS pushes the image to the secondary and runs the order itself:
  secondary reboots onto it, failover, then the old primary reboots.
- **The upload request does not answer until the secondary is done and the cluster has failed
  over**: 242 s here, against ~8 s on a standalone gate. A short response timeout reports a
  failure for an upgrade that is working. The old primary reboots ~15 s after it answers.
- ~8 min end to end; each member was unreachable ~3 min on its own address.
- **The primary role returned to the original primary on its own**, ~3 min after it went down
  (override off, it had the higher priority). Don't assume which member ends up primary.
- `GET /api/v2/monitor/system/ha-peer` (from either member) lists every member's `serial_no`,
  `hostname`, `priority` and `primary` flag, but **no addresses and no versions**. Read each
  member's version from its own `monitor/system/status`, through its own address.
- **Reserved HA management (`ha-mgmt-status enable`, `ha-mgmt-interfaces`)** gives each member
  its own address that does not move at failover. The primary's API does not report the
  secondary's reserved address. The API admin, its token and its trusthost are synced, so one
  token reaches both members.
- **Certificates are partly per member.** `admin-server-cert` and imported local certs are
  synced, so a cluster serving an imported cert presents the same leaf from every member.
  The built-ins are not: `Fortinet_GUI_Server` (the 7.x default HTTPS cert, CN "FortiGate")
  differs per member, and `Fortinet_Factory` names each member's own serial. `Fortinet_SSL` was
  identical (synced). So on a SHARED address with the default cert, a failover changes the leaf
  that address presents. This upgrade regenerated none of the built-ins.
- A `monitor/system/certificate/download?mkey=Fortinet_Factory&type=local` made through the
  secondary's own address once returned the PRIMARY's factory cert (its serial), and the same
  read later returned the secondary's own. Don't identify a member by one such download.

## Reboot and restart (FortiGate, managed FortiSwitch, managed FortiAP)

Read off the gate's own schema (`GET /api/v2/monitor/?action=schema` → `directory[]`, one entry
per `path`/`name`/`action` with its `request.parameters` and `access_group`) on 7.6.7 build3704
and 8.0.1 build245, then exercised on a 61F, 2026-10:

| Action | Request | Access group |
|---|---|---|
| reboot the gate | `POST /api/v2/monitor/system/os/reboot` `{ "event_log_message": "…" }` (optional) | `sysgrp.cfg` |
| restart a FortiLink switch | `POST /api/v2/monitor/switch-controller/managed-switch/restart` `{ "mkey": "<switch-id>" }` | `wifi` |
| restart a managed AP | `POST /api/v2/monitor/wifi/managed_ap/restart` `{ "wtpname": "<wtp-id>" }` | `wifi` |

- A `GET` on an action path answers **405** (it exists) where an unknown path answers 404 — a
  side-effect-free way to probe that a build has the action.
- `mkey` is the switch's **name** (`switch-id`), which a FortiLink fleet may rename away from the
  serial. Look it up by serial in `GET …/managed-switch/status` (`serial`, `switch-id`, `status`
  `Connected`/`Idle`, `join_time`) first; `wtpname` is `wtp_id` in `GET …/wifi/managed_ap`.
- The gate reboot answers before it goes down (~16 s on a 61F, back ~2.5 min later); a dropped
  socket on the POST itself means "maybe taken", not "refused".
- **Proof it rebooted:** `GET /api/v2/monitor/web-ui/state` → `results.utc_last_reboot` (epoch
  ms). Compare it with the value read *before* the reboot, never with your own clock — a lab gate's
  clock ran ~7 min behind. A managed switch shows `Idle` on its gate while it restarts, then
  `Connected` with a new `join_time`.
- A reboot ends every admin session. A client signed in with username + password must sign in
  again afterwards — the old cookie is refused (401) on a gate that is fully back, which reads
  exactly like "still down" if refusals are treated as unreachability.
- An HA cluster member reboots alone and fails the cluster over; check `cmdb/system/ha` → `mode`.
- CLI equivalents (SSH): `execute reboot` (answers a `(y/n)` prompt),
  `execute switch-controller switch-action restart swtp <switch-id>`,
  `execute wireless-controller reset-wtp <serial|all|wtp-group>`. Over the CLI, `get system ha` →
  `mode : standalone` and `get system status` → `Serial-Number:` read the same facts.

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

### Interface state: `link`, `speed` and `duplex` are numbers and booleans, not words

`GET /api/v2/monitor/system/interface` reports each interface's runtime state as `link`
(boolean — the oper status), `speed` (number, Mb/s; `0` when the link is down) and `duplex`
(number: `1` on a full-duplex link that is up, `0` on every link that is down). Confirmed on
FortiOS 7.6.7 and 8.0.1. It carries no `status` at all, so the admin up/down comes from the
CMDB `system/interface` `status` ("up" / "down").

- Read `duplex` only while `link` is `true`. `0` is what a down link reports, so a reader that
  maps `0` to "half" without that gate reports every unplugged port as half duplex.
- On an up link, `0` can only mean half duplex. That reading follows from the encoding and
  has not been seen on a real half-duplex port.

## Detected devices: the field names moved, and a missing field is silent

`GET /api/v2/monitor/user/device/query` (the device inventory / detected devices) renamed
several fields across releases. On 7.x the row carries:

| meaning | 7.x field | older field |
|---|---|---|
| IPv4 address | `ipv4_address` | `ip` |
| OS family | `os_name` | `os` (or `type`) |
| interface the client was seen on | `detected_interface` | `interface` |
| user | `unauth_user` (or `user` when authenticated) | `user` / `detected_user` |
| FortiSwitch | `fortiswitch_id`, `fortiswitch_port_id` (number), `fortiswitch_port_name` (`"port3"`) | `switch_fortilink` / `fortiswitch`, `switch_port` |

Two traps compound here. Asking for a field the build doesn't have in `format=` is not an
error: the key is simply absent from every row. And a parser that reads the old name gets an
empty string, which looks like a legitimate "no address" rather than a bug. So request BOTH
spellings in `format=` (`ipv4_address|ip|…`) and read the new name first, falling back to the
old. The row is also the gate's *current* view of the client — built from traffic it sees,
with `is_online` and `last_seen` — which makes its address a stronger "where is this device
right now" answer than a DHCP lease, which only says what was handed out.

`last_seen` and `active_start_time` are Unix seconds. `is_online` is the gate's cached view and
can lag a client that just left. A client reached through a ZTNA access proxy gets a row on the
gate it connects *through*, with no switch / AP attribution: treat a row as local only when it
carries a FortiSwitch or FortiAP name, or the same MAC is in that gate's ARP table.

The row holds ONE address per MAC: a source address the gate saw traffic from. For a network
card with secondary addresses that is whichever one last talked, which proves the card has that
address and nothing about which is its primary — and FortiManager, proxying the same call,
repeats it unchanged. Cross-check against the gate's ARP table before trusting it: when ARP
has the MAC bound to a different address, or to several, treat the detected address as a hint
only. A detected address merely absent from ARP is not a contradiction — an idle neighbour
entry ages out within minutes.

## IPv6: where FortiOS keeps it, and where it does not

Read off FortiOS 7.6.7 and 8.0.1; FortiManager's device-DB copy of the same CMDB answers the
same fields with option values as integers (`0` / `1`) rather than `"disable"` / `"enable"`.

- **Interface addresses live in the CMDB, under a sub-object.** `cmdb/system/interface` rows
  carry `ipv6: { "ip6-mode", "ip6-address": "2001:db8::1/64", "ip6-extra-addr": [{ prefix }],
  "ip6-send-adv", "ip6-prefix-list": [{ prefix, … }], … }`. `ip6-address` is `"::/0"` when
  unset. An interface in `ip6-mode` dhcp / pppoe / delegated holds its address only at runtime,
  and it is not in this object. A projected FortiManager read must name `ipv6` in `fields` to
  get the block at all.
- **`monitor/system/interface` carries no IPv6 field.** Its keys stop at `ip` / `mask`; the
  CMDB block above is the only interface-IPv6 source over REST.
- **DHCPv6 has no reservations.** `cmdb/system.dhcp6/server` has `subnet` (the prefix,
  `"::/0"` until configured or when delegated), `ip-mode`, `ip-range`, `prefix-range`, and no
  `reserved-address` table. Every DHCPv6 binding is a lease, and there is nothing to push.
- **`monitor/system/dhcp?ipv6=true` returns the COMBINED table.** The IPv4 rows are unchanged
  and every row gains `type: "ipv4" | "ipv6"`, so adding the flag only adds rows. Ask for
  `type` in `format=` to tell the families apart.
- **The device inventory reports one `ipv6_address` per client** (`user/device/query`), and it
  is the only place a SLAAC or privacy address shows up — no DHCP table lists one. The
  FortiSwitch `detected-device` table has the same field, and its `last_seen` is an AGE in
  seconds (a live row read 7), unlike the inventory's Unix-seconds `last_seen`.
- **There is no IPv6 neighbour-cache endpoint** at `network/ipv6-neighbor`, `network/ndp`,
  `network/ipv6/neighbor`, `router/ipv6-neighbor` or `system/ipv6-neighbor-cache` (all 404);
  `network/arp?ipv6=true` still returns IPv4 rows only.

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
exists as an ADVPN shortcut.

**A remote-access child carries the user in `user`, not `username`** (confirmed on 7.6 with an
IKEv2 EAP dial-up template): `user` is the authenticated identity, beside `auth_type: "eap"`,
`user_two_factor_auth`, `fct_uid` (the FortiClient UID) and `rport`. `username` there holds the
client's own LAN address (behind NAT) — an address in `username` is never an identity, whether
it is that or a spoke's echoed `rgwy`. `auth_type` alone is enough to classify the child as
remote access, even when the template's cmdb flags were not readable. IKEv1 XAuth field names
remain unconfirmed.

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

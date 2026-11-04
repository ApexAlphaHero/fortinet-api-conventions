---
name: fortinet-api-conventions
description: "FortiManager JSON-RPC and FortiOS REST conventions and traps: api-key Bearer auth and why /sys/logout must never be called, RPC fault classes and retry policy, proxy vs direct device transport, /dvmdb rosters and HA serial flips, CMDB vs monitor reads, DHCP reservation/lease/VIP shapes, quarantine address groups, FortiSwitch and FortiAP SNMP quirks, the FortiSwitch / FortiAP local web-UI firmware-upgrade protocol (login forms, image header token = serial prefix, upload field order, deploy, reboot wait), the FortiGate admin session login (/api/v2/authentication) and REST firmware upgrade. Load whenever code calls a FortiManager, FortiGate, FortiSwitch or FortiAP API or MIB, or debugs an FMG/FortiOS error."
---

# Fortinet API conventions

Learned operating a fleet of ~190 FortiGates (mostly FortiManager-managed, some standalone),
~780 FortiAPs and their FortiSwitches from a Node.js management application. Vendor behavior
only; the app's own names are left out.

## The rules

1. **FortiManager auth is a permanent api-key over the `Authorization: Bearer` header.**
   FMG 7.4.7+ / 7.6.2+ removed `access_token` in the query string. A REST API Admin's api-key
   is permanent and shares ONE session per user — **never call `/sys/logout`** (or `/sys/login`):
   those are for session auth. A periodic logout tears the shared session out from under every
   other process using the key and shows up as RPC `-11` "no valid session" churn.
2. **Classify RPC faults before retrying.** Retry only transient faults (HTTP 5xx, network
   errors) with a small bounded backoff (≤2 retries), serialized per FMG. Fail fast on
   permanent ones: 401/403/404/405 and JSON-RPC `-11`. Retrying `-11` only amplifies the churn.
3. **Two transports to a managed gate, one narrow lane.** FMG's `/sys/proxy/json` relays
   `/api/v2/...` calls to a device through the manager; treat it as a **concurrency-1 lane per
   FMG** and keep native FMG reads (`/dvmdb`, CMDB) in a separate, wider lane. "Direct" mode uses
   FMG only to enumerate devices and talks REST to each gate with its own api-user token.
4. **The roster is `/dvmdb/adom/<adom>/device`.** It answers even for a gate that is offline
   (`conn_status !== 1`), and carries `sn`, `ha_slave[]`, connection state and platform. One
   roster read serves every gate — cache it per tick instead of asking per device.
5. **Identity is the serial, and in HA the serial is a SET.** FMG flips a cluster's top-level
   `sn` to whichever member is active, so compare against `{sn} ∪ ha_slave[].sn`, never a single
   value — or every failover reads as a chassis replacement. A device NAME cannot tell a rename
   from a replacement.
6. **CMDB is the config; monitor is the state.** In proxy mode read config natively from FMG's
   device database (works offline, lags) and live state (`/api/v2/monitor/...`) through the
   proxy (needs the gate reachable). Never take monitor output as configuration truth or vice
   versa. FMG's proxy also filters fields on some objects — see the reference.
7. **Reads that "succeed" with nothing are not answers.** A walk or list that returned an
   error must not be cached as "empty"; an unreadable roster means unknown, not "manages none".
   Distinguish absent / `[]` / populated everywhere you store what a gate manages.
8. **Write through the manager when the device is managed.** Under central management a
   device-side change must be mirrored into FMG's database with JSON-RPC `update` (never
   `set`, which replaces the object); otherwise the next install/push reverts it.
9. **Management access is a per-interface policy on the gate and a per-controller policy for
   switches.** A FortiGate's `allowaccess` lists what an interface accepts; a managed FortiSwitch
   has no profile of its own — its access comes from the controller's
   `switch-controller security-policy local-access` (`internal-allowaccess` /
   `mgmt-allowaccess`). Unknown must never render as "nothing permitted".
10. **FortiSwitch and FortiAP SNMP lie in specific, documented ways** (see the SNMP
    reference): `ifDescr` is the operator's port description, the FDB is indexed by row number,
    trunk names are truncated peer serials, and the FortiAP MIB carries no `UNITS` clause.
11. **A FortiSwitch or FortiAP flashes only through its OWN web UI** (see the local-HTTPS
    reference): username + password to `/login` (switch) or `/logincheck` (AP), the image
    header's platform token — the serial's first six characters — decides what an image is
    for, the staged-upload fields go in a fixed order, and a dropped socket on deploy is
    success. A FortiGate-managed AP usually has that UI disabled; there is no gate-side path.
12. **A FortiGate admin session is `POST /api/v2/authentication`, judged by its BODY** (see the
    REST reference): JSON `{username, password}`; `LOGIN_FAILED` still sets session and
    `ccsrf_token_<port>_<hash>` cookies, and `/logincheck` answers the login page on 7.6. A
    FortiGate flashes through `monitor/system/firmware/upgrade` (`source=upload`, multipart
    `file`); its `.out` is a gzip whose embedded name is the platform/version token.
13. **A managed FortiSwitch has no MAC in its own records, and is also a "client"** (see the REST
    reference, detected devices). `managed-switch/status` has no MAC; the detected-device
    `is_fortilink_peer` row is the far end of the FortiLink, never the switch. The gate's
    `user/device/query` lists the switch's own traffic with `hardware_family` "FortiSwitch"
    (request the field) — filter it out or every switch is found twice. The switch's MAC is
    what its own gate's lease / ARP shows at `connecting_from`.

## Which file

| Topic | Read |
|---|---|
| JSON-RPC envelope, auth, fault classes, lanes, `/dvmdb`, ADOMs, offline gates, proxy field filtering, mirroring writes | [references/fmg-jsonrpc.md](references/fmg-jsonrpc.md) |
| FortiOS REST: api-user tokens per gate, access-profile groups per CMDB tree, monitor vs cmdb paths, DHCP/ARP/system reads, managed-switch and managed-AP status, `allowaccess`, quarantine, per-model workarounds | [references/fortios-rest.md](references/fortios-rest.md) |
| the data shapes behind IP management: DHCP server scopes, reserved addresses, leases, VIPs, interface IPs, what is device-owned vs assignable, lease release | [references/dhcp-vip-shapes.md](references/dhcp-vip-shapes.md) |
| FortiSwitch / FortiAP SNMP: which tables, which quirks, which MIB objects have no units, trunk naming, PoE, VLAN bitmaps | [references/fortiswitch-fortiap-snmp.md](references/fortiswitch-fortiap-snmp.md) |
| FortiSwitch / FortiAP LOCAL web UI over HTTPS — the firmware-upgrade protocol (login forms, image header token = serial prefix, staged upload field order, compat check, deploy, progress, reboot wait, verify; why a dropped socket is success) | [references/fortiswitch-fortiap-local-https.md](references/fortiswitch-fortiap-local-https.md) |

## Parity rule

FortiManager-managed and standalone FortiGates talk to the same FortiOS via different
transports. Build push / release / quarantine / description-sync as a transport abstraction
dispatched on integration type, and ship every feature on both paths unless it is structurally
FMG-only (multi-device filters, ADOM scoping, proxy-lane tuning).

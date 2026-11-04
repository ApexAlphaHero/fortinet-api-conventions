# DHCP, VIP and interface-address shapes on FortiOS

The address facts a gate exposes, what each one MEANS for an IP registry, and how to write
them back.

## Sources and their meaning

| Source | Path | What it says | Ownership |
|---|---|---|---|
| DHCP server scope | `cmdb/system/dhcp/server` (`ip-range[]`, `netmask`, `default-gateway`, `interface`, `vci-*`) | the network the gate hands addresses out on | the subnet definition |
| Reserved address | `cmdb/system/dhcp/server/<id>/reserved-address[]` (`ip`, `mac`, `description`, `action`) | the gate will always give this MAC this IP | **assignable**: an operator decision the registry may own and push |
| Lease | `monitor/system/dhcp` (`ip`, `mac`, `hostname`, `expire_time`, `status`, `reserved`) | who holds the address right now | **observed presence**, never ownership; a registry entry may supersede it |
| Interface IP | `cmdb/system/interface` (`ip`, `secondaryip[]`) | the gate's own address on the network | **device-owned**: report, never edit or release |
| VIP | `cmdb/firewall/vip` (`extip`, `mappedip`) | a NAT'd address the gate answers for | **device-owned** |
| ARP entry | `monitor/network/arp` | L3 neighbor binding | evidence of presence, MAC-matched only |

Two facts, not one: **who owns an address** (manual entry vs discovered lease vs VIP) and
**how the gate serves it** (as a reservation, as a lease, or not at all). Store them in two
fields; discovery may flip the second without touching the first.

## Reservation identity and the placeholder MAC

A reservation needs a MAC. When the operator reserves before the device exists, generate a
**locally-administered unicast** placeholder (second hex digit 2/6/A/E) under one fixed prefix
and let a later ARP or device-inventory sighting replace it; the prefix is the only marker that
the MAC is synthetic, which keeps it visible on the gate's own reserved-address table. Reject
any placeholder prefix that is not locally-administered so no factory MAC can fall inside it.

## Writing a reservation

1. Find the scope: the server object whose `ip-range` contains the address (per VDOM).
2. Append to that object's `reserved-address` (`set` on the sub-object list; read → modify →
   write, because the list is replaced wholesale).
3. Under FortiManager, mirror the same change into FMG's CMDB with JSON-RPC `update`.
4. Read back and verify; treat a transient failure as **queued** with a retry tick and a retry
   hook on gate recovery, and let edit/release on a queued entry short-circuit locally.

## Releasing

- A lease: `POST monitor/system/dhcp/revoke` `{ ip: [...] }` — expires it; the client may
  re-lease. There is NO `release-lease` (or similar) path — a guessed monitor path answers 404
  "Endpoint not found" — and `ip` is an ARRAY even for one address; the reference describes a
  `revoke` with no `ip` as revoking every lease on the server, so never send it without one
  (confirmed on 7.6.8, 2026-11-03).
- A reservation: remove the `reserved-address` entry (device first, then FMG mirror).
- A VIP or interface IP: **refuse**. It is configured on the device; the registry only reports it.

## Collision rules that held up

- Creating over an observed lease is a plain create that supersedes the lease entry; creating
  over a reservation, VIP or interface IP is a conflict (409).
- A managed switch or AP that is decommissioned gives its reserved address back; one that
  merely stopped being discovered does not.
- The scope's `interface` plus the gate's serial identify which gate serves a subnet. A device
  name cannot tell a rename from an RMA swap; the serial (as an HA set) can.
- Address space several sites serve identically (a management VLAN, an OOB range) cannot be
  one row per CIDR; make it an explicit exclusion rather than a conflict that fires every run.

## Interface descriptions

Description sync (registry ↔ device port descriptions) is a policy choice; the workable one
is "a non-empty registry value always wins, an empty one adopts the device value", pushed on
save and re-asserted on every reconcile, mirrored into FMG under central management. No
conflict state is needed.

# FortiSwitch and FortiAP over SNMP

Direct SNMP (v2c/v3) to a managed FortiSwitch or FortiAP is the only source for several
facts the controller does not republish. It also has vendor-specific traps.

## FortiSwitch

- **`ifDescr` is the operator's port DESCRIPTION**, not a port name. Name a port from
  `ifName` and fall back to `ifDescr` only when `ifName` is absent; otherwise a walk whose
  `ifName` half failed renames `port9` to whatever is plugged into it, and that label becomes
  an identity in every table keyed by interface name. Reconcile collected names against the
  previous inventory: a real `ifName` wins, a label two ports claim resolves to nothing.
- **LLDP-MIB local port ids** (`lldpLocPortId` / `lldpLocPortDesc`) also carry the description
  on FortiSwitch; canonicalize the local interface through the `ifName` inventory before using
  it as a join key, or one neighbor appears under two names.
- **The FDB (Q-BRIDGE `dot1qTpFdbTable`, BRIDGE-MIB fallback) is indexed by row number, not by
  MAC** — read the address COLUMN. A table that answered but decoded to zero rows means
  "preserve what you had", not "empty".
- **Trunks** come from the vendor scalar `fsTrunkMember` (`1.3.6.1.4.1.12356.106.3.1.0`). An
  auto-created FortiLink/ICL trunk is named for the peer's serial **left-truncated to 15
  characters**; match a peer by SUFFIX, never reconstruct, and never resolve an ambiguous tail.
  Hand-configured LAGs carry operator names (`Storage Node 2`) — only a serial-shaped tail
  (alphanumeric, ≥10 chars) goes to the peer lookup. The ifTable publishes no aggregation, so
  this scalar is the only source of trunk → member edges on a directly polled switch.
- **The ifTable can name a trunk with a role suffix the scalar does not carry.** Observed on an
  FS-248E: `ifDescr` `1CD11A4549B7-0(primary)` and `_FlInK1_ICL0_(primary)` while
  `fsTrunkMember` says `1CD11A4549B7-0` / `_FlInK1_ICL0_`; `ifType` is `54` (propMultiplexor),
  not `161` (ieee8023adLag). Join trunk names with a trailing `(<word>)` stripped, and do not
  rely on `ifType` to recognize a trunk.
- A **FortiLink trunk interface leaves the ifTable while the link is down**. Preserve the row
  as "down" when the peer is still active and reciprocates; deleting it makes a downed uplink
  vanish at exactly the moment its down alert fires.
- **Switch-port VLANs**: Q-BRIDGE `dot1qVlanStaticUntaggedPorts` / `EgressPorts` bitmaps; a
  FortiSwitch echoes the untagged bitmap into egress, so derive tagged = egress − untagged.
  Controller CMDB is the fallback when SNMP lacks it.
- **PoE**: POWER-ETHERNET-MIB `pethPsePortTable` where present. Only a walk that ANSWERED may be
  cached as "no PSE"; an error `.catch()`-ed into an empty map blanks PoE for the whole cache TTL.
- **MCLAG** peers: the vendor ICL port table; `peerSn` is the pairing key.
- **ENTITY-MIB** (`entPhysicalTable`) gives transceivers/PSUs/fans with serial + model for RMAs.
  Do not correlate `entPhysical*` to interfaces by index; it is right often enough to look
  correct and silently wrong the rest of the time.
- **SFP DDM readings have no RFC 4133 link to their port** (FortiSwitchOS 7.x, verified on
  FSW-148F and FSR-112F-POE v7.6.5). Every ENTITY-SENSOR row's `entPhysicalContainedIn` is the
  chassis (1); `entAliasMappingTable` holds one row (the CPU port); `entPhysicalParentRelPos` is
  a running counter; `entPhysicalName` / `Descr` are positional labels that can be wrong about
  the reading's TYPE (the 112F names its two power-input truthvalue sensors `SFP1`/`SFP2`), and
  the 148F stops publishing `entPhysicalTable` rows after index 8 while sensors continue to 24.
  What IS reliable (matched value-for-value against `get switch modules status`): each SFP cage
  is five consecutive `entPhySensorTable` rows — celsius, volts, dBm, dBm, mAmps — with **Tx
  before Rx**; an empty cage still emits its five rows as `unavailable(2)`, so the k-th group is
  the k-th cage; the cages are the highest-numbered `portN` interfaces. A copper (RJ45) SFP has
  no DDM and reads `unavailable` like an empty cage while the port has link — never treat
  "unavailable cage on a link-up port" as a contradiction. Use `entPhySensorUnitsDisplay` to
  tell the rows apart, not the names. Module type (SX/LX/T, wavelength, part number) is not in
  SNMP at all; per-module `alarm_flags` / `warning_flags` are CLI-only.
- SNMP is often DISABLED on managed switches/APs by default; expect `allowaccess` without
  `snmp` and fall back to the controller's REST tables.

## FortiAP

- The vendor FortiAP MIB (`fapRadioTable`, `fapVapTable`) is the only source for CONFIGURED
  and MAXIMUM tx power, radio mode and country; the controller's `wifi/managed_ap` `radio[]`
  gives the operating percentage of the ceiling. **The MIB has no `UNITS` clause** — store the
  bare integers and render them without a unit rather than labeling them dBm on a guess;
  converting between percentage and the bare value needs a per-model ceiling you do not have.
- The two sources disagree on VAP NAMES (FortiOS publishes the VAP object name, the MIB only
  the SSID). Match VAP rows on **BSSID** first and treat the name as an attribute, or each
  source inserts its own row and deletes the other's every pass. Several VAPs commonly share
  one SSID, so SSID is never an identity.
- Merge the two sources PER COLUMN (a null from a source that does not collect a column must
  not erase the other's value); radio identity is not merged — a radio index absent from a
  scrape is gone along with its VAPs.
- Station tables (`AssetWirelessStation`-style) join to VAPs on BSSID.
- IF-MIB has no parent column, and FortiAP sub-interface names do not always repeat the
  parent's name: the sub-interfaces of `wlan01` / `wlan10` are `w01.N` / `w10.N`, and those of
  `br0` are `br.N`, beside `br0` itself. Matching the dotted base literally leaves them
  parentless. Map the abbreviated base back (`wNN` → `wlanNN`, `br` → `br0`) only when that
  parent is in the same scrape, and do not read the suffix as a VLAN id: that is not documented.

## General SNMP discipline (applies to both)

- An absent or error varbind decodes to **null, never 0**: `Number(null) === 0` turns every
  unpublished OID into a confident zero and suppresses fallbacks guarded on `== null`.
- ENTITY-SENSOR-MIB `entPhySensorOperStatus = unavailable(2)` maps to null, never alarm — an
  empty SFP cage is not a fault. Prefer the device's own alarm bit over a threshold on the
  reading.
- Serialize all collectors against one agent (`host:port`) through a FIFO gate with a bounded
  wait; one dead host's 60 s net-snmp timeout otherwise wedges every queued caller.
- `sysDescr` self-reports are enrichment, not ownership: they say what the device claims to
  be, never that anything holds it in an inventory.

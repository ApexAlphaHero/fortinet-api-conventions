# fortinet-api-conventions

A Claude Code plugin carrying the FortiManager / FortiOS / FortiSwitch / FortiAP API and MIB
knowledge. Vendor behavior only — no application-specific paths.

## Use it in another project

Clone it once, then point Claude Code at the clone:

```
git clone https://github.com/davidmoore-rogers/fortinet-api-conventions.git
claude --plugin-dir <path-to-clone>/fortinet-api-conventions
```

`git pull` in the clone picks up a new version (check `version` in `.claude-plugin/plugin.json`).

The skill `fortinet-api-conventions` auto-loads whenever code calls a FortiManager,
FortiGate, FortiSwitch or FortiAP API or MIB, or debugs an FMG/FortiOS error. Invoke by hand
with `/fortinet-api-conventions:fortinet-api-conventions`.

## Layout

```
.claude-plugin/plugin.json
skills/fortinet-api-conventions/SKILL.md
skills/fortinet-api-conventions/references/fmg-jsonrpc.md
skills/fortinet-api-conventions/references/fortios-rest.md
skills/fortinet-api-conventions/references/dhcp-vip-shapes.md
skills/fortinet-api-conventions/references/fortiswitch-fortiap-snmp.md
```

Bump `version` in `.claude-plugin/plugin.json` when the content changes materially.

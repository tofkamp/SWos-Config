# Writing a SWos-Config JSON file

This guide explains how to write the JSON config file that `config2swos.py` uploads to a MikroTik SwOS switch. It matches [`configs.schema.json`](./configs.schema.json) — if you use an editor with JSON Schema support (VS Code, for example), point it at that file and you'll get autocomplete and inline validation while you type.

I have included an manual for writing a configure file. You can find this at (./writing-a-config-file).

This is an example configuration file
```json
{
  "identity": "48poort",
  "hostname": "192.168.88.1",
  "has_POE": true,
  "vlans": [
    { "name": "beheer", "vlan_id": 1 },
    { "name": "kantoor", "vlan_id": 10 },
    { "name": "publiek", "vlan_id": 20 }
  ],
  "access_ports": [
    { "interface": 1, "name": "Port 1", "vlan_id": 10 },
    { "interface": 2, "name": "Port 2", "vlan_id": 10 }
  ],
  "trunk_ports": [],
  "bonding_ports": [
    {
      "interface1": 51,
      "interface2": 52,
      "name": "uplink2core",
      "vlans": [1, 10, 20]
    }
  ],
  "poe_ports": [
    { "port_id": 1, "poe_output": "off" },
    { "port_id": 2, "poe_output": "off" }
  ]
}
```


```
python config2swos.py --hostname 192.168.88.1 --configfile myconfig.json --password thepassword
```

## The basic shape

Every config file is a single JSON object with these top-level keys:

| Key | Required? | What it is |
|---|---|---|
| `identity` | **yes** | The name shown for the switch |
| `vlans` | **yes** | The list of VLANs that should exist |
| `access_ports` | **yes** | Ports with one untagged VLAN (a normal PC/AP port) |
| `trunk_ports` | **yes** | Ports carrying several tagged VLANs |
| `bonding_ports` | **yes** | LACP-bonded port pairs carrying several VLANs |
| `hostname` | no | Switch IP address (default `192.168.88.1`, or use `--hostname`) |
| `has_POE` | no | Set `true` if the switch is a PoE model (default `false`, or use `--has_POE`) |
| `poe_ports` | no | Per-port PoE settings (only used when `has_POE` is `true`) |

Even if a switch has no trunk ports or no bonded ports, you still need to include `"trunk_ports": []` and `"bonding_ports": []` — the script reads these keys unconditionally and will crash if they're missing.

`username` and `password` don't belong in this file. The switch always logs in as `admin`, and the password is passed on the command line with `--password`, never stored in the JSON.

A minimal skeleton looks like this:

```json
{
  "identity": "my-switch",
  "vlans": [],
  "access_ports": [],
  "trunk_ports": [],
  "bonding_ports": []
}
```

## VLANs

List every VLAN you want to exist on the switch:

```json
"vlans": [
  { "vlan_id": 10, "name": "kantoor" },
  { "vlan_id": 20, "name": "publiek" }
]
```

- `vlan_id` (required) — a number from 1 to 4094.
- `name` (optional) — if you leave it out, the switch just calls it `VLAN <id>`.
- `port_isolation`, `learning`, `mirror`, `igmp_snooping` (all optional booleans) — advanced per-VLAN switch settings. Leave these out unless you specifically need to change them; they default to `false`, `true`, `false`, `false` respectively.

You only need to list a VLAN once here — it doesn't matter how many ports use it.

## Access ports

An access port carries exactly one VLAN, untagged — the typical port for a PC, printer, or access point that doesn't do its own VLAN tagging.

```json
"access_ports": [
  { "interface": 1, "name": "Port 1", "vlan_id": 10 },
  { "interface": 2, "name": "Port 2", "vlan_id": 10 }
]
```

- `interface` (required) — the physical port number.
- `vlan_id` (required) — the VLAN this port belongs to. It's added to the `vlans` list automatically if you didn't already declare it there, but it's clearer to declare all your VLANs explicitly.
- `name` (optional) — a label for the port. It gets prefixed with the port number and cut to 16 characters by the switch, so keep it short.

You can also add any of the link settings described in [Port and link settings](#port-and-link-settings) below.

## Trunk ports

A trunk port carries multiple VLANs, tagged — typically used towards another switch, a router, or a hypervisor host that handles its own VLAN tagging.

```json
"trunk_ports": [
  {
    "interface": 49,
    "name": "to-router",
    "vlans": [10, 20, 50, 100]
  }
]
```

- `interface` (required)
- `vlans` (required) — the list of tagged VLANs on this trunk.
- `name` (optional)

### Hybrid trunk (one untagged VLAN + tagged VLANs)

If the same physical link also needs to carry one VLAN untagged (for example, a hypervisor host with a "native"/management VLAN plus several tagged VLANs), add `untagged_vlan`:

```json
{
  "interface": 49,
  "name": "to-esx",
  "vlans": [20, 50, 100],
  "untagged_vlan": 10
}
```

With `untagged_vlan` set, that VLAN is received untagged and is automatically added to the port's VLAN membership alongside the ones listed in `vlans` — you don't need to repeat it in `vlans` yourself.

Without `untagged_vlan`, the port only accepts tagged frames.

## Bonding ports (LACP)

A bonding port is a pair of physical ports combined into one LACP link-aggregation group, typically used for an uplink where you want redundancy and/or extra bandwidth.

```json
"bonding_ports": [
  {
    "interface1": 51,
    "interface2": 52,
    "name": "uplink2core",
    "vlans": [4, 10, 20, 50, 100, 101, 102, 103, 112, 201, 202, 220, 251, 254]
  }
]
```

- `interface1`, `interface2` (both required) — the two physical ports being bonded.
- `name` (required for bonding ports) — applied to both ports as-is.
- `vlans` (required) — the tagged VLANs carried over the bond.
- `untagged_vlan` (optional) — same idea as for trunk ports: adds one untagged VLAN on both member ports in addition to the tagged ones in `vlans`.

Both sides of the switch pair (or the switch and its upstream partner) need matching LACP configuration for a bond to come up.

## Port and link settings

These optional settings can be added to any `access_ports` or `trunk_ports` entry (and `mode`/`group_id` can also go on `bonding_ports` entries) to override the defaults:

| Key | Default | Meaning |
|---|---|---|
| `enabled` | `true` | Whether the port is switched on |
| `autoneg` | `true` | Auto-negotiate speed/duplex |
| `duplex` | `false` | `true` = half duplex, `false` = full duplex |
| `tx_flow_control` | `true` | Flow control, transmit direction |
| `rx_flow_control` | `false` | Flow control, receive direction |
| `speed` | `1000` | Link speed in Mbps: `10`, `100`, `1000`, `2500`, `5000`, or `10000` |
| `mode` | `passive` (access/trunk) / `active` (bonding) | LACP mode: `off`, `passive`, `active`, or `static` |
| `group_id` | — | LACP group id, only needed (and required) when `mode` is `static` |

Most of the time you can leave all of these out and just rely on the defaults.

## PoE ports

Only relevant if `has_POE` is `true`. List the PoE settings you want per port:

```json
"has_POE": true,
"poe_ports": [
  { "port_id": 1, "poe_output": "off" },
  { "port_id": 2, "poe_output": "auto" }
]
```

- `port_id` (required)
- `poe_output` (required) — one of `off`, `auto`, `low`, `high`.
- `priority` (optional, default `4`) — used if the switch's total power budget is exceeded.
- `lldp_enabled` (optional, default `false`).

A port not listed in `poe_ports` is left as-is.

## Putting it together

The example below is a trimmed-down version of a real 48-port switch config: three VLANs, two access ports, and one bonded uplink carrying all VLANs.

```json
{
  "identity": "48poort",
  "hostname": "192.168.88.1",
  "has_POE": true,
  "vlans": [
    { "name": "beheer", "vlan_id": 1 },
    { "name": "kantoor", "vlan_id": 10 },
    { "name": "publiek", "vlan_id": 20 }
  ],
  "access_ports": [
    { "interface": 1, "name": "Port 1", "vlan_id": 10 },
    { "interface": 2, "name": "Port 2", "vlan_id": 10 }
  ],
  "trunk_ports": [],
  "bonding_ports": [
    {
      "interface1": 51,
      "interface2": 52,
      "name": "uplink2core",
      "vlans": [1, 10, 20]
    }
  ],
  "poe_ports": [
    { "port_id": 1, "poe_output": "off" },
    { "port_id": 2, "poe_output": "off" }
  ]
}
```

## Common mistakes

- Forgetting `"trunk_ports": []` or `"bonding_ports": []` when you don't use them — the script needs the key to exist even if it's empty.
- Putting a `username` or `password` in the file — these are ignored/not read from JSON; use `--password` on the command line.
- Using the same `interface` number in more than one of `access_ports`, `trunk_ports`, or as `interface1`/`interface2` in `bonding_ports` — a physical port can only be configured once.
- Referencing a `vlan_id` in `access_ports`/`trunk_ports`/`bonding_ports` that isn't listed in `vlans` — the script will create it automatically with a default name, but it's better to declare it explicitly so you control the name and settings.

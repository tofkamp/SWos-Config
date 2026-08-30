This repository uses the mikrotik_swos library to make and upload a config to a switch.
It has been tested with smaller and larger switches (CRS354-48P-4S+2Q+__2.18).

In the JSON config file you can specify:
- Vlans
- Trunks (multiple tagged vlan's)
- Access ports (just 1 untagged vlan)
- Bondig ports (multiple vlan's with two LACP ports)

Things to be done
- Create JSON schema for config file
- Make mikrotik_swos a ready to use library (pipy?)
- Analyse comments of Claud over the library
- Give Claud this config file
- Support statitics

Future improvements
- swos to config
- checkmk agent with this library
- 

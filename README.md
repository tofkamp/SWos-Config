This repository uses my mikrotik_swos library to make and upload a config to a switch.
It has been tested with smaller and larger switches (CRS354-48P-4S+2Q+__2.18 and CRS328-24P-4S+_r2_2.18).

In the JSON config file you can specify:
- Vlans
- Trunks (multiple tagged vlan's and optional 1 untagged)
- Access ports (just 1 untagged vlan)
- Bondig ports (multiple vlan's with two LACP ports)
```
python3 config2swos --hostname 192.168.88.1 --configfile Configs/someswitch.json --password "" --has_Poe --debug --version
```
The python script will read the config file, creates API requests to the switch to set as desired.

Things to be done
- Make mikrotik_swos a ready to use library (pipy?)

Future improvements
- swos to config
 

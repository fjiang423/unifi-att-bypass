# Steps

## Backup
On controller: **Settings** > **Maintenance** > **Backup** > **Download Backup**

## Hardwire: 
`ONT -> USG.WAN, USG.LAN -> Switch, USG.VOIP -> AT&T_router.ONT`
  
 > For USG Pro: `ONT -> USG.WAN1, USG.LAN1 -> Switch, USG.LAN2 -> AT&T_router.ONT`

## Ensure we’re not binding the VOIP port to WAN2. (Skip if USG Pro)
On controller: Settings > Site > Services and verify **Configure VOIP port as WAN2 on UniFi Security Gateway** is unchecked. (If you don’t see this option you’re good to move on)

## Now we need to update our WAN network. 
On controller: **Settings** > **Networks** > **Edit** (Next to WAN). Check **Use VLAN ID** and enter 0 in the box to the right.

## Next create our LAN2 network.
On controller: **Settings** > **Networks** > **Create New Network** and enter the below options:
```
Name: LAN2
Purpose: Corporate
Parent Interface: LAN2
Gateway/Subnet: 192.168.254.1/24 (or whatever you prefer)
DHCP Mode: None
```

## Copy scripts 
Download from https://github.com/jaysoffian/eap_proxy. Use any SFTP client:
* Copy `eap_proxy.sh` (rename from  `eap_proxy_example.sh`) to `/config/scripts/post-config.d/eap_proxy.sh`
* Copy `eap_proxy.py` to `/config/scripts/eap_proxy.py`
Then `chown` to root, `chmod` to make it executable

## Config interfaces

> * Replace `set interfaces ethernet eth1 address 192.168.0.1/24` with your subnet range, I'm using `192.168.0.1/24`.
> * Also replace `aa:bb:cc:dd:ee:ff` with AT&T router MAC address (you should be able to find from Unifi controller), otherwise remove this line and add `--set-mac` run option in `eap_proxy.sh`

```
configure
set interfaces ethernet eth0 description WAN
set interfaces ethernet eth0 duplex auto
set interfaces ethernet eth0 firewall in name WAN_IN
set interfaces ethernet eth0 firewall local name WAN_LOCAL
set interfaces ethernet eth0 speed auto
set interfaces ethernet eth0 vif 0 address dhcp
set interfaces ethernet eth0 vif 0 description 'WAN VLAN 0'
set interfaces ethernet eth0 vif 0 dhcp-options default-route update
set interfaces ethernet eth0 vif 0 dhcp-options default-route-distance 210
set interfaces ethernet eth0 vif 0 dhcp-options name-server update
set interfaces ethernet eth0 vif 0 firewall in name WAN_IN
set interfaces ethernet eth0 vif 0 firewall local name WAN_LOCAL
set interfaces ethernet eth0 vif 0 mac 'aa:bb:cc:dd:ee:ff'
set interfaces ethernet eth1 address 192.168.0.1/24
set interfaces ethernet eth1 description LAN
set interfaces ethernet eth1 duplex auto
set interfaces ethernet eth1 speed auto
set interfaces ethernet eth2 description 'AT&T router'
set interfaces ethernet eth2 duplex auto
set interfaces ethernet eth2 speed auto
set service nat rule 5010 description 'masquerade for WAN'
set service nat rule 5010 outbound-interface eth0.0
set service nat rule 5010 protocol all
set service nat rule 5010 type masquerade
set system offload ipv4 vlan enable
commit
save
exit
```

> USG Pro has different set of network interfaces:
> 
> * USG: WAN - `eth0`, LAN - `eth1`, VOIP - `eth2`
> * USG Pro: LAN1 - `eth0`, LAN2 - `eth1`, WAN1 - `eth2`, WAN2 - `eth3`
> 
> We need to replace interface as:
> 
> | Source | USG Interface | USG Physical Port | USG Pro Interface | USG Pro Physical Port |
> | -- | -- | -- | -- | -- |
> | ONT | `eth0` | USG WAN | `eth2` | USG Pro WAN1 |
> | Switch | `eth1` | USG LAN | `eth0` | USG Pro LAN1 |
> | AT&T Router | `eth2` | USG VOIP | `eth1` | USG Pro LAN2 |
> 
> **Keep in mind this mapping, we need to replace any interface below for USG Pro setup**.
> *Note `set service nat rule 5010 outbound-interface eth0.0` will be `set service nat rule 5010 outbound-interface eth2.0`*

## Testing
`ssh` into USG, then do
`sudo python /config/scripts/eap_proxy.py --restart-dhcp --ignore-when-wan-up --ignore-logoff --ping-gateway --set-mac eth0 eth2`, without `--set-mac` if MAC address already configured.

> Change interface for USG Pro

You are expecting to see `eth0.0: restarting dhclient` (or `eth2.0` for USG Pro) as following example:
```
fjiang@USG:~$ sudo python /config/scripts/eap_proxy.py --restart-dhcp --ignore-when-wan-up --ignore-logoff --ping-gateway --set-mac eth0 eth2
[2019-04-10 10:03:44,992]: proxy_loop starting
[2019-04-10 10:04:13,929]: eth0.0: setting mac to 88:71:b1:45:31:f1
[2019-04-10 10:04:15,525]: eth2: 88:71:b1:45:31:f1 > 01:80:c2:00:00:03, EAPOL start (1) v2, len 0 > eth0
[2019-04-10 10:04:15,531]: eth0: 00:90:d0:63:ff:01 > 01:80:c2:00:00:03, EAP packet (0) v1, len 4, Failure (4) id 11, len 4 [0] > eth2
[2019-04-10 10:04:15,536]: eth0: 00:90:d0:63:ff:01 > 01:80:c2:00:00:03, EAP packet (0) v1, len 15, Request (1) id 12, len 15 [11] > eth2
[2019-04-10 10:04:15,540]: eth0: 00:90:d0:63:ff:01 > 88:71:b1:45:31:f1, EAP packet (0) v1, len 15, Request (1) id 12, len 15 [11] > eth2
[2019-04-10 10:04:15,860]: eth2: 88:71:b1:45:31:f1 > 01:80:c2:00:00:03, EAP packet (0) v2, len 22, Response (2) id 12, len 22 [18] > eth0
[2019-04-10 10:04:15,878]: eth0: 00:90:d0:63:ff:01 > 88:71:b1:45:31:f1, EAP packet (0) v1, len 6, Request (1) id 13, len 6 [2] > eth2
[2019-04-10 10:04:16,169]: eth2: 88:71:b1:45:31:f1 > 01:80:c2:00:00:03, EAP packet (0) v2, len 22, Response (2) id 12, len 22 [18] > eth0
[2019-04-10 10:04:16,481]: eth2: 88:71:b1:45:31:f1 > 01:80:c2:00:00:03, EAP packet (0) v2, len 92, Response (2) id 13, len 92 [88] > eth0
[2019-04-10 10:04:16,499]: eth0: 00:90:d0:63:ff:01 > 88:71:b1:45:31:f1, EAP packet (0) v1, len 1020, Request (1) id 14, len 1020 [1016] > eth2
[2019-04-10 10:04:16,785]: eth2: 88:71:b1:45:31:f1 > 01:80:c2:00:00:03, EAP packet (0) v2, len 6, Response (2) id 14, len 6 [2] > eth0
[2019-04-10 10:04:16,808]: eth0: 00:90:d0:63:ff:01 > 88:71:b1:45:31:f1, EAP packet (0) v1, len 1020, Request (1) id 15, len 1020 [1016] > eth2
[2019-04-10 10:04:17,090]: eth2: 88:71:b1:45:31:f1 > 01:80:c2:00:00:03, EAP packet (0) v2, len 6, Response (2) id 15, len 6 [2] > eth0
[2019-04-10 10:04:17,110]: eth0: 00:90:d0:63:ff:01 > 88:71:b1:45:31:f1, EAP packet (0) v1, len 1020, Request (1) id 16, len 1020 [1016] > eth2
[2019-04-10 10:04:17,396]: eth2: 88:71:b1:45:31:f1 > 01:80:c2:00:00:03, EAP packet (0) v2, len 6, Response (2) id 16, len 6 [2] > eth0
[2019-04-10 10:04:17,417]: eth0: 00:90:d0:63:ff:01 > 88:71:b1:45:31:f1, EAP packet (0) v1, len 193, Request (1) id 17, len 193 [189] > eth2
[2019-04-10 10:04:17,718]: eth2: 88:71:b1:45:31:f1 > 01:80:c2:00:00:03, EAP packet (0) v2, len 906, Response (2) id 17, len 906 [902] > eth0
[2019-04-10 10:04:17,735]: eth0: 00:90:d0:63:ff:01 > 88:71:b1:45:31:f1, EAP packet (0) v1, len 6, Request (1) id 18, len 6 [2] > eth2
[2019-04-10 10:04:18,023]: eth2: 88:71:b1:45:31:f1 > 01:80:c2:00:00:03, EAP packet (0) v2, len 902, Response (2) id 18, len 902 [898] > eth0
[2019-04-10 10:04:18,049]: eth0: 00:90:d0:63:ff:01 > 88:71:b1:45:31:f1, EAP packet (0) v1, len 6, Request (1) id 19, len 6 [2] > eth2
[2019-04-10 10:04:18,328]: eth2: 88:71:b1:45:31:f1 > 01:80:c2:00:00:03, EAP packet (0) v2, len 902, Response (2) id 19, len 902 [898] > eth0
[2019-04-10 10:04:18,344]: eth0: 00:90:d0:63:ff:01 > 88:71:b1:45:31:f1, EAP packet (0) v1, len 6, Request (1) id 20, len 6 [2] > eth2
[2019-04-10 10:04:18,633]: eth2: 88:71:b1:45:31:f1 > 01:80:c2:00:00:03, EAP packet (0) v2, len 484, Response (2) id 20, len 484 [480] > eth0
[2019-04-10 10:04:18,737]: eth0: 00:90:d0:63:ff:01 > 88:71:b1:45:31:f1, EAP packet (0) v1, len 69, Request (1) id 21, len 69 [65] > eth2
[2019-04-10 10:04:18,939]: eth2: 88:71:b1:45:31:f1 > 01:80:c2:00:00:03, EAP packet (0) v2, len 6, Response (2) id 21, len 6 [2] > eth0
[2019-04-10 10:04:18,969]: eth0.0: restarting dhclient
```

You should have internet now.

## Do `reboot now`

# Troubleshoot 
1. After USG rebooting still no internet? 
* Power cycle AT&T router should fix it.
* Or replug cable for router -> USG.

2. Sometimes, USG interface configuration may be reset by controller, apply interface config again to restore connection.

3. Use advanced config for Unifi to prevent manual update interface config, especially when power off or USG upgrade: https://help.ui.com/hc/en-us/articles/215458888#4
```json
{
  "interfaces": {
    "ethernet": {
      "eth0": {
        "description": "LAN",
        "duplex": "auto",
        "speed": "auto"
      },
      "eth1": {
        "description": "AT&T router",
        "duplex": "auto",
        "speed": "auto"
      },
      "eth2": {
        "description": "WAN",
        "duplex": "auto",
        "firewall": {
          "in": {
            "name": "WAN_IN"
          },
          "local": {
            "name": "WAN_LOCAL"
          }
        },
        "speed": "auto",
        "vif": {
          "0": {
            "address": [
              "dhcp"
            ],
            "description": "WAN VLAN 0",
            "dhcp-options": {
              "default-route": "update",
              "default-route-distance": "210",
              "name-server": "update"
            },
            "firewall": {
              "in": {
                "ipv6-name": "WANv6_IN",
                "name": "WAN_IN"
              },
              "local": {
                "ipv6-name": "WANv6_LOCAL",
                "name": "WAN_LOCAL"
              }
            },
            "mac": "88:71:b1:45:31:f1"
          }
        }
      }
    }
  },
  "service": {
    "nat": {
      "rule": {
        "5010": {
          "description": "masquerade for WAN",
          "outbound-interface": "eth2.0",
          "protocol": "all",
          "type": "masquerade"
        }
      }
    },
    "offload": {
      "ipv4": {
        "vlan": "enable"
      }
    }
  }
}
```

# Appendix

1. Interface Config for USG Pro
```
configure
set interfaces ethernet eth2 description WAN
set interfaces ethernet eth2 duplex auto
set interfaces ethernet eth2 firewall in name WAN_IN
set interfaces ethernet eth2 firewall local name WAN_LOCAL
set interfaces ethernet eth2 speed auto
set interfaces ethernet eth2 vif 0 address dhcp
set interfaces ethernet eth2 vif 0 description 'WAN VLAN 0'
set interfaces ethernet eth2 vif 0 dhcp-options default-route update
set interfaces ethernet eth2 vif 0 dhcp-options default-route-distance 210
set interfaces ethernet eth2 vif 0 dhcp-options name-server update
set interfaces ethernet eth2 vif 0 firewall in name WAN_IN
set interfaces ethernet eth2 vif 0 firewall local name WAN_LOCAL
set interfaces ethernet eth2 vif 0 mac '88:71:b1:45:31:f1'
set interfaces ethernet eth0 address 192.168.0.1/24
set interfaces ethernet eth0 description LAN
set interfaces ethernet eth0 duplex auto
set interfaces ethernet eth0 speed auto
set interfaces ethernet eth1 description 'AT&T router'
set interfaces ethernet eth1 duplex auto
set interfaces ethernet eth1 speed auto
set service nat rule 5010 description 'masquerade for WAN'
set service nat rule 5010 outbound-interface eth2.0
set service nat rule 5010 protocol all
set service nat rule 5010 type masquerade
set system offload ipv4 vlan enable
commit
save
exit
```

2. Testing command for USG Pro
```
sudo python /config/scripts/eap_proxy.py --restart-dhcp --ignore-when-wan-up --ignore-logoff --ping-gateway eth2 eth1
```

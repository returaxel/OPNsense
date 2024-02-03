**Additional information and inspiration:**
- [schnerring.net](https://schnerring.net/blog/opnsense-baseline-guide-with-vpn-guest-and-vlan-support/)
- [OPNsense Docs](https://docs.opnsense.org/manual/how-tos/wireguard-selective-routing.html)

## About
_Disclaimer: Not a networking guru. Do your homework before copy pasting!_

I spent too long getting this to work with the out of date & overly explanatory (I'm a dummy) information available, so here's a very "streamlined" setup of Selective Routing with WireGuard and Mullvad. This is just the steps needed to get it up and running. Don't expect to learn why it's working here.

Works with OPNsense 24.1_1

## Install WireGuard

Navigate to: `System > Firmware > Plugin`

- Install WireGuard

## Download Mullvad config - _I'll call it .conf_

1. Login mullvad.net & go to wireguard-config
1. Generate Key
2. Scroll down and select server
3. Select IPv4
4. Select Only IPv4
5. Configure Content Blocking
	- Personal preference, it changes the DNS server provided in .conf
6. Download .conf

**Additional Mullvad info**

These can be used as monitoring IP for gateway(s):

- [Mullvad - How to set up ad-blocking in our app](https://mullvad.net/it/blog/2021/5/27/how-set-ad-blocking-our-app)
   - 100.64.0.1 for Ad-blocking
   - 100.64.0.2 for Tracker-blocking
   - 100.64.0.3 for Ad- + Tracker-blocking.

- [Mullvad - Adding another layer: malware DNS blocking](https://mullvad.net/en/blog/2022/3/16/adding-another-layer-malware-dns-blocking)
   - 100.64.0.4 Malware blocking only
   - 100.64.0.5 Ad and malware blocking, no tracker blocking
   - 100.64.0.6 Tracker and malware blocking, no ad blocking
   - 100.64.0.7 Ad, tracker and malware blocking (“everything”)

## Setup the WireGuard tunnel

### WireGuard INSTANCE - _[interface] in .conf_

Navigate to: `VPN > WireGuard > Settings > Instances`

_Fields not mentioned = BLANK_

- ADD

| **Field**         | **Value**              |
| --------------- | ---------------------------- |
| **Name**            | Instance Name                |
| **Pub Key**         | Available on Mullvad page      |
| **Priv Key**        | In downloaded .conf        |
| **Port**            | 51820                        |
| **Tunnel Address**  | Address_In_Conf/32            |
| **Disable Routes**  | [x]                |

- Save (don't apply yet)

### WireGuard PEER - _[peer] in .conf_

Navigate to: `VPN > WireGuard > Settings > Peers`

- ADD

| **Field**               | **Value**              |
| ------------------- | ---------------------------- |
| **Name**                | Peer Name                    |
| **Pub Key**             | In downloaded .conf        |
| **Allowed IPs**         | 0.0.0.0/0                    |
| **Endpoint Address**    | In downloaded .conf        |
| **Endpoint Port**       | 51820                        |
| **Instance**            | The one you set up earlier   |
| **Keepalive internal**  | 25                           |

- Save and hit apply

Navigate to: `VPN > WireGuard > Settings > General`

- Enable WireGuard
- Verify tunnel is UP in VPN > WireGuard > Diagnostics

### Add an interface for the tunnel

Navigate to: `Interfaces > Assignments > Assign a new interface`

- Expand list and select the WireGuard interface
- Device **wg1**
   - ADD
   - SAVE (above)

- Click on the new interface (above)
   - Enable Interface: **[x]**
   - SAVE

### Add a gateway for the tunnel

Navigate to: `System > Gateways > Configuration`

- ADD

| **Field**                           | **Value**                                             |
| ------------------------------- | ------------------------------------------------------- |
| **Name**                            | GW name                                                 |
| **Interface**                       | wg1                                                     |
| **Address Family**                  | IPv4                                                    |
| **IP Address**                      | .conf > [interface] > address (-1)*                   |
| **Far Gateway**                     | **[x]**                                                 |
| **Disable Gateway Monitoring**      | **[ ]**                                               |
| **Monitor IP**                       | 10.64.0.1 or one of the DNS servers                      |

* If .conf address is xx.xx.xx.10/32 you can use xx.xx.xx.9 - i.e. remove the subnet mask and subtract one from the last segment. 

-  SAVE
-  APPLY

## Setup Firewall Rules
_This configuration is as barebones as they come, modify it to your liking_

Navigate to: `Firewall > Aliases`

- ADD

| **Field**             | **Value**                                      |
| ----------------- | ------------------------------------------------ |
| **Name**              | [selected hosts] - any name you want              |
| **Type**              | Host(s)                                          |
| **Content**           | Add the IP of each device you want to use WireGuard|

- SAVE
- APPLY


### FIRST rule: Redirect DNS traffic for [selected hosts]

Navigate to: `Firewall > Rules > Floating`

_This rule is basically optional, I use it for troubleshooting_

- ADD

| **Field**                | **Value**                        |
| -------------------- | ---------------------------------- |
| **Action**               | Pass                               |
| **Quick**                | **[x]**                            |
| **Interface**            | Interfaces for selected devices    |
| **Direction**            | In                                 |
| **TCP/IP Version**       | IPv4                               |
| **Protocol**             | TCP/UDP                            |
| **Source**               | [selected hosts]                   |
| **Destination**          | A Mullvad DNS server: 100.64.0.X    |
| **Dst Port Range**       | DNS                                |
| **Gateway**              | WG Gateway                         |
| **Show Advanced Features**| YES                                |
| **MATCH Local tag**      | NO_WAN_EGRESS                      |

- SAVE


### SECOND rule: Route [selected hosts] traffic through the tunnel

- ADD

| **Field**                | **Value**                        |
| -------------------- | ---------------------------------- |
| **Action**               | Pass                               |
| **Quick**                | **[x]**                            |
| **Interface**            | Interfaces for selected devices    |
| **Direction**            | In                                 |
| **TCP/IP Version**       | IPv4                               |
| **Protocol**             | Any                                |
| **Source**               | [selected hosts]                   |
| **Destination**          | Any                                |
| **Gateway**              | WG Gateway                         |
| **Show Advanced Features**| -                                  |
| **SET local tag**        | NO_WAN_EGRESS                      |

- SAVE

### THIRD rule: Kill switch
_Optional for people living on the edge_

- ADD 
- [OPNsense Docs: Kill Switch](https://docs.opnsense.org/manual/how-tos/wireguard-selective-routing.html#step-11-add-a-kill-switch-optional)

### NAT Rule: NAT WireGuard for [selected hosts]

Navigate to: `Firewall > NAT > Outbound`

- Change mode to **Hybrid outbound NAT rule generation**

- ADD

| **Field**                    | **Value**                                      |
| ------------------------ | ------------------------------------------------ |
| **Interface**                | WG interface                                     |
| **TCP/IP Version**           | IPv4                                             |
| **Protocol**                 | Any                                              |
| **Source**                   | [selected hosts]                                 |
| **Src Port**                 | Any                                              |
| **Destination**              | Any                                              |
| **Dst Port**                 | Any                                              |
| **Translation / Target**     | Interface Address                                |
| **SET local tag**            | NO_WAN_EGRESS                                    |


- SAVE
- APPLY to save all the firewall rules

## Verify It's working as intended

- Add a device IP to the [selected hosts] Alias
- Use [Mullvad Check](https://mullvad.net/en/check)
   - All three should be green

- API, Powershell
```
(curl https://am.i.mullvad.net/json).Content | ConvertFrom-Json
```

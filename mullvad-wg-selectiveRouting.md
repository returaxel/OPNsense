- [OPNsense Docs](https://docs.opnsense.org/manual/how-tos/wireguard-selective-routing.html)
- [OPNsense forum post](https://forum.opnsense.org/index.php?topic=38550.0) If you find errors please let everyone know! 😀

`Tested on OPNsense 25.7.2`

## About
This is the bare minimum needed to get it up and running on a "clean" OPNsense installation.
Don't expect to learn why it's working here.

## 1. Download Mullvad config - _I'll call it_ `.conf`

1. Login mullvad.net & go to wireguard-config
2. Generate Key
3. Scroll down and select server
4. Select IPv4
5. Select Only IPv4
6. Configure Content Blocking
   - Irrelevant here, changes the DNS server provided in `.conf`
7. Download `.conf`

## 2 WireGuard Configuration

### 2.1 Add a WireGuard INSTANCE

Navigate to: `VPN -> WireGuard -> Instances`

Toggle: `advanced mode` _top left_

- ADD

| **Field**           | **Value**                          |
| --------------- | ------------------------------ |
| **Name**            | instance_name |
| **Priv Key**        | value of_`PrivateKey`  |
| **Tunnel Address**  | value of_`Address`    |
| **Disable Routes**  | **[x]**                         |
| **Gateway**         | value of_`Tunnel_Address(-1)`[^1]           |

[^1]: "The IP you choose for the Gateway is essentially arbitrary; pretty much any unique IP will do. The suggestion here is for convenience and to avoid conflicts" See: [OPNsense-DOCS](https://docs.opnsense.org/manual/how-tos/wireguard-selective-routing.html#step-2-configure-the-wireguard-instance)

- SAVE  (_don't apply yet_)

### 2.2 Add a WireGuard PEER

Navigate to: `VPN -> WireGuard -> Peers`

- ADD

| **Field**               | **Value**                        |
| ------------------- | ---------------------------- |
| **Name**                | peer_name                    |
| **Pub Key**             | value of_`PublicKey`|
| **Allowed IPs**         | 0.0.0.0/0                    |
| **Endpoint Address**    | value of_`Endpoint` |
| **Endpoint Port**       | 51820                        |
| **Instance**            | `instance_name`   |
| **Keepalive internal**  | 25                           |

- SAVE
- Enable WireGuard: **[x]**
- **APPLY**

Navigate to: `VPN -> WireGuard -> Status`

- Verify `Status` is ✅[^2]

[^2]: This checkmark is expertly placed by me, not ChatGPT

### 3. Add an interface

Navigate to: `Interfaces -> Assignments -> Assign a new interface`

* Expand the `Device list`
   - Select your WireGuard interface: `wg1` 
   - ADD
   - SAVE
* Click on the interface
   - Enable Interface: **[x]**
   - SAVE

### 4. Add a gateway

Navigate to: `System -> Gateways -> Configuration`

- ADD

| **Field**                           | **Value**                                             |
| ------------------------------- | ------------------------------------------------------- |
| **Name**                            |`mullvad_gateway`                |
| **Interface**                       | `wg1`                              |
| **Address Family**                  | IPv4                                |
| **IP Address**                      | value of_`[Interface]Address`|
| **Far Gateway**                     | **[x]**                               |
| **Disable Gateway Monitoring**      | **[ ]**                                |
| **Monitor IP**                      | 10.64.0.x (DNS server in_`.conf`)[^3]  |

[^3]: Monitor IP can be hit or miss, disable gateway monitoring for now if it's not working. See: [OPNSense-DOCS-Note](https://docs.opnsense.org/manual/how-tos/wireguard-selective-routing.html#step-6-create-a-gateway)

-  SAVE
-  **APPLY**

## 5 Firewall configuration
_This configuration is as barebones as they come, modify it to your liking_

Navigate to: `Firewall -> Aliases`

- ADD

| **Field**             | **Value**                                          |
| ----------------- | ---------------------------------------------- |
| **Name**              | `selected_hosts`           |
| **Type**              | Host(s)                                        |
| **Content**           | Add the IP of devices to _Selectively Route_ |

- SAVE
- **APPLY**

### 5.1 First rule: Route `selected_hosts`

Navigate to: `Firewall -> Rules -> Floating`

- ADD

| **Field**                | **Value**                        |
| -------------------- | ---------------------------------- |
| **Action**               | Pass                               |
| **Quick**                | **[x]**                            |
| **Interface**            | Interface(s) where your `selected_hosts` live    |
| **Direction**            | In                                 |
| **TCP/IP Version**       | IPv4                               |
| **Protocol**             | Any                                |
| **Source**               | `selected_hosts`                   |
| **Destination**          | Any                                |
| **Gateway**              | `mullvad_gateway`                 |
| **Show Advanced Features** | `SHOW`                        |
| **SET local tag**_(Advanced)_| NO_WAN_EGRESS | 

Note: Local tag is for kill switch

- SAVE

### 5.2 Second rule: Kill switch

_Optional... for people living on the edge_

- [OPNsense Docs: Kill Switch](https://docs.opnsense.org/manual/how-tos/wireguard-selective-routing.html#step-11-add-a-kill-switch-optional)

### 5.3 NAT rule

Navigate to: `Firewall -> NAT -> Outbound`

- Change mode to: **Hybrid outbound NAT rule generation**

- ADD

| **Field**                    | **Value**                                      |
| ------------------------ | ------------------------------------------------ |
| **Interface**                | `wg1`              |
| **TCP/IP Version**           | IPv4               |
| **Protocol**                 | Any                |
| **Source**                   | `selected_hosts`   |
| **Src Port**                 | Any                |
| **Destination**              | Any                |
| **Dst Port**                 | Any                |
| **Translation / Target**     | Interface Address  |

- SAVE
- **APPLY** to save all the firewall rules

## Verify It's working as intended
_Important to check that your use case has all bases covered._

- Add a device to your `selected_hosts` Alias
- Use [Mullvad Check](https://mullvad.net/en/check)
   - All three should be green

- API Powershell
```
(curl https://am.i.mullvad.net/json).Content | ConvertFrom-Json
```

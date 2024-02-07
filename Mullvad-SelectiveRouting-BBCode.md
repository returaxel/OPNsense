[size=12pt]Additional information and inspiration:[/size]
[url=https://schnerring.net/blog/opnsense-baseline-guide-with-vpn-guest-and-vlan-support/]schnerring.net[/url]
[url=https://docs.opnsense.org/manual/how-tos/wireguard-selective-routing.html]OPNsense Docs[/url]

[size=12pt]About[/size]

This is basically the existing guide(s) without much explanation, straight to the point, for us who get cross eyed by walls of text. Don't expect to learn why it's working here.

[glow=green,2,300]This is the minimum needed to get it up and running on an unconfigured OPNsense host. [/glow]
[i]Tested with 24.1.1[/i]

Hopefully, it can be of help to someone and lets hope I never have to do BBcode formatting ever again 🤦

[hr]

[size=14pt][b]Install WireGuard[/b][/size]

[code]Navigate to: System > Firmware > Plugin[/code]
- Install WireGuard

[hr]

[size=14pt][b]Download Mullvad config[/b] - I'll call it .conf[/size]

1. Login mullvad.net & go to wireguard-config
2. Generate Key
3. Scroll down and select server
4. Select IPv4
5. Select Only IPv4
6. Configure Content Blocking
   - Personal preference, it changes the DNS server provided in .conf
7. Download .conf

[b]Additional Mullvad info[/b]

These can be used as monitoring IP for gateway(s):
- [url=https://mullvad.net/it/blog/2021/5/27/how-set-ad-blocking-our-app]Mullvad - How to set up ad-blocking in our app[/url]
   - 100.64.0.1 for Ad-blocking
   - 100.64.0.2 for Tracker-blocking
   - 100.64.0.3 for Ad- + Tracker-blocking.

- [url=https://mullvad.net/en/blog/2022/3/16/adding-another-layer-malware-dns-blocking]Mullvad - Adding another layer: malware DNS blocking[/url]
   - 100.64.0.4 Malware blocking only
   - 100.64.0.5 Ad and malware blocking, no tracker blocking
   - 100.64.0.6 Tracker and malware blocking, no ad blocking
   - 100.64.0.7 Ad, tracker and malware blocking (“everything”)

[hr]

[size=14pt][b]WireGuard Configuration[/b][/size]

[size=12pt][b]WireGuard INSTANCE[/b] - [interface] in .conf[/size]

[code]Navigate to: VPN > WireGuard > Settings > Instances[/code]

Fields not mentioned = BLANK / Default

- ADD
[code]
| Field            | Value                        |
| --------------- | ---------------------------- |
| Name            | Instance Name                |
| Pub Key         | The one you generated        |
| Priv Key        | In downloaded .config        |
| Port            | 51820                        |
| Tunnel Address  | AddressInConf/32            |
| Disable Routes  | CHECKED                |
| Gateway         | Tunnel_Address (-1)*          |
[/code]

* See note: [url=https://docs.opnsense.org/manual/how-tos/wireguard-selective-routing.html#step-2-configure-the-wireguard-instance]OPnsense Docs - wireguard-selective-routing[/url]

- Save (don't apply yet)

[size=12pt][b]WireGuard PEER[/b] - [peer] in .conf[/size]

[code]Navigate to: VPN > WireGuard > Settings > Peers[/code]

- ADD
[code]
| Field               | Value                        |
| ------------------- | ---------------------------- |
| Name                | Peer Name                    |
| Pub Key             | In downloaded .config        |
| Allowed IPs         | 0.0.0.0/0                    |
| Endpoint Address    | In downloaded .config        |
| Endpoint Port       | 51820                        |
| Instance            | The one you set up earlier   |
| Keepalive internal  | 25                           |
[/code]

- Save and hit apply

[code]Navigate to: VPN > WireGuard > Settings > General[/code]
- Enable WireGuard
- Verify tunnel is UP in VPN > WireGuard > Diagnostics

[hr]

[size=12pt][b]Add an interface[/b][/size]

[code]Navigate to: Interfaces > Assignments > Assign a new interface[/code]

- Expand list and select the WireGuard interface
- Device [b]wg1[/b]
   - ADD
   - SAVE (above)

- Click on the new interface (above)
   - Enable Interface: CHECKED
   - SAVE

[hr]

[size=12pt][b]Add a gateway[/b][/size]

[code]Navigate to: System > Gateways > Configuration[/code]

- ADD
[code]
| Field                           | Value                                             |
| ------------------------------- | ------------------------------------------------- |
| Name                            | GW name                                           |
| Interface                       | wg1                                               |
| Address Family                  | IPv4                                              |
| IP Address                      | .conf > [interface] > address (-1)*               |
| Far Gateway                     | CHECKED                                           |
| Disable Gateway Monitoring      | UNCHECKED                                         |
| Monitor IP                       | 10.64.0.1 or one of the DNS servers              |
[/code]

* If .conf address is xx.xx.xx.10/32 you can use xx.xx.xx.9 - i.e. remove the subnet mask and subtract one from the last segment.

-  SAVE
-  APPLY

[hr]

[size=14pt][b]Firewall configuration[/b][/size]
[i]This configuration is as barebones as they come, modify it to your liking[/i]

[code]Navigate to: Firewall > Aliases[/code]

- ADD
[code]
| Field             | Value                                          |
| ----------------- | ---------------------------------------------- |
| Name              | [selected hosts] - any name you want           |
| Type              | Host(s)                                        |
| Content           | Add the IP of each device you want to use WireGuard
[/code]

- SAVE
- APPLY

[size=12pt][b]FIRST rule: Route [selected hosts] traffic through the tunnel[/b][/size]

[code]Navigate to: Firewall > Rules > Floating[/code]

- ADD
[code]
| Field                | Value                          |
| -------------------- | ------------------------------ |
| Action               | Pass                           |
| Quick                | CHECKED                        |
| Interface            | Interface(s) where your [selected hosts] live
| Direction            | In                             |
| TCP/IP Version       | IPv4                           |
| Protocol             | Any                            |
| Source               | [selected hosts]               |
| Destination          | Any                            |
| Gateway              | WG Gateway                     |
|              Show Advanced Features                   |
| SET local tag        | NO_WAN_EGRESS                  |
[/code]

- SAVE

[size=12pt][b]SECOND rule: Kill switch[/b][/size]
[i]May not be needed depending on your configuration, better safe than sorry?[/i]

- [url=https://docs.opnsense.org/manual/how-tos/wireguard-selective-routing.html#step-11-add-a-kill-switch-optional]OPNsense Docs: Kill Switch[/url]

[size=12pt][b]THIRD rule: Route DNS traffic for [selected hosts][/b][/size]
[i]This rule is optional,use for troubleshooting or with port forwards.[/i]

- ADD
[code]
| Field                | Value                          |
| -------------------- | ------------------------------ |
| Action               | Pass                           |
| Quick                | CHECKED                        |
| Interface            | Interface(s) where your [selected hosts] live
| Direction            | In                             |
| TCP/IP Version       | IPv4                           |
| Protocol             | TCP/UDP                        |
| Source               | [selected hosts]               |
| Destination          | A Mullvad DNS server: 100.64.0.X
| Dst Port Range       | DNS                            |
| Gateway              | WG Gateway                     |
[/code]

- SAVE

[size=12pt][b]NAT Rule: NAT WireGuard for [selected hosts][/b][/size]

[code]Navigate to: Firewall > NAT > Outbound[/code]

- Change mode to [b]Hybrid outbound NAT rule generation[/b]

- ADD
[code]
| Field                    | Value                                          |
| ------------------------ | ---------------------------------------------- |
| Interface                | WG interface                                   |
| TCP/IP Version           | IPv4                                           |
| Protocol                 | Any                                            |
| Source                   | [selected hosts]                               |
| Src Port                 | Any                                            |
| Destination              | Any                                            |
| Dst Port                 | Any                                            |
| Translation / Target     | Interface Address                              |
[/code]

- SAVE
- APPLY to save all the firewall rules

[hr]

[size=14pt][b]Verify it's working as intended[/b][/size]

- Add a device IP to the [selected hosts] Alias
- Use [url=https://mullvad.net/en/check]Mullvad Check[/url]
   - All three should be green

- API, Powershell
[code]
(curl https://am.i.mullvad.net/json).Content | ConvertFrom-Json
[/code]

[glow=cyan,2,300]Thanks for reading![/glow]
[i]Please educate me where there are misstakes![/i]
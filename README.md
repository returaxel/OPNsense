# OPNsense selective routing
Note: Should work with just about any VPN provider - although they might have different gateway settings or ways of authenticating.

## Baseline configuration
https://github.com/returaxel/OPNsense/blob/main/mullvad-wg-selectiveRouting.md

* Wireguard tunnel with per host basis for opnsense selective routing
* Devices (IPs) added to alias will be routed through the tunnel

## Optional: Selected target domains/hosts routed thru tunnel
Prerequisites: Baseline
Then add the following

*  An alias of all the domains / hosts that should route thru the tunnel
*  A new floating rule and add the interfaces that it should work for:
*     IPv4, *, *, *, selectiveDst[alias], *, mullvad_gw, *, 4, <description>
* a new NAT rule:
*     WGinterface, any, *, selectiveDst[alias], *, Interface address, *, NO, <description>

**Floating**
<img width="1387" height="34" alt="image" src="https://github.com/user-attachments/assets/93cbe1ba-a4cc-4ee1-b2f6-2fb1b7a186a5" />

**NAT**
<img width="1352" height="37" alt="image" src="https://github.com/user-attachments/assets/b18718ba-2d20-4e2f-924c-b4e6477f5cf2" />

### Verify
* Add "am.i.mullvad.net" to the new alias.
* Browse to https://mullvad.net/en/check (2/3 green)

`DNS will leak unless you only use mullvad DNS tunneled. Not really an issue though depending on your setup.`

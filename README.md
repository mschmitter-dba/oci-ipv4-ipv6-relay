# IPv4-to-IPv6 Relay on OCI with socat and systemd

Companion files for [IPv4-to-IPv6 Relay for Home Services Behind CGNAT](https://blog.dieschmitterlinge.de/networking/network-home/oracle-free-tier-ipv4-ipv6-home-relay/) by Martin Schmitter.

A small dual-stack cloud VM forwards selected IPv4 connections to IPv6 services at home. The examples cover SSH to a Raspberry Pi and WireGuard to a home router. OCI is the host used in the article; another Linux VM with public IPv4 and working IPv6 can serve the same purpose.

## What is included

| File | Install location | Purpose |
| --- | --- | --- |
| [443_socat.service](443_socat.service) | /etc/systemd/system/443_socat.service | systemd service for the SSH relay |
| [443_socat](443_socat) | /etc/socat/443_socat | IPv4 TCP 443 to pi.dyn.foo.bar over IPv6 TCP 22 |
| [51820_socat_wireguard.service](51820_socat_wireguard.service) | /etc/systemd/system/51820_socat_wireguard.service | systemd service for the WireGuard relay |
| [51820_socat_wireguard](51820_socat_wireguard) | /etc/socat/51820_socat_wireguard | IPv4 UDP 51820 to corerouter.dyn.foo.bar over IPv6 UDP 51820 |

The four service and script files reproduce the published code blocks, with a final newline added. Hostnames are examples from the article: replace them before starting. The repository does not contain private keys, real endpoint configuration or an automated cloud deployment.

## Prerequisites

- A Debian/Ubuntu-style Linux relay host with systemd, Bash and socat at /usr/bin/socat.
- A public IPv4 endpoint on the relay and working IPv6 routing to the home services.
- Publicly resolvable AAAA records for the home endpoints, updated when the home IPv6 prefix changes.
- An existing SSH server and/or WireGuard endpoint at home. The relay does not configure either one.
- Appropriate OCI routing, Security List or NSG rules, plus host and home firewall rules.

## Prepare the files

```bash
git clone https://github.com/mschmitter-dba/oci-ipv4-ipv6-relay.git
cd oci-ipv4-ipv6-relay
sudo apt-get update
sudo apt-get install -y socat
```

Edit 443_socat and replace pi.dyn.foo.bar with your SSH endpoint's AAAA hostname. Edit 51820_socat_wireguard and replace corerouter.dyn.foo.bar with the WireGuard endpoint's AAAA hostname. Adjust listener and destination ports only if the matching firewall rules and client configurations are updated too.

TCP 443 must be available on the relay. It cannot also be used by an HTTPS listener on the same IPv4 address and port. SSH over TCP 443 remains SSH; it does not become HTTPS or bypass application-aware filtering.

## Install

Install only the relay you need. Review the edited scripts first; the commands below copy both examples without opening any firewall rules or starting them.

```bash
sudo install -d -o root -g root -m 0755 /etc/socat
sudo install -o root -g root -m 0755 443_socat /etc/socat/443_socat
sudo install -o root -g root -m 0755 51820_socat_wireguard /etc/socat/51820_socat_wireguard
sudo install -o root -g root -m 0644 443_socat.service /etc/systemd/system/443_socat.service
sudo install -o root -g root -m 0644 51820_socat_wireguard.service /etc/systemd/system/51820_socat_wireguard.service
sudo systemctl daemon-reload
```

The scripts must be executable and readable by the services' dynamic users. Keep the installed scripts root-owned and not writable by those users. The SSH service includes CAP_NET_BIND_SERVICE so its dynamic user can bind TCP 443. These are system services, managed with sudo systemctl, not systemctl --user.

## Networking and access

| Relay ingress | IPv6 destination | Application authentication |
| --- | --- | --- |
| TCP 443 | SSH host, TCP 22 | SSH authentication and host-key verification |
| UDP 51820 | Home router, UDP 51820 | Existing WireGuard peer keys |

Allow only the relay listeners you need in the OCI NSG or Security List and the relay host firewall. Restrict client source addresses where practical. Permit the relay's IPv6 source to reach the selected home endpoints and ports. The home services see the relay as the network source, so source-based logging and filtering there do not identify the original IPv4 client.

This is a userspace Layer 4 relay, not general NAT64, full network routing or a VPN terminator. Do not enable broad forwarding or open unrelated ports for these examples. Keep SSH key-based authentication and verify the home server's host key. WireGuard remains responsible for peer authentication and encryption; socat only forwards its packets.

## Start and inspect

After adapting the endpoints and firewall policy, start either or both services:

```bash
sudo systemctl enable --now 443_socat.service
sudo systemctl enable --now 51820_socat_wireguard.service
sudo systemctl status 443_socat.service 51820_socat_wireguard.service
sudo journalctl -u 443_socat.service -u 51820_socat_wireguard.service -f
sudo ss -lnptu
```

From an authorized external IPv4 client, test SSH with your actual relay hostname and home SSH account:

```bash
ssh -p 443 HOME_SSH_USER@RELAY_HOSTNAME
```

For WireGuard, use the relay's public IPv4 endpoint with UDP port 51820 in a test client's existing peer configuration. Validate an actual handshake and traffic at both ends. A UDP scan reporting open|filtered is not proof that WireGuard works. Test concurrent clients, reconnection, idle periods and a home IPv6/DNS change before relying on this relay.

## Changes and shutdown

After changing an installed service file, run sudo systemctl daemon-reload and restart the affected service. After editing only an installed wrapper script, restart that service; no daemon reload is needed. Update the installed copy under /etc/socat, not only the copy in your Git checkout.

```bash
sudo systemctl restart 443_socat.service
sudo systemctl restart 51820_socat_wireguard.service
```

To stop and disable a relay:

```bash
sudo systemctl disable --now 443_socat.service
sudo systemctl disable --now 51820_socat_wireguard.service
```

Remove any corresponding ingress permissions separately when a relay is retired.

## Limits and validation

- The article's -T 120 sets an inactivity timeout during data transfer. Idle SSH sessions can close; it is not a hard limit on concurrent children or a complete resource-exhaustion defence.
- The UDP example retains fork and reuseaddr from the article. Test its behaviour for your actual clients and socat version, especially with multiple WireGuard peers.
- Dynamic DNS is not seamless failover for established sessions. Test reconnection after an IPv6 prefix change and restart the affected relay when troubleshooting stale endpoints.
- There is no high availability or guaranteed throughput in this example. OCI capacity, pricing and Free Tier eligibility depend on your account and region.

The files were compared with the blog code blocks and the shell scripts were syntax-checked. No live cloud deployment or end-to-end SSH/WireGuard test was performed when creating this repository.

## References

- [Full walkthrough and architecture on the blog](https://blog.dieschmitterlinge.de/networking/network-home/oracle-free-tier-ipv4-ipv6-home-relay/)
- [socat manual distributed by Debian](https://manpages.debian.org/trixie/socat/socat.1.en.html)
- [systemd execution settings: DynamicUser and capabilities](https://www.freedesktop.org/software/systemd/man/latest/systemd.exec.html)
- [OCI Security Lists](https://docs.oracle.com/en-us/iaas/Content/Network/Concepts/securitylists.htm)
- [OCI Network Security Groups](https://docs.oracle.com/en-us/iaas/Content/Network/Concepts/networksecuritygroups.htm)

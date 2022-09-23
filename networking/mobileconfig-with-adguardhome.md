# mobileconfig setup with AdGuardHome
How to setup a mobileconfig profile to provide DNS-over-HTTP (DoH), DNS-over-TLS (DoT) and DNS-over-QUIC (DoQ) on MacOS using AdGuardHome.

## Requirements
- MacOS or iOS
- [AdGuardHome](https://github.com/AdguardTeam/AdGuardHome)
- (optional) SD-WAN such as [Nebula](https://github.com/slackhq/nebula/), [TailScale](https://github.com/tailscale/tailscale), or
  [WireGuard](https://github.com/WireGuard)
- (optional) [Traefik](https://github.com/traefik/traefik/) (reverse proxy for AdGuardHome)
- (optional) [Lego](https://github.com/go-acme/lego) for Let's Encrypt
- Linux hosting

## Reasoning (Why)
While Android supports Private DNS, it's left up to the VPN provider to reconfigure DNS after the configuration is made. When using Nebula, DNS has to
be public facing until Nebula [configures DNS on the client](https://github.com/slackhq/nebula/issues/318). Exposing DNS to the internet is really for
mobile device support.

## Gotchas (Read carefully!)

### DoH/DoT/DoQ cannot be configured in MacOS/iOS without using a mobileconfig
`networksetup` can set the DNS servers per interface. For example, this sets two DNS servers for Wi-Fi and Thunderbolt (Ethernet) connections.

```bash
networksetup -setdnsservers Wi-Fi 172.25.0.1 9.9.9.9
networksetup -setdnsservers "Thunderbolt Bridge" 172.25.0.1 9.9.9.9
```

However, `networksetup` accepts only IPs, not protocols like `tls://` or `https://`. DoH/DoT/DoQ cannot be configured in MacOS without using a mobileconfig.

```bash
networksetup -setdnsservers Wi-Fi tls://dns.a8n.tools:853 172.25.0.1 9.9.9.9
tls://dns.a8n.tools:853 is not a valid IP address. No changes were saved...
** Error: The parameters were not valid.
```

### AdGuardHome uses single IP for DoH and admin interface.
AdGuardHome supports multiple interfaces for the various protocols **except** for DoH. The HTTP server for DoH is shared with the HTTP server for
the admin interface (see [this comment](https://github.com/AdguardTeam/AdGuardHome/issues/741#issuecomment-934233825) in issue #741). A reverse proxy
is needed to:
1. separate the DoH and admin interfaces. AdGuardHome does not support 2FA and the admin interface should have better protections.
2. provide a listener for additional IPs to proxy to AdGuardHome.

### ClientIDs require additional TLS certificates
AdGuardHome supports ClientIDs to identify clients when using encrypted connections. The DoT and DoQ protocols use subdomains to identify the clients.
AdGuardHome states [wildcard certificates are required](https://github.com/AdguardTeam/AdGuardHome/wiki/Clients#clientid). That's not technically true
as the only requirement is to have all names included in a single TLS certificate. If there are a handfull of clients, using SANs will work the same as
wildcard certificates.

Traefik can request TLS certificates for any domain it proxies. The downside is that there is a
[one-to-one relationship](https://doc.traefik.io/traefik/https/acme/) between the Traefik `router` and the cert requested. If Traefik uses one route for
the admin interface and a second route for DoH, two certs will be issued. This conflicts with AdGuardHomes requirement for using one cert. Lego overcomes
this by using a wildcard certificate or SAN (subdomains).

## Procedure
The `mobileconfig` is located in AdGuardHome > Setup Guide > DNS Privacy. Before downloading, you should perform due diligence to make sure everything
works.

Here is the important sections of the AdGuardHome config.

```yaml
# admin and DoH bind IP
bind_host: 127.0.0.1
bind_port: 8081
# http_proxy is used for the AdGuardHome HTTP client to download filters and such.
http_proxy: ""
dns:
  # DoT/DoQ/DNS bind IPs
  bind_hosts:
    # Public IP
    - 1.1.1.1
    # SD-WAN IP
    - 172.25.0.1
  port: 53
  trusted_proxies:
    # Trust Traefik
    - 127.0.0.0/8
    - ::1/128
tls:
  enabled: true
  # Admin interface domain. Needs to match cert name
  server_name: dns.a8n.tools
  force_https: false
  port_https: 8444
  port_dns_over_tls: 853
  port_dns_over_quic: 784
  port_dnscrypt: 0
  dnscrypt_config_file: ""
  allow_unencrypted_doh: false
  strict_sni_check: false
  certificate_chain: ""
  private_key: ""
  certificate_path: /etc/adguardhome/tls/dns.a8n.tools/certificate.crt
  private_key_path: /etc/adguardhome/tls/dns.a8n.tools/privatekey.key
# Clients are configured in the web GUI
clients:
  persistent:
    - name: laptop-01
      tags:
        - os_macos
      ids:
        - laptop-01
# Useful for troubleshooting
verbose: false
log_file: /var/log/adguardhome/adguardhome.log
# Some log settings don't seem to work. Be careful not to fill up the disk.
log_max_backups: 3
log_max_size: 100
log_max_age: 2
log_compress: true
log_localtime: false
```


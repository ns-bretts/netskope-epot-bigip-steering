# Netskope EPoT Steering using BIG-IP to replace PAC files

This solution works in conjunction with [Netskope GRE configuration for BIG-IP](https://github.com/ns-bretts/netskope-gre-bigip). It's assumed you have already deployed GRE tunnel configuration and section **4.2. Explicit Proxy over Tunnel (EPoT)**.

The solution adds advanced steering options, that would typically be used in Proxy Auto-Configuration (PAC) files. This solution can be used when the Explicit Proxy configuration has been "hard coded" to a device/server or doesn't support PAC files.

It supports the following scenarios in the following order of precedence:

- Destination IP address matches a non-routable local network (RFC1918, Link-Local, CGNAT etc) specified in the "local_networks_dg" data-group.
  - The BIG-IP will act as the Explicit Proxy and process the HTTP/s request.

- The local DNS resolver has Split DNS zones where "A" records exist in both RFC1918 and non-RFC1918 space. The Domain or FQDN must be added to the "split_dns_dg" for the DNS resolution to occur.
  - The BIG-IP will perform a DNS resolution with the local DNS resolver.
  - If an "A" record is returned and the IP address matches the "local_networks_dg" data-group:
    - The BIG-IP will act as the Explicit Proxy and process the HTTP/s request.
  - If the "A" record does not match OR NXDOMAIN:
    - Send the HTTP/s request to the Netskope EPoT and process the HTTP/s request.

- Source IP addresses (10.1.1.1) or Networks (10.0.0.0/8) match the "source_ip_steering_dg".
   - IP or CIDR is this data-group with a value of "0" will be steered to Netskope EPoT.
   - IP or CIDR is this data-group with a value of "1" will be processed by the BIG-IP Explicit Proxy.

- Domains (example.com will match exact and .example.com) or FQDNs (www.example.com) match the "fqdn_steering_dg".
   - Any Domain or FQDN is this data-group with a value of "0" will be steered to Netskope.
   - Any Domain or FQDN is this data-group with a value of "1" will be processed by the BIG-IP Explicit Proxy.

Note: The default steering behaviour is to steer all traffic to Netskope EPoT.

## 1. Data-Groups

The following data-groups are required by the [iRule](https://github.com/ns-bretts/netskope-epot-bigip-steering/blob/main/netskope_steering_irule.tcl)

```
create ltm data-group internal local_networks_dg type ip
create ltm data-group internal split_dns_dg type string
create ltm data-group internal source_ip_steering_dg type ip
create ltm data-group internal fqdn_steering_dg type string

modify ltm data-group internal local_networks_dg records add { 10.0.0.0/8 { data RFC1918 } }
modify ltm data-group internal local_networks_dg records add { 100.64.0.0/10 { data RFC6598 } }
modify ltm data-group internal local_networks_dg records add { 169.254.0.0/16 { data RFC3927 } }
modify ltm data-group internal local_networks_dg records add { 172.16.0.0/12 { data RFC1918 } }
modify ltm data-group internal local_networks_dg records add { 192.0.0.0/24 { data RFC6890 } }
modify ltm data-group internal local_networks_dg records add { 192.0.2.0/24 { data RFC5737 } }
modify ltm data-group internal local_networks_dg records add { 192.168.0.0/16 { data RFC1918 } }
modify ltm data-group internal local_networks_dg records add { 198.18.0.0/15 { data RFC2544 } }
modify ltm data-group internal local_networks_dg records add { 198.51.100.0/24 { data RFC5737 } }
modify ltm data-group internal local_networks_dg records add { 203.0.113.0/24 { data RFC5737 } }
modify ltm data-group internal local_networks_dg records add { 240.0.0.0/4 { data RFC1112 } }
```

## 2. iRule

Copy/Paste the [iRule](https://github.com/ns-bretts/netskope-epot-bigip-steering/blob/main/netskope_steering_irule.tcl) into the BIG-IP. The Explicit Proxy configuration below assumes the iRule is called "netskope_steering_irule".

## 3. BIG-IP Explicit Proxy

Create the BIG-IP Explicit Proxy. The configuration below uses the same IP as the Netskope EPoT in section **4.2. Explicit Proxy over Tunnel (EPoT)** of [Netskope GRE configuration for BIG-IP](https://github.com/ns-bretts/netskope-gre-bigip). But the port has been changed to 8080. All Explicit Proxy traffic should target this Explicit Proxy IP:Port to take advantage of the advanced steering.

Change the DNS server and Explicit Proxy IP to match your environment.

```
create net dns-resolver proxy_dns_resolver forward-zones replace-all-with { . { nameservers replace-all-with { 10.1.30.31:53 10.1.30.32:53 } } }
create net tunnels tunnel proxy_tcp_tunnel { profile tcp-forward }
create ltm profile http proxy_http_profile defaults-from http-explicit oneconnect-transformations disabled explicit-proxy { default-connect-handling allow tunnel-name proxy_tcp_tunnel dns-resolver proxy_dns_resolver }
create ltm profile tcp proxy_tcp_profile { defaults-from f5-tcp-progressive syn-cookie-enable disabled }
create ltm virtual explicit_proxy_8080_vs { destination 10.254.2.200:8080 ip-protocol tcp profiles replace-all-with { proxy_tcp_profile proxy_http_profile } source-address-translation { type automap } vlans-enabled vlans replace-all-with { private } rules { netskope_steering_irule } description "Explicit Proxy" }
```

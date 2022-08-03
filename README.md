# Netskope EPoT Steering using BIG-IP to replace PAC files




## Data-Groups
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

## iRule


## BIG-IP Explicit Proxy

```
create net dns-resolver proxy_dns_resolver forward-zones replace-all-with { . { nameservers replace-all-with { 10.1.30.31:53 10.1.30.32:53 } } }
create net tunnels tunnel proxy_tcp_tunnel { profile tcp-forward }
create ltm profile http proxy_http_profile defaults-from http-explicit oneconnect-transformations disabled explicit-proxy { default-connect-handling allow tunnel-name proxy_tcp_tunnel dns-resolver proxy_dns_resolver }
create ltm profile tcp proxy_tcp_profile { defaults-from f5-tcp-progressive syn-cookie-enable disabled }
create ltm virtual explicit_proxy_8080_vs { destination 10.254.2.200:8080 ip-protocol tcp profiles replace-all-with { proxy_tcp_profile proxy_http_profile } source-address-translation { type automap } vlans-enabled vlans replace-all-with { private } rules { netskope_steering_irule } description "Explicit Proxy" }
```

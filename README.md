# Highload Software Architecture 8 Lesson 17 Homework

DDoS Attacks
---

## Project setup

The project contains a single [docker-compose.yaml](docker-compose.yaml) file for both attacker and defender setups.

The defender part consists of Wordpress/Apache server, MySQL database and Nginx reverse proxy. The performance of the defender setup is
measured using siege.

The attacker part consists of multiple containers, each of which is running a different type of attack.

## Attacks and Defenses

The Defender performance will be measured with:

```shell
siege -i -c 10 -t 10s http://localhost:8080
```

### No attack performed

The defender setup is running without any attack performed. The siege results are as follows:

```shell
Transactions:                   1858 hits
Failed transactions:               0
Availability:                 100.00 %
Response time:                  0.06 secs
Transaction rate:             173.48 trans/sec
Longest transaction:            0.40
Shortest transaction:           0.00
```

### UDP Flood

```shell
hping3 --rand-source --flood --udp nginx -p 8080
```

Siege results:

```shell
Transactions:                   1023 hits
Failed transactions:               0
Availability:                 100.00 %
Response time:                  0.10 secs
Transaction rate:              95.43 trans/sec
Longest transaction:            4.15
Shortest transaction:           0.00
```

The UDP flood attack is causing minor performance degradation. The performance drop by approximately 45%.

#### Defence approach: block UDP traffic on the firewall

Since it's not really possible use iptables in docker, this approach was emulated by allowing only `tcp` traffic on the `nginx` container
and using host network mode by the attacker container:

Nginx:

```yaml
    ports:
        - '8080:8080/tcp'
```

Attacker:

```yaml
    network_mode: host
    command: [ "--rand-source", "--flood", "--udp", "host.docker.internal", "-p", "8080" ]
```

Siege results:

```shell
Transactions:                   1410 hits
Availability:                 100.00 %
Failed transactions:               0
Response time:                  0.07 secs
Transaction rate:             135.45 trans/sec
Longest transaction:            0.71
Shortest transaction:           0.00

```

Results are much better now. The performance drop is only caused by the overall load on a host network.

### ICMP Flood

```shell
hping3 --rand-source --flood --icmp nginx -p 8080
```

Siege results:

```shell
Transactions:                   1073 hits
Availability:                 100.00 %
Failed transactions:               0
Response time:                  0.10 secs
Transaction rate:              98.35 trans/sec
Longest transaction:            3.11
Shortest transaction:           0.00
```

Same as with UDP flood, the ICMP flood attack is causing minor performance degradation. The performance drop by approximately 45%.
This is because ICMP attack also uses UDP protocol, and causes ICMP response packets to be sent back to the attacker.

The defence approach is the same as for UDP flood.

#### Defence approach: block UDP traffic on the firewall

Again, using host network mode by the attacker container:

```yaml
    network_mode: host
    command: [ "--rand-source", "--flood", "--icmp", "host.docker.internal", "-p", "8080" ]
```

Siege results:

```shell
Transactions:                   1404 hits
Availability:                 100.00 %
Failed transactions:               0
Response time:                  0.07 secs
Transaction rate:             131.83 trans/sec
Longest transaction:            0.61
Shortest transaction:           0.00
```

Having the same results as for UDP flood: the performance drop is only caused by the overall load on a host network.

### HTTP Flood

```shell
hping3 --rand-source --flood nginx -p 8080
```

Siege results:

```shell
Transactions:                   1405 hits
Availability:                 100.00 %
Failed transactions:               0
Response time:                  0.07 secs
Transaction rate:             137.34 trans/sec
Longest transaction:            0.64
Shortest transaction:           0.00
```

This attack causes no significant performance degradation. The performance drop by approximately 20%, same as previously caused by the
overall load on a host network by blocked UDP attacks.

Looks like host system resources and initial Defender server setup have sufficient capacity to handle this type of attack.

### Slowloris

Using 1024 concurrent connections, same as configured nginx worker processes, to overwhelm the defender server:

```shell
slowloris -s 1024 nginx -p 8080
```

Siege results:

```shell
Transactions:                      0 hits
Availability:                   0.00 %
Failed transactions:            1033
Successful transactions:           0
Transaction rate:               0.00 trans/sec
```

The Defender server is completely overwhelmed by this attack. Siege is not able to perform any requests.

#### Defence approach: Limiting the number of connections and requests from each IP address

The following configuration was added to the `nginx` container:

```nginx
http {
  ...
  limit_conn_zone $binary_remote_addr zone=addr:10m;
  limit_conn addr 10;
  limit_req_zone $binary_remote_addr zone=one:10m rate=100r/s;
  limit_req zone=one burst=5;
  ...
}
```

Now siege results **without attack** are:

```shell
Transactions:                   1009 hits
Availability:                  76.73 %
Failed transactions:             306
Response time:                  0.10 secs
Transaction rate:              96.00 trans/sec
Longest transaction:            0.39
Shortest transaction:           0.00
```

There is some performance degradation, because traffic from siege is also limited.

Siege results **under attack** are:

```shell
Transactions:                      0 hits
Availability:                   0.00 %
Failed transactions:            1033
Successful transactions:           0
Transaction rate:               0.00 trans/sec
```

Defence did not work. The attack is still overwhelming the server.

#### Defence approach: Closing slow connections

The following configuration was added to the `nginx` container, as an addition to the previous one:

```nginx
server {
  ...
  client_body_timeout 1s;
  client_header_timeout 1s;
  ...
}
```

Siege results **under attack** are:

```shell
Transactions:                   959 hits
Availability:                  85.55 %
Failed transactions:             162
Response time:                  0.10 secs
Transaction rate:              94.58 trans/sec
Longest transaction:            0.44
Shortest transaction:           0.00
```

The results are now as good as without attack. However, the countermeasures impacted the performance of benchmark, since it originates from
a single IP address.

### SYN Flood

```shell
hping3 --rand-source --flood -S nginx -p 8080
```

Siege results:

```shell
Transactions:                   710 hits
Availability:                 100.00 %
Failed transactions:               0
Response time:                  0.13 secs
Transaction rate:              65.02 trans/sec
Longest transaction:            3.25
Shortest transaction:           0.00
```

The attack causes a significant performance degradation. The performance drop by approximately 70%.

#### Defence approach: Drop excessive SYN traffic on the firewall

The attack can be mitigated by configuring firewall with something like:

```shell
iptables -A INPUT -p tcp --syn -m limit --limit 100/s --limit-burst 150 -j RETURN
iptables -A INPUT -p tcp --syn -j DROP
```

However, this approach was not tested, since it's not really possible use iptables in docker.

### Ping of Death

The Ping of Death attack is performed by sending a malformed ping packet to the target machine. The packet size is larger than the maximum
allowed 65,535 bytes. The target machine then tries to reassemble the malformed packet, causing a buffer overflow.

```shell
ping -s 65507 nginx
```

However, modern operating systems are not vulnerable to this attack. The attack makes no impact on the Defender server.

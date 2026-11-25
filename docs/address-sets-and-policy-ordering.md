# Address Sets, Nested Address Sets, and Security Policy Ordering

## 1. Lab purpose

This experiment builds on the earlier [SRX Advanced Security Policies](https://github.com/emadalshishani/SRX-Advanced-Security-Policies) project.

The previous project created an Ubuntu-specific DNS deny policy. This lab reuses that existing policy and the same SRX environment to explore:

- Global Address Book organization.
- Address Objects and Address Sets.
- Nested Address Sets.
- Address Set references in source and destination policy matching.
- Policy order, first-match behavior, and policy shadowing.
- Verification using Ubuntu and a second test host.

## 2. Lab environment

| Component | Details |
|---|---|
| Firewall | HQ Juniper SRX (`HQ`) |
| Source zone | `internal` |
| Destination zone | `branch` |
| Layer 3 switch | `L3_SW` |
| PC1 | Ubuntu, `192.168.30.2/32`, VLAN 30 |
| PC2 | `192.168.10.3`, VLAN 10 |
| VLAN 10 network | `192.168.10.0/29` |
| VLAN 20 network | `192.168.20.0/29` |
| VLAN 30 network | `192.168.30.0/29` |
| DNS primary | `8.8.8.8/32` |
| DNS secondary | `8.8.4.4/32` |

The SRX-to-switch transit network is `10.10.10.0/30`.

![SRX Address Books, Nested Address Sets, and Policy Ordering Topology](images/SRX%20AddressBook-Sets%20%26%20Nested-AddressBook%20-Advanced%20Policy%20Topology.svg)

The diagram provides the visual reference for the devices, VLAN networks, and Address Set relationships used throughout this lab.

## 3. Relationship to the previous repository

In [SRX-Advanced-Security-Policies](https://github.com/emadalshishani/SRX-Advanced-Security-Policies), the following policy was created:

`internal-branch_ubuntu-user-test`

Its relevant behavior was:

- Source address: `ubuntu-user-ip` (`192.168.30.2/32`).
- Destination address: `any`.
- Applications: `junos-dns-udp` and `junos-dns-tcp`.
- Action: `deny`.
- Logging: `session-init` and `session-close`.

This earlier policy is deliberately retained in the environment. It became relevant when the new DNS permit policy was initially placed below it.

## 4. Global Address Book configuration

### 4.1 Address Objects

The following Address Objects were configured:

~~~bash
set security address-book global address ubuntu-user-ip 192.168.30.2/32
set security address-book global address HQ-VLAN10-NET 192.168.10.0/29
set security address-book global address HQ-VLAN20-NET 192.168.20.0/29
set security address-book global address DNS-PRIMARY 8.8.8.8/32
set security address-book global address DNS-SECONDARY 8.8.4.4/32
~~~

The Ubuntu object already existed from the previous project. It is included to show the complete logical Address Book design.

### 4.2 Child Address Sets

~~~bash
set security address-book global address-set VLAN10-SOURCES address HQ-VLAN10-NET
set security address-book global address-set VLAN20-SOURCES address HQ-VLAN20-NET
set security address-book global address-set VLAN30-TEST-HOSTS address ubuntu-user-ip
~~~

### 4.3 Parent / Nested Address Set

~~~bash
set security address-book global address-set HQ-SOURCE-GROUP address-set VLAN10-SOURCES
set security address-book global address-set HQ-SOURCE-GROUP address-set VLAN20-SOURCES
set security address-book global address-set HQ-SOURCE-GROUP address-set VLAN30-TEST-HOSTS
~~~

The parent `HQ-SOURCE-GROUP` contains three child Address Sets:

| Child Address Set | Member |
|---|---|
| `VLAN10-SOURCES` | `HQ-VLAN10-NET` – `192.168.10.0/29` |
| `VLAN20-SOURCES` | `HQ-VLAN20-NET` – `192.168.20.0/29` |
| `VLAN30-TEST-HOSTS` | `ubuntu-user-ip` – `192.168.30.2/32` |

PC2 (`192.168.10.3`) belongs to the VLAN 10 network member. PC1 (Ubuntu, `192.168.30.2`) belongs to the VLAN 30 host member.

### 4.4 Destination Address Set

~~~bash
set security address-book global address-set PUBLIC-DNS-SERVERS address DNS-PRIMARY
set security address-book global address-set PUBLIC-DNS-SERVERS address DNS-SECONDARY
~~~

The `PUBLIC-DNS-SERVERS` set contains:

- `DNS-PRIMARY`: `8.8.8.8/32`
- `DNS-SECONDARY`: `8.8.4.4/32`

### 4.5 Configuration verification

The Address Book configuration was checked with:

~~~bash
show security address-book
~~~

The displayed output confirmed the five Address Objects, three child Address Sets, the parent `HQ-SOURCE-GROUP`, and the `PUBLIC-DNS-SERVERS` destination group.

## 5. DNS permit policy using Address Sets

The following policy uses the nested source group and the DNS destination group:

~~~bash
set security policies from-zone internal to-zone branch policy internal-branch_ubuntu-user-address_set_test match source-address HQ-SOURCE-GROUP
set security policies from-zone internal to-zone branch policy internal-branch_ubuntu-user-address_set_test match destination-address PUBLIC-DNS-SERVERS
set security policies from-zone internal to-zone branch policy internal-branch_ubuntu-user-address_set_test match application junos-dns-udp
set security policies from-zone internal to-zone branch policy internal-branch_ubuntu-user-address_set_test match application junos-dns-tcp
set security policies from-zone internal to-zone branch policy internal-branch_ubuntu-user-address_set_test then permit
set security policies from-zone internal to-zone branch policy internal-branch_ubuntu-user-address_set_test then log session-init
set security policies from-zone internal to-zone branch policy internal-branch_ubuntu-user-address_set_test then log session-close
~~~

### 5.1 Initial behavior

Before the new DNS permit policy was applied, Ubuntu traffic to the DNS servers was being denied by the previous project policy. The observed pre-change results were:

~~~text
$ ping 8.8.8.8
3 packets transmitted, 3 received, 0% packet loss

$ nslookup 8.8.8.8
;; communications error to 127.0.0.53#53: timed out
;; communications error to 127.0.0.53#53: timed out

$ nslookup 8.8.4.4
;; communications error to 127.0.0.53#53: timed out

$ curl -I https://google.com
curl: (6) Could not resolve host: google.com
~~~

The existing deny policy was:

`internal-branch_ubuntu-user-test`

The new DNS permit rule had initially been placed below that deny policy. The earlier rule matched the Ubuntu DNS traffic first, so the permit policy was not reached.

### 5.2 Correcting the policy order

The DNS permit policy was inserted before the previous Ubuntu DNS deny policy:

~~~bash
insert security policies from-zone internal to-zone branch policy internal-branch_ubuntu-user-address_set_test before policy internal-branch_ubuntu-user-test
~~~

After the insertion, the Ubuntu tests showed successful reverse DNS responses:

~~~text
$ nslookup 8.8.8.8
8.8.8.8.in-addr.arpa    name = dns.google.

$ nslookup 8.8.4.4
4.4.8.8.in-addr.arpa    name = dns.google.
~~~

The HTTPS request to Google also succeeded:

~~~text
$ curl -I https://google.com
HTTP/2 301
location: https://www.google.com/
content-type: text/html; charset=UTF-8
server: gws
~~~

**Interpretation:** the old Ubuntu DNS deny policy no longer took precedence over the new DNS permit policy. DNS lookups and the HTTPS request succeeded after the policy order was corrected.

**Testing note:** `nslookup 8.8.8.8` and `nslookup 8.8.4.4` are reverse DNS lookups. The successful `curl` request for `google.com` provides the accompanying test that the hostname could be resolved and an HTTPS request could be made.

## 6. ICMP deny policy using Address Sets

A separate policy was created to deny ping traffic from the parent source group to the public DNS server group:

~~~bash
set security policies from-zone internal to-zone branch policy internal-branch_ubuntu-user-address_set_test_ping_Deny match source-address HQ-SOURCE-GROUP
set security policies from-zone internal to-zone branch policy internal-branch_ubuntu-user-address_set_test_ping_Deny match destination-address PUBLIC-DNS-SERVERS
set security policies from-zone internal to-zone branch policy internal-branch_ubuntu-user-address_set_test_ping_Deny match application junos-ping
set security policies from-zone internal to-zone branch policy internal-branch_ubuntu-user-address_set_test_ping_Deny then deny
set security policies from-zone internal to-zone branch policy internal-branch_ubuntu-user-address_set_test_ping_Deny then log session-init
set security policies from-zone internal to-zone branch policy internal-branch_ubuntu-user-address_set_test_ping_Deny then log session-close
~~~

### 6.1 Why the first ICMP test did not block ping

The environment already contained this general permit policy:

~~~text
policy internal-branch {
    match {
        source-address any;
        destination-address any;
        application any;
    }
    then {
        permit;
    }
}
~~~

The ICMP deny policy was initially below `internal-branch`. The general policy matched the traffic first and permitted it, so the specific ICMP deny rule was not reached.

PC2 (`192.168.10.3`) initially continued to receive ping replies from `8.8.8.8`.

### 6.2 Correcting the ICMP policy order

The specific ICMP deny rule was inserted before the general `internal-branch` permit policy:

~~~bash
insert security policies from-zone internal to-zone branch policy internal-branch_ubuntu-user-address_set_test_ping_Deny before policy internal-branch
~~~

### 6.3 PC2 test results

Before the insertion, PC2 received successful replies:

~~~text
HQ-PC2> ping 8.8.8.8 -t

84 bytes from 8.8.8.8 icmp_seq=1 ttl=109 time=53.082 ms
84 bytes from 8.8.8.8 icmp_seq=2 ttl=109 time=53.771 ms
84 bytes from 8.8.8.8 icmp_seq=3 ttl=109 time=54.114 ms
84 bytes from 8.8.8.8 icmp_seq=4 ttl=109 time=54.337 ms
84 bytes from 8.8.8.8 icmp_seq=5 ttl=109 time=53.205 ms
84 bytes from 8.8.8.8 icmp_seq=6 ttl=109 time=53.015 ms
84 bytes from 8.8.8.8 icmp_seq=7 ttl=109 time=53.835 ms
84 bytes from 8.8.8.8 icmp_seq=8 ttl=109 time=58.659 ms
84 bytes from 8.8.8.8 icmp_seq=9 ttl=109 time=54.892 ms
84 bytes from 8.8.8.8 icmp_seq=10 ttl=109 time=53.920 ms
~~~

After the insertion, the continuous ping test showed replies initially, followed by timeouts:

~~~text
HQ-PC2> ping 8.8.8.8 -t

84 bytes from 8.8.8.8 icmp_seq=1 ttl=109 time=53.808 ms
84 bytes from 8.8.8.8 icmp_seq=2 ttl=109 time=54.495 ms
84 bytes from 8.8.8.8 icmp_seq=3 ttl=109 time=54.893 ms
84 bytes from 8.8.8.8 icmp_seq=4 ttl=109 time=54.722 ms
84 bytes from 8.8.8.8 icmp_seq=5 ttl=109 time=54.144 ms
84 bytes from 8.8.8.8 icmp_seq=6 ttl=109 time=55.112 ms
84 bytes from 8.8.8.8 icmp_seq=7 ttl=109 time=55.333 ms
8.8.8.8 icmp_seq=8 timeout
8.8.8.8 icmp_seq=9 timeout
8.8.8.8 icmp_seq=10 timeout
8.8.8.8 icmp_seq=11 timeout
8.8.8.8 icmp_seq=12 timeout
8.8.8.8 icmp_seq=13 timeout
8.8.8.8 icmp_seq=14 timeout
~~~

The test was performed while the ping process was running, so the output records the transition from successful replies to timeouts during the policy-order change.

### 6.4 Ubuntu verification after the ICMP deny policy

~~~text
$ ping 8.8.8.8
2 packets transmitted, 0 received, 100% packet loss

$ nslookup 8.8.8.8
8.8.8.8.in-addr.arpa    name = dns.google.

$ curl -I https://google.com
HTTP/2 301
location: https://www.google.com/
content-type: text/html; charset=UTF-8
server: gws
~~~

The observed results show that ping to the DNS server address was blocked while DNS reverse lookup and HTTPS access continued to work.

## 7. Policy ordering and policy shadowing

Two separate policy-order issues were identified and corrected.

| Test case | Specific policy | Earlier policy that took precedence | Correct insertion |
|---|---|---|---|
| DNS permit | `internal-branch_ubuntu-user-address_set_test` | `internal-branch_ubuntu-user-test` – previous Ubuntu DNS deny | Before `internal-branch_ubuntu-user-test` |
| ICMP deny | `internal-branch_ubuntu-user-address_set_test_ping_Deny` | `internal-branch` – any source, any destination, any application, permit | Before `internal-branch` |

The experiments demonstrate that a correctly configured policy may not take effect when an earlier policy already matches the traffic.

The policy order must therefore be considered together with the source address, destination address, application match, and action.

## 8. Verification commands

The following commands can be used to inspect the relevant configuration and runtime behavior:

### Address Book

~~~bash
show security address-book
~~~

### Security policy configuration and order

~~~bash
show configuration security policies from-zone internal to-zone branch | display set
~~~

### Policy hit counts

~~~bash
run show security policies hit-count from-zone internal to-zone branch
~~~

### Session inspection

~~~bash
run show security flow session source-prefix 192.168.10.3
run show security flow session source-prefix 192.168.30.2
~~~

### Local session logs

~~~bash
run show log messages | match RT_FLOW_SESSION
~~~

The traffic outputs above were captured during the lab. Detailed log excerpts and hit-count output for the new Address Set policies have not yet been added; they can be appended after those specific outputs are collected.

## 9. Results summary

| Scenario | Source | Destination | Application | Observed result |
|---|---|---|---|---|
| DNS permit after policy reorder | Ubuntu `192.168.30.2` | `8.8.8.8`, `8.8.4.4` | DNS UDP/TCP | Reverse lookups and HTTPS request succeeded |
| ICMP before deny reorder | PC2 `192.168.10.3` | `8.8.8.8` | ICMP | Ping succeeded due to the earlier general permit policy |
| ICMP after deny reorder | PC2 `192.168.10.3` | `8.8.8.8` | ICMP | Ping replies stopped after the policy-order change |
| ICMP after deny reorder | Ubuntu `192.168.30.2` | `8.8.8.8` | ICMP | 100% packet loss in the reported test |
| DNS/HTTPS after ICMP deny reorder | Ubuntu `192.168.30.2` | DNS servers / Google | Reverse DNS and HTTPS | Continued to work |

## 10. Conclusion

This lab demonstrated the use of nested Global Address Sets in Juniper SRX security policies and showed why policy order matters.

The source group `HQ-SOURCE-GROUP` consolidated three child Address Sets. The destination group `PUBLIC-DNS-SERVERS` consolidated two DNS server addresses. These groups were then reused in DNS permit and ICMP deny policies.

The test also demonstrated two policy-ordering cases:

1. The DNS permit policy needed to be placed before the older Ubuntu DNS deny policy.
2. The ICMP deny policy needed to be placed before the general `internal-branch` permit policy.

The practical result is that Address Book design and Security Policy ordering must be considered together when building predictable firewall rules.

## Related repository

[SRX-Advanced-Security-Policies](https://github.com/emadalshishani/SRX-Advanced-Security-Policies) – the earlier project that introduced the Ubuntu DNS UDP/TCP deny policy reused in this experiment.

# miniupnpd: Core functionality issues

> [!WARNING]
> - Please update the daemon to version 2.3.12 or later to address the [CVE-2026-107159](https://www.cve.org/CVERecord?id=CVE-2026-107159) DoS vulnerability
> - Please update the daemon to version 2.3.11 or later to address a [security vulnerability](https://github.com/miniupnp/miniupnp/issues/898)
> - Please update the daemon to version 2.3.10 or later to address the [CVE-2026-5720](https://www.cve.org/CVERecord?id=CVE-2026-5720) security vulnerability
> - [CVE-2026-7069](https://www.cve.org/CVERecord?id=CVE-2026-7069) AddPortMapping description buffer overflow ([details](https://tzh00203.notion.site/D-Link-DIR-825-miniupnpd-AddPortMapping-Stack-Overflow-337b5c52018a8028988ecc9daded409e)), seems to be fixed in latest version, but it appears to be related to the second Linux-only issue below

### BSD only (tested with latest OPNsense 26.7 on FreeBSD 15.1)

- [X] ⚠️ Renewing/~~deleting~~ IPv6 port maps does not work; first adding and expiration works; fix proposed   
  Resulting in IPv6 port maps expiring regularly (~1/2 lifetime). 501 Action Failed (UPnP IGDv2) or 8 No resource (PCP) returned on renewing and `ioctl(dev, DIOCCHANGERULE, ...) PF_CHANGE_GET_TICKET: Invalid argument` gets logged on deletion. pf renewing seems not yet implemented, deleting ~~not~~ now working: https://github.com/miniupnp/miniupnp/issues/747 https://github.com/miniupnp/miniupnp/pull/916
- [X] ❗ Does not delete the additional hairpinning `nat` rule for IPv4 UDP port maps; fixed with https://github.com/miniupnp/miniupnp/commit/0f668f1  
  Rules are accumulating. `ioctl(dev, DIOCCHANGERULE, ...) PF_CHANGE_REMOVE: Invalid argument` (<2.3.11: `could not find nat rule to delete`) gets logged. Somehow related: https://github.com/miniupnp/miniupnp/issues/715 https://github.com/miniupnp/miniupnp/issues/793 https://github.com/miniupnp/miniupnp/pull/903
- [X] ❗ Returning an invalid epoch (~80 y) via PCP/NAT-PMP if system uptime is enabled; fix proposed  
  Should also be disabled in sample `miniupnpd.conf` (follow daemon default) to have the epoch reset on daemon restarts, for PCP/NAT-PMP compliance with no lease file
- [X] ❗ Listing of IPv4 port maps using IGDv2 GetListOfPortMappings does not work (empty list); fixed with https://github.com/miniupnp/miniupnp/commit/1b57a72  
  IGDv1 GetGenericPortMappingEntry/GetSpecificPortMappingEntry works, issue in `get_portmappings_in_range`
- [X] ❗ `pfctl -a miniupnpd -s nat` returns `?` as internal IP on FreeBSD >=15.1 with IPv4, but port maps seems to work; fixed with https://github.com/miniupnp/miniupnp/commit/d9ffcdd
- [ ] Use modern (`rdr-to`/`nat-to`) PF rules on FreeBSD 15.0 (2025-12), supports OpenBSD style NAT syntax  
  https://man.freebsd.org/cgi/man.cgi?query=pf.conf#TRANSLATION_EXAMPLES  
  https://man.openbsd.org/pf.conf#Translation  
  https://cgit.freebsd.org/src/commit/?id=e0fe26691fc9  

### Linux only (tested with latest OpenWrt 25.12)

- [X] ❗Unsupported revision returned via `iptables -L` >= 1.8.8 (2022-05) for `DNAT` rules; fixed with https://github.com/miniupnp/miniupnp/pull/923  
  Breaks the existing parsing regex in OpenWrt… when iptables is used; no active port maps displayed in the LuCI UI https://github.com/miniupnp/miniupnp/issues/837. Following the comment (next link), the highest kernel-supported revision should be used. Possibly helpful links:
  - `iptables 1.8.8 breaks compatibility with older versions for some rules`:
  https://bugzilla.netfilter.org/show_bug.cgi?id=1632#c5  
  - `case IPT_SO_GET_REVISION_TARGET`:
  https://github.com/torvalds/linux/blob/master/net/ipv4/netfilter/ip_tables.c#L1672
  - `xt_find_revision()`:
  https://github.com/torvalds/linux/blob/master/net/netfilter/x_tables.c#L393
- [X] ❗Renewing an IPv4 port map with a remote IP filter set via UPnP IGD does not work, error 718 ConflictInMappingEntry is returned with nftables; fixed with https://github.com/miniupnp/miniupnp/pull/925
- [ ] UPnP IGD DeletePortMapping return no error with a non-existing port map  
  Must return 714 NoSuchEntryInArray instead of 200 OK.  
  https://upnp.org/specs/gw/UPnP-gw-WANIPConnection-v2-Service.pdf#page=57
- [ ] UPnP IGDv2 DeletePinhole/CheckPinhole… return an incorrect error with a non-existent UID with nftables  
  Must return 704 NoSuchEntry instead of 501 Action Failed, logs (uninitialised memory?): `inet_pton(�KF)`  
  https://upnp.org/specs/gw/UPnP-gw-WANIPv6FirewallControl-v1-Service.pdf#page=21
- [ ] Long port map descriptions truncated with invalid XML data when listing or 501 Action Failed  
  iptables: Truncated with extra garbage XML data (e.g. `P\��NewPortMapp`, buffer overflow?); fixed on head  
  nftables: AddPortMapping returns 501 Action Failed; fixed on head  
  Both: Now accepts (and writes to the lease file) 1024 chars, but returns only 256 chars to clients and truncates it on restarts to 256 chars instead of 1024, according to commit: https://github.com/miniupnp/miniupnp/commit/b393e45. See https://github.com/miniupnp/miniupnp/pull/812#issuecomment-4446926454

### All platforms

- [X] ⚠️ Parsing of ACL entries has a potential buffer over-read and ignores the description filter field (in 2.3.10)  
  https://github.com/miniupnp/miniupnp/pull/853
- [X] ❗ All IPv4 port maps expire (get deleted) on `reload_from_lease_file` since https://github.com/miniupnp/miniupnp/commit/671e5c0 (in Git head)
- [ ] ❗ Check PCP nonce on IPv4 mapping renewals/deletions, as with IPv6  
  It was partly implemented, only for IPv6: https://github.com/miniupnp/miniupnp/commit/814e855
- [ ] ❗ If IPv6 is available after IPv4, no IPv6 port mapping via UPnP IGDv2 possible  
  The necessary IPv6 GUA SSDP M-SEARCH replies/announcements are disabled. `SSDP packet sender %s (if_index=%d) not from a LAN, ignoring` gets logged, enables then only IPv6 ULA replies/announcements, that doesn't work
- [ ] ❗ Don't disable IPv6 mapping with no address via PCP  
  `IPV6DISABLEDMASK` seems not be set/checked properly, and `ipv6` is set to `PF_INET`/`PF_INET6`/`0`/`1`
- [ ] ❗ A single internal interface with no IPv6 disables it for all interfaces
- [ ] ❗ STUN IPv4 CGNAT false positive filtered test on most routers, nftables/pf tested  
  Test is blocked by default by firewall rules (allow-filtered is required), and, for security reasons, the entire port range cannot be opened permanently on the WAN interface https://redirect.github.com/openwrt/packages/issues/21841. Therefore, a standard port map (using existing firewall functions) to the first LAN IPv4 address of the daemon could be added before (and deleted after) running the filtering test. Then, perform the STUN filtering test with random UDP ports, as currently, via the port map to daemon's newly opened test ports on the LAN interface, this also performs a standard mapping self-test. Using this approach, the CGNAT test would then work with any router OSs such as OpenWrt/OPNsense…, without the need for additional changes/chains. Could be extended to perform the filtering test (increased startup time, DNS/internet req.) only with a private/CGNAT external IPv4, or when config option set to `force-test`. Additionally, the three ports CGNAT filtering tests should only be performed after a successful STUN binding test, first request, with several attempts when DNS/internet availability is delayed
- [ ] ❗ IPv4 port maps time out with sample `miniupnpd.conf` https://github.com/miniupnp/miniupnp/issues/823
- [ ] ❗ PCP nonce `012345670123456701234567` sent is recognised as `674523016745230167452301` (byte order?), but is correctly logged when built with debug in `pcpserver.c`: `syslog(LOG_DEBUG, "MAP nonce:   \t%08x%08x%08x", READNU32(buf), READNU32(buf+4), READNU32(buf+8));`
- [ ] ❗ Does not map a random/high port with an external port of `0` with NAT-PMP  
- [X] ❗ Build failure with `make` (using musl) fails with `testupnphttp.c:97-98:9: error: unknown type name 'fd_set'` (in Git head)
- [ ] Renewing an existing IPv6 port map logs `unrecognized data in lease file`, fails in `lease_file6_expire` at first field
  https://www.rfc-editor.org/info/rfc6886#section-3.3 and https://github.com/miniupnp/libnatpmp/issues/29  
- [ ] Does resolve hostname in remote IP to `0.0.0.0` with UPnP IGD IPv4, works with IPv6
- [ ] Generated UPnP IGD IPv6 port map UIDs should be unique identifiers, preferably random
- [ ] UPnP IGD GetOutboundPinholeTimeout return an incorrect error for the optional, not implemented action  
  Must return 602 Optional Action Not Implemented instead of 501 Action Failed
- [ ] Remote IP/port filtering should also work with PCP, as with UPnP IGD  
  Currently not implemented, PCP option returned; should easily be implemented using the existing backend function calls, as with UPnP IGDv1 AddPortMapping and UPnP IGDv2 IPv6 AddPinhole. From the PCP standard: `All PCP servers MUST support at least one filter per MAP mapping.`
  https://github.com/miniupnp/miniupnp/issues/779
- [ ] Daemon should not send PCP announce from external interface (to multicast address); correctly rejects requests  
  https://redirect.github.com/opnsense/plugins/issues/4769
- [ ] ACL entries do not support IPv6 or MAC addresses (currently, only possible to disable IPv6 in daemon)  
  https://github.com/miniupnp/miniupnp/issues/694
- [ ] With IPv6 access control: Fix for: Cannot open ports <1024 with IPv6 and IGD unlike IPv4 or PCP  
  https://github.com/miniupnp/miniupnp/issues/692
- [ ] Logging issues and recommendations
  - Daemon should not log normal occurring events as errors (spamming)  
  Such as a non-existing (not yet created) IPv6 lease file, which is deleted by the daemon itself, and repeated logging of e.g. normally occurring `rule with label '%s' is not a IGD pinhole` with debug, which then deletes other important log entries on routers with limited log storage https://github.com/miniupnp/miniupnp/issues/764. 5 reported issues on OpenWrt: https://redirect.github.com/openwrt/packages/issues/11971 https://redirect.github.com/openwrt/packages/issues/26483 https://redirect.github.com/openwrt/packages/issues/17601 https://redirect.github.com/openwrt/packages/issues/21685 https://redirect.github.com/openwrt/packages/issues/17258
  - A recommendation for the default log level would be to log only the start banner and daemon-wide warnings/errors, with info level the relevant client mapping requests with errors, and only with debug other and SSDP requests https://github.com/miniupnp/miniupnp/issues/764
  - Daemon does not log IPv4/IPv6 mapping requests clearly  
    The logging (with info level) should be mapping protocol and IPv4/IPv6 agnostic, compact, and clear for humans and machines (with important info, but without long/redundant strings (no user-agent header, description, ext. IPv4). See https://github.com/miniupnp/miniupnp/issues/764 https://github.com/miniupnp/miniupnp/issues/707. Examples:
    ```
    Add IPv4 port map 10000:192.168.1.100:10000/TCP remote IP 198.51.100.1 lifetime 3600 via UPnP IGD
    Renew IPv6 port map [2001:db8::100]:10000/TCP remote IP 2001:db8::1 remote port 10000 lifetime 3600 via UPnP IGDv2 IPv6 [Update] uid …
    Reject IPv6 port map [2001:db8::101]:10001/TCP via PCP nonce … reason ACL/port in use/for third-party/nonce not matching…
    Delete/Expire…
    ```
    Currently, this is difficult to achieve as the daemon has an independent codebase for each mapping protocol, even with very different/incomplete log messages
- [ ] Remote IP filter for IPv4 port maps are lost on lease file reloads, in Git head not backwards compatible fixed  
  - Current lease file functionality had/has many issues and should possibly be replaced at the same time by a new implementation that follows a standard of a unified IPv4/IPv6 port mapping lease file format specifying more fields, e.g. a random mapping ID for easy/reliable deletion in the first field, IP family, reserve fields for future use, more conveniently ordered fields and better separator (`|`, from PSV), also more directly usable in UIs
  - For compatibility, it should be built-in if no (old) lease file support has been enabled (via `#ifndef ENABLE_LEASEFILE`). Using new format only then ensures backwards compatibility, so that the switch to new unified IPv4/IPv6 format can be made downstream at any time, also as the IPv4 lease file format has not changed since introduction. Support for old format can therefore also be easily removed from daemon later
  - Unified IPv4/IPv6 lease file format example, fields currently unavailable in daemon are added (reserved/empty) for forward compatibility:
     ```
     UUID (for unique deletion of entries) | (Res) | (Res) | IP family | IP address    | Port  | Protocol | (Res) | (Res) | Ext. port | Remote IP address | Remote port | Expires    | Added via       | Map ID (UID/nonce)       | (Res) | Map description
     701eee8c-c5be-4c16-a439-8607bd6deb47  |       |       | IPv4      | 192.168.1.100 | 10000 | TCP      |       |       | 10000     | 198.51.100.1      | 0           | 1704067200 | UPnP IGD        |                          |       | Example descr.
     fea4f496-ab22-47e2-8a2d-7f59f29a5f55  |       |       | IPv6      | 2001:db8::100 | 10000 | TCP      |       |       |           | 2001:db8::1       | 30000       | 1704067200 | UPnP IGDv2 IPv6 | 49364                    |       |
     eda19c1c-fb41-4c59-b6e0-5839c74df8db  |       |       | IPv6      | 2001:db8::101 | 20000 | TCP      |       |       |           | 2001:db8::/64     | 30000       | 1704067200 | PCP             | 3407329f1c1de5f4c337726b |       |
     ```

  - At the moment, to avoid unexpected breaks soon (downstream devs would possibly be happy), a quick fix could be to replace two `#ifdef SUPPORT_REMOTEHOST` with `#if 0` in the code until 2.4.0, to not break in the next patch release, as many router UIs (OpenWrt/X-Wrt/ASUS…) parse the file
- [ ] Build check failure with `make check` fails with multiple network interfaces `make: *** [check.mk:13: validategetifaddr] Error 1`

### Repository/project

- [ ] ❗ All release archives differ (`$ID:` header) from the git repository  
  This makes backporting patches more time-consuming.  
  ```
  curl -Ls https://miniupnp.tuxfamily.org/files/miniupnpd-2.3.9.tar.gz | tar -xz
  git clone --branch miniupnpd_2_3_9 --depth 1 https://github.com/miniupnp/miniupnp 2>/dev/null
  diff miniupnpd-2.3.9 miniupnp/miniupnpd | wc -l
  184
  diff miniupnpd-2.3.9 miniupnp/miniupnpd
  Only in miniupnp/miniupnpd: .gitignore
  diff miniupnpd-2.3.9/Makefile.linux miniupnp/miniupnpd/Makefile.linux
  1c1
  < # $Id: Makefile.linux,v 1.114 2025/04/21 21:48:56 nanard Exp $
  ---
  > # $Id: Makefile.linux,v 1.113 2025/03/30 22:36:04 nanard Exp $
  diff miniupnpd-2.3.9/Makefile.linux_nft miniupnp/miniupnpd/Makefile.linux_nft
  …
  ```
- [ ] GitHub releases URLs/tags cannot be used as a more reliable download alternative without substitution  
  To be directly usable downstream, new tags should be made in the form `miniupnpd-1.2.3` instead of `miniupnpd_1_2_3` https://github.com/miniupnp/miniupnp/issues/770

Updated: 2026-10-29, tested with the latest commit `05bb29e` of the [daemon](https://github.com/miniupnp/miniupnp/commits/master/miniupnpd) from 2026-10-20

(Note: These are core functionality issues that I would like to report, but cannot. However, you are welcome to forward this (markdown or link) to the project as an [issue](https://github.com/miniupnp/miniupnp/issues))

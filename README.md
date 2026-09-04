# miniupnpd: Core functionality issues

> [!WARNING]
> - Please update the daemon to version 2.3.11 or later to address a [security vulnerability](https://github.com/miniupnp/miniupnp/issues/898)
> - Please update the daemon to version 2.3.10 or later to address the [CVE-2026-5720](https://www.cve.org/CVERecord?id=CVE-2026-5720) security vulnerability
> - [CVE-2026-7069](https://www.cve.org/CVERecord?id=CVE-2026-7069) AddPortMapping description buffer overflow ([details](https://tzh00203.notion.site/D-Link-DIR-825-miniupnpd-AddPortMapping-Stack-Overflow-337b5c52018a8028988ecc9daded409e)), seems to be fixed in latest version, but it appears to be related to the second Linux-only issue below

### BSD only (tested with latest OPNsense 26.7)

- [ ] ⚠️ Renewing/deleting IPv6 port maps does not work; adding and expiration works  
  Resulting in IPv6 port maps expiring regularly. pf renewing not yet implemented, deleting not working: https://github.com/miniupnp/miniupnp/issues/747
- [ ] ❗ Does not delete the additional IPv4 hairpinning `nat` rule  
  `could not find nat rule to delete`  and `ioctl(dev, DIOCCHANGERULE, ...) PF_CHANGE_REMOVE: Invalid argument` on 2.3.11 gets logged. See https://github.com/miniupnp/miniupnp/issues/715 and related https://github.com/miniupnp/miniupnp/issues/793
- [ ] ❗ Only adds the additional IPv4 hairpinning `nat` rule for UDP, not TCP
- [ ] ❗ Returns an invalid epoch (~80 y) via PCP/NAT-PMP if system uptime is enabled  
  Should also be disabled in sample `miniupnpd.conf` (follow daemon default) to have the epoch reset on daemon restarts, for PCP/NAT-PMP compliance with no lease file
- [ ] ❗ Listing of IPv4 port maps using IGDv2 GetListOfPortMappings does not work (empty list)  
  IGDv1 GetGenericPortMappingEntry/GetSpecificPortMappingEntry works
- [X] ❗ `pfctl -a miniupnpd -s nat` returns `?` as internal IP on FreeBSD >=15.1 with IPv4, but port maps seems to work; fixed with https://github.com/miniupnp/miniupnp/commit/d9ffcdd
- [ ] Use modern (`rdr-to`/`nat-to`) PF rules on FreeBSD 15.0 (2025-12), supports OpenBSD style NAT syntax  
  https://man.freebsd.org/cgi/man.cgi?query=pf.conf#TRANSLATION_EXAMPLES  
  https://man.openbsd.org/pf.conf#Translation  
  https://cgit.freebsd.org/src/commit/?id=e0fe26691fc9  

### Linux only (tested with latest OpenWrt 25.12)

- [ ] ❗Unsupported revision returned via `iptables -L` >= 1.8.8 (2022-05) for `DNAT` rules  
  Breaks the existing parsing regex in OpenWrt… when iptables is used; no active port maps displayed in the LuCI UI https://github.com/miniupnp/miniupnp/issues/837. Following the comment (next link), the highest kernel-supported revision should be used. Possibly helpful links:
  - `iptables 1.8.8 breaks compatibility with older versions for some rules`:
  https://bugzilla.netfilter.org/show_bug.cgi?id=1632#c5  
  - `case IPT_SO_GET_REVISION_TARGET`:
  https://github.com/torvalds/linux/blob/master/net/ipv4/netfilter/ip_tables.c#L1672
  - `xt_find_revision()`:
  https://github.com/torvalds/linux/blob/master/net/netfilter/x_tables.c#L393
- [ ] ❗Long port map descriptions truncated with invalid XML data when listing or 501 Action Failed  
  iptables: Truncated with extra garbage XML data (e.g. `P\��NewPortMapp`, buffer overflow?); fixed on head  
  nftables: AddPortMapping returns 501 Action Failed; fixed on head  
  Both: Now truncated on head to 256 chars instead of 1024, according to commit https://github.com/miniupnp/miniupnp/commit/b393e45. See https://github.com/miniupnp/miniupnp/pull/812#issuecomment-4446926454
- [ ] ❗UPnP IGD DeletePortMapping return no error with a non-existent port map  
  Must return 714 NoSuchEntryInArray instead of 200 OK.  
  https://upnp.org/specs/gw/UPnP-gw-WANIPConnection-v2-Service.pdf#page=57
- [ ] ❗UPnP IGDv2 DeletePinhole/CheckPinhole… return an incorrect error with a non-existent UID with nftables  
  Must return 704 NoSuchEntry instead of 501 Action Failed, logs (uninitialised memory?): `inet_pton(�KF)`  
  https://upnp.org/specs/gw/UPnP-gw-WANIPv6FirewallControl-v1-Service.pdf#page=21
- [ ] Renewing an existing IPv6 port map logs `unrecognized data in lease file`

### All platforms

- [X] ⚠️ Parsing of ACL entries has a potential buffer over-read and ignores the description filter field (in 2.3.10)  
  https://github.com/miniupnp/miniupnp/pull/853
- [ ] ❗ Compilation failures
  - `make` fails with `testupnphttp.c:97-98:9: error: unknown type name 'fd_set'` (tested on OpenWrt/musl)
  - `make check` fails with multiple network interfaces `make: *** [check.mk:13: validategetifaddr] Error 1`. Fix for testgetifaddr.sh:
  ```
  -       EXTIF="`LC_ALL=C $IP -4 route | grep 'default' | sed -e 's/.*dev[[:space:]]*//' -e 's/[[:space:]].*//'`" || exit 1
  +       EXTIF="`LC_ALL=C $IP -4 route | grep 'default' | head -1 | sed -e 's/.*dev[[:space:]]*//' -e 's/[[:space:]].*//'`" || exit 1
  ```
- [ ] ❗ IPv4 port maps time out with sample `miniupnpd.conf` https://github.com/miniupnp/miniupnp/issues/823
- [ ] ❗ PCP nonce `012345670123456701234567` sent is recognised as `674523016745230167452301` (byte order?), but is correctly logged when built with debug in `pcpserver.c`: `syslog(LOG_DEBUG, "MAP nonce:   \t%08x%08x%08x", READNU32(buf), READNU32(buf+4), READNU32(buf+8));`
- [ ] ❗ If IPv6 is available after IPv4, no IPv6 port mapping via UPnP IGDv2 possible  
  The necessary IPv6 GUA SSDP M-SEARCH replies/announcements are disabled. `SSDP packet sender %s (if_index=%d) not from a LAN, ignoring` gets logged. Also, not handling external interface down/up events like with IPv4, and don't properly disable IPv6 with no address and PCP if UPnP IGD is not enabled, then `PCPSendUnsolicitedAnnounce() IPv6 sendto(): Network is unreachable` gets logged
- [ ] A single internal interface with no IPv6 disables it for all interfaces
- [ ] Does not map a random/high port with an external port of `0` with NAT-PMP  
  https://www.rfc-editor.org/info/rfc6886#section-3.3  
  https://github.com/miniupnp/libnatpmp/issues/29  
- [ ] Does resolve hostname in remote/source IP to `0.0.0.0` with UPnP IGD IPv4, works with IPv6
- [ ] Generated UPnP IGD IPv6 port map UIDs should be unique identifiers, preferably random
- [ ] UPnP IGD GetOutboundPinholeTimeout return an incorrect error for the optional, not implemented action  
  Must return 602 Optional Action Not Implemented instead of 501 Action Failed
- [ ] Remote/source IP/port filtering should also work with PCP, as with UPnP IGD  
  Currently not implemented, sent options are ignored; should easily be implemented using the existing backend function calls, as with UPnP IGDv1 AddPortMapping and UPnP IGDv2 IPv6 AddPinhole. From the PCP standard: `All PCP servers MUST support at least one filter per MAP mapping.`
  https://github.com/miniupnp/miniupnp/issues/779
- [X] IPv4 port maps with a remote/source IP filter set are lost on lease file reload on daemon restarts; fixed with https://github.com/miniupnp/miniupnp/commit/671e5c0   
  Reload functionality should possibly be replaced right away by a new (automatically built-in) implementation that follows a common and unified IPv4/IPv6 port mapping lease file format with a random mapping ID for easy/reliable deletion in the first field. For backwards compatibility reasons, it should be enabled if no (old) lease file support been built-in
- [ ] Daemon should not send PCP announce from external interface (to multicast address); correctly rejects requests  
  https://redirect.github.com/opnsense/plugins/issues/4769
- [ ] STUN IPv4 CGNAT test not optimally implemented for use in routers  
  Test is blocked by default by firewall rules (allow-filtered is required, nftables/pf tested) on most routers, and, for security reasons, the entire port range cannot be opened permanently on the WAN interface https://redirect.github.com/openwrt/packages/issues/21841. Therefore, a standard port map to the first LAN IPv4 address of the daemon should be added before (and deleted after) running the filtering test. Then, perform the STUN filtering test with random UDP ports, as currently, via the port map to daemon's newly opened test ports on the LAN interface. With this approach, the CGNAT test would then work with router OSs such as OpenWrt/OPNsense…, which currently do not work. Could be extended to perform the filtering test (increased startup time, DNS/internet req.) only with a private/CGNAT external IPv4, or when config option set to `force-test`. Additionally, the three ports CGNAT filtering tests should only be performed after a successful STUN binding test (first request), but with several attempts when DNS/internet availability is delayed
- [ ] ACL entries do not support IPv6 or MAC addresses (currently, only possible to disable IPv6 in daemon)  
  https://github.com/miniupnp/miniupnp/issues/694
- [ ] With IPv6 ACL: Fix for: Cannot open ports <1024 with IPv6 and IGD unlike IPv4 or PCP  
  https://github.com/miniupnp/miniupnp/issues/692
- [ ] Logging issues and recommendations
  - Daemon should not log normal occurring events as errors (spamming)  
  Such as a non-existing (not yet created) IPv6 lease file, which is deleted by the daemon itself, and repeated logging of e.g. normally occurring `rule with label '%s' is not a IGD pinhole` with debug, which then deletes other important log entries on routers with limited log storage https://github.com/miniupnp/miniupnp/issues/764. 5 reported issues on OpenWrt: https://redirect.github.com/openwrt/packages/issues/11971 https://redirect.github.com/openwrt/packages/issues/26483 https://redirect.github.com/openwrt/packages/issues/17601 https://redirect.github.com/openwrt/packages/issues/21685 https://redirect.github.com/openwrt/packages/issues/17258
  - A recommendation for the default log level would be to log only the start banner and daemon-wide warnings/errors, with info level the relevant client mapping requests with errors, and only with debug other and SSDP requests https://github.com/miniupnp/miniupnp/issues/764
  - Daemon does not log IPv4/IPv6 mapping requests clearly  
    The logging (with info level) should be mapping protocol and IPv4/IPv6 agnostic, compact, and clear for humans and machines (with important info, but without long/redundant strings (no user-agent header, description, ext. IPv4). See https://github.com/miniupnp/miniupnp/issues/764 https://github.com/miniupnp/miniupnp/issues/707. Examples:
    ```
    Add|Renew|Delete|Expire|Reject…
    Add IPv4 port map ext_port:ipv4:int_port/TCP remote_ip=1.2.3.4 lifetime=3600 via=UPnP IGD
    Renew IPv6 port map [ipv6]:port/TCP remote_ip=2123:: remote_port=1234 lifetime=3600 via=UPnP IGDv2 IPv6 [update] uid=…
    Reject IPv6 port map [ipv6]:port/TCP lifetime=3600 via=PCP nonce=… reason=ACL|conflict|for-third-party…
    ```
    Currently, this is difficult to achieve as the daemon has an independent codebase for each mapping protocol, even with very different/incomplete log messages

### Repository/project

- [ ] ❗ All release archives differ from the git repository  
  This makes backporting patches more time-consuming.  
  ```
  curl -Ls https://miniupnp.tuxfamily.org/files/miniupnpd-2.3.9.tar.gz | tar -xz
  git clone --branch miniupnpd_2_3_9 --depth 1 https://github.com/miniupnp/miniupnp 2>/dev/null
  git diff --shortstat miniupnpd-2.3.9 miniupnp/miniupnpd
  diff miniupnpd-2.3.9 miniupnp/miniupnpd
  78 files changed, 2658 insertions(+), 167 deletions(-)
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

Updated: 2026-09-04, tested with the latest commit `5afea3c` of the [daemon](https://github.com/miniupnp/miniupnp/commits/master/miniupnpd) from 2026-09-04

(Note: These are core functionality issues that I would like to report, but cannot. However, you are welcome to forward this (markdown or link) to the project as an [issue](https://github.com/miniupnp/miniupnp/issues))

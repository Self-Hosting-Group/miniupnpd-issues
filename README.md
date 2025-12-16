# miniupnpd: Core functionality issues

### BSD only (tested with current OPNsense)

- [ ] ⚠️ Deleting IPv6 port maps (and IGDv2 update function) does not work; adding and expiration works  
  Only about IGDv2 update: https://github.com/miniupnp/miniupnp/issues/747
- [ ] ❗ Does not delete the additional IPv4 hairpinning `nat` rule, and only adds it for UDP, not TCP  
  https://github.com/miniupnp/miniupnp/issues/715 and related https://github.com/miniupnp/miniupnp/issues/793
- [ ] ❗ Returns an invalid epoch (~80y) via PCP/NAT-PMP if system uptime is enabled  
  Should also be disabled in sample `miniupnpd.conf` (follow daemon default) to have the epoch reset on daemon restarts, for PCP/NAT-PMP compliance with no lease file
- [ ] Listing of IPv4 port maps using IGDv2 GetListOfPortMappings does not work (empty list)  
  IGDv1 GetGenericPortMappingEntry/GetSpecificPortMappingEntry works
- [ ] Use modern pf rules on FreeBSD 15.0 (2025-12), supports OpenBSD style NAT syntax  
  https://cgit.freebsd.org/src/commit/?id=e0fe26691fc9 and search for `FreeBSD 15.0 will support OpenBSD` for rule example in https://www.netgate.com/blog/updates-to-the-pf-packet-filter-in-freebsd-and-pfsense-software

### Linux only (tested with current OpenWrt)

- [ ] ❗Unsupported revision returned via `iptables -L` >= 1.8.8 (2022-05) for `DNAT` rules  
  Breaks the existing parsing regex in OpenWrt... when iptables is used; no active port maps displayed in the LuCI UI https://github.com/miniupnp/miniupnp/issues/837  
  I searched and found the following about iptables revision, maybe helpful:
  - `case IPT_SO_GET_REVISION_TARGET`:
  https://github.com/torvalds/linux/blob/master/net/ipv4/netfilter/ip_tables.c#L1672
  - `xt_find_revision()`:
  https://github.com/torvalds/linux/blob/master/net/netfilter/x_tables.c#L393
  - `The argument to IPT_SO_GET_REVISION_*.  Returns highest revision kernel supports, if >= revision.` (older header):
  https://github.com/torvalds/linux/blob/master/include/uapi/linux/netfilter/x_tables.h#L77

### All platforms

- [ ] ⚠️ Parsing of ACL entries has a potential buffer over-read and ignores the description filter field
  https://github.com/miniupnp/miniupnp/pull/853
- [ ] ❗Longer port map descriptions return invalid XML data with all UPnP IGD port map listing functions
  Truncated to 64 chars and with extra garbage data (since some time, uninitialised memory?)
- [ ] Daemon should not send PCP announce from external interface (to multicast address); correctly rejects requests https://redirect.github.com/opnsense/plugins/issues/4769
- [ ] Remote/source IP/port filtering should also work with PCP, as with UPnP IGD  
  Currently not implemented, but sent options get positively accepted, but ignored; should easily be implemented using the existing backend function calls, as with UPnP IGDv1 AddPortMapping and UPnP IGDv2 IPv6 AddPinhole. From the PCP standard: `All PCP servers MUST support at least one filter per MAP mapping.`
  https://github.com/miniupnp/miniupnp/issues/779
- [ ] IPv4 port maps time out with sample `miniupnpd.conf` https://github.com/miniupnp/miniupnp/issues/823
- [ ] STUN IPv4 CGNAT test not optimally implemented for use in routers since introduction in ~2008  
  Test is blocked by default by firewall rules (allow-filtered is required) on most routers, and, for security reasons, the entire port range cannot be opened permanently https://redirect.github.com/openwrt/packages/issues/21841. Therefore, a standard port map to the first LAN IPv4 address of the daemon should be added before (and deleted after) running the filtering test. Then, perform the STUN filtering test with random UDP ports, as currently, via the port map to daemon's newly opened test ports on the LAN interface. This setup would work without any changes in routers, e.g., with OpenWrt/OPNsense..., and would then also provide a complete/standard IPv4 port mapping self-test at daemon start with STUN enabled. Could be extended to perform only the filtering test (increased startup time, DNS req.), with a private/CGNAT external IPv4, or with an additional config option to force it (disabled by default). Additionally, the three ports CGNAT filtering tests should only be performed after the first successful STUN binding test
- [ ] IPv4 port maps with a remote/source IP filter set are lost on lease file reload on daemon restarts
- [ ] A single internal interface with no IPv6 disables it for all interfaces
- [ ] ACL entries do not support IPv6 or MAC addresses (currently, only possible to disable IPv6)
  https://github.com/miniupnp/miniupnp/issues/694
- [ ] With IPv6 ACL: Fix for: Cannot open ports <1024 with IPv6 and IGD unlike IPv4 or PCP
  https://github.com/miniupnp/miniupnp/issues/692
- [ ] Logging issues and recommendations
  - A recommendation for the default log level would be to log only the start banner and daemon-wide warnings/errors, with info level the relevant client mapping SOAP requests with errors, and the `SSDP M-SEARCH...` logging, and only with debug all/other SOAP and SSDP requests https://github.com/miniupnp/miniupnp/issues/764
  - Daemon should not log normal occurring events as errors (spamming)  
  Such as a non-existing (not yet created) IPv6 lease file, which is deleted by the daemon itself, and repeated logging of e.g. normally occurring `rule with label '%s' is not a IGD pinhole` with debug, which then deletes other important log entries on routers with limited log storage https://github.com/miniupnp/miniupnp/issues/764. 5 reported issues on OpenWrt: https://redirect.github.com/openwrt/packages/issues/11971 https://redirect.github.com/openwrt/packages/issues/26483 https://redirect.github.com/openwrt/packages/issues/17601 https://redirect.github.com/openwrt/packages/issues/21685 https://redirect.github.com/openwrt/packages/issues/17258
  - Daemon does not log IPv4/IPv6 mapping requests clearly https://github.com/miniupnp/miniupnp/issues/707  
    The logging (with info level) should be mapping protocol and IPv4/IPv6 agnostic, compact, and clear for humans and machines (incl. errors), with important info but without long/redundant strings (no user-agent header, description, ext. IPv4). Examples:
    ```
    Add|Delete|Expired IPv4 port map ip:int_port/TCP ext_port=1234 [lifetime=3600 via=UPnP IGD error=606 failed]
    Add|Delete|Expired IPv6 port map [ip]:int_port/TCP [remote_ip=2123:: remote_port=1234 lifetime=3600 via=PCP nonce=...]
    Add|Delete|Expired IPv6 port map [ip]:int_port/TCP [lifetime=3600 via=UPnP IGDv2 [update] uid=...]
    ```

(Note: These are core functionality issues that I would like to report, but cannot. However, you are welcome to forward this (markdown or link) to the project as an [issue](https://github.com/miniupnp/miniupnp/issues).)

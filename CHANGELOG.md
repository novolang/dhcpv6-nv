# Changelog

All notable changes to dhcpv6-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.1.0

First release.

### Added

- The DHCPv6 message of RFC 8415 in a 1024-byte buffer: the client and
  server header, the relay header and the message a relay message carries,
  the walk over options, and writers that open and close an option so that
  options nest.
- The DUIDs of RFC 8415 and RFC 6355: DUID-LLT, DUID-EN, DUID-LL and
  DUID-UUID.
- The options: Client Identifier, Server Identifier, IA_NA, IA_TA and
  IA_PD with their IA Address and IA Prefix options and the T1 and T2
  rules, Option Request, Preference, Elapsed Time, Relay Message,
  Authentication (read only), Server Unicast, Status Code with its codes,
  Rapid Commit, User Class, Vendor Class, Vendor-specific Information,
  Interface-Id, Reconfigure Message, Reconfigure Accept, Information
  Refresh Time, SOL_MAX_RT and INF_MAX_RT, the DNS servers and Domain
  Search List of RFC 3646 and the NTP Server option of RFC 5908.
- The client of RFC 8415 section 18.2 as a step the caller drives: Solicit
  with the retransmission schedule of section 15, Advertise selection by
  preference, Request, Reply, Renew at T1, Rebind at T2, Release, Decline,
  Confirm, Information-request for a stateless client, and Rapid Commit.
  Time is the caller's, in milliseconds, and so is the random word.
- `tests/embedded_probe.nv`, a client run to a lease, a Renew and a Release
  on a Cortex-M4 under QEMU. The image for `--target=nrf52-qemu` is 25,144
  bytes of flash, of which the program is 16,954 bytes and the runtime and
  libgcc the rest; the probe uses no heap.
- Six packets of the Wireshark sample capture `DHCPv6.pcap` as test
  vectors, and the client run through their exchange.

### Not included

- A relay agent and a server. The relay form is read and written.
- Verifying the Authentication option, and with it acting on a Reconfigure
  message, which is ignored.
- IA_TA, and more than one address or prefix, in the client.

### Known

- `tests/dhcp6_opt_tests.nv` builds one test vector in a helper function
  because `(a + b).repeat(n)` leaks the join under `novo test`; the
  filing is `memory-perceus/a-method-call-on-a-parenthesised-str-join-leaks-the-join`
  in the novo repository.

This release requires novo 0.24.1 or later.

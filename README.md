# dhcpv6-nv

The Dynamic Host Configuration Protocol for IPv6 is how a device on an
IPv6 network asks for an address, a delegated prefix and the addresses of
a DNS server and a time server, and keeps the answer for a limited time
called a lease. It is specified in
[RFC 8415](https://www.rfc-editor.org/rfc/rfc8415). This package is the
protocol without a network: the message and the relay form, the options,
and a client that obtains and keeps a lease, written for a device with no
heap. Every message is a fixed 1024-byte buffer. The caller supplies the
clock in milliseconds, a random number, and the two functions that send
and receive a UDP datagram.

DHCPv4 is a different protocol with a different message. For it, see
[dhcp-nv](https://novo-lang.org/packages/dhcp-nv).

## A device that obtains a lease

A device with a network interface runs the client in a loop of two calls.
`client_tick` is called when the time `client_next_at` comes, and it
builds the next message to send. `client_receive` is called when a
datagram arrives on port 546, and it builds a reply when one is due. Both
answer an action that says what to do with the buffer.

The program below runs that loop against messages it builds itself, so
that it can be run anywhere. On a board the lines marked *send* write the
buffer to the interface, to ff02::1:2 port 547, and the datagram that
`client_receive` reads is the one the interface received.

```novo
use dhcp6msg
use dhcp6opt
use dhcp6client

// The DUID of a device or of a server: a link-layer address, 02:00:00:00:00:<last>.
fn duid_ll(last: Int, var d: Vec[u8; 130])
    var ll: Vec[u8; 128] = Vec.new()
    for b in [2, 0, 0, 0, 0, last]
        let _p = ll.push(b as u8)
    let _ok = dhcp6msg.duid_ll(var d, 1, var ll)

// Stand-in for the server: a message of `kind` for the transaction `txid`
// that grants 2001:db8::100 for two hours.
fn server_message(kind: Int, txid: Int, var out: Vec[u8; 1024]) -> Bool
    var client: Vec[u8; 130] = Vec.new()
    duid_ll(1, var client)
    var server: Vec[u8; 130] = Vec.new()
    duid_ll(0x99, var server)
    var ok = dhcp6msg.begin(var out, kind, txid)
    ok = ok and dhcp6msg.put_client_id(var out, var client)
    ok = ok and dhcp6msg.put_server_id(var out, var server)
    let ia = dhcp6msg.open_ia_na(var out, 1, 0, 0)
    ok = ok and dhcp6msg.put_ia_addr(var out, dhcp6msg.ip6(0x20010db8, 0, 0, 0x100), 3600, 7200)
    ok and dhcp6msg.close_option(var out, ia)

fn main() [io]
    var duid: Vec[u8; 130] = Vec.new()
    duid_ll(1, var duid)
    var client = dhcp6client.client_new(var duid, dhcp6client.DHCP6_WANT_NA, 1, 2)

    var datagram: Vec[u8; 1024] = Vec.new()
    var answer: Vec[u8; 1024] = Vec.new()

    // The first tick schedules the Solicit after a random delay of up to
    // a second; the second, at that time, builds it.
    let _a = dhcp6client.client_tick(var client, 0, 0x0001F401, var datagram)
    let sent = dhcp6client.client_tick(var client, 500, 0x0001F402, var datagram)
    if sent == dhcp6client.DHCP6_ACT_SEND
        // Send: write `datagram` to ff02::1:2 port 547.
        // Receive: here the server's Advertise comes back at once.
        let _b = server_message(dhcp6msg.DHCP6_ADVERTISE, dhcp6msg.xid(var datagram), var answer)
        let _c = dhcp6client.client_receive(var client, 600, 0, var answer, var datagram)

    // The client waits out the first timeout for better offers, then asks.
    let asked = dhcp6client.client_tick(var client, 1550, 0x0003E803, var datagram)
    if asked == dhcp6client.DHCP6_ACT_SEND
        let _d = server_message(dhcp6msg.DHCP6_REPLY, dhcp6msg.xid(var datagram), var answer)
        let _e = dhcp6client.client_receive(var client, 1600, 0, var answer, var datagram)

    if dhcp6client.client_has_lease(var client)
        let a = dhcp6client.client_address(var client)
        println("address, as four 32-bit words: ${a.a} ${a.b} ${a.c} ${a.d}")
        println("renew at ${dhcp6client.client_t1_at(var client)} ms")
```

## What DHCPv6 is

A client without an address sends a **Solicit** to the multicast address
ff02::1:2. Each server that can serve it answers with an **Advertise**
that names the addresses or prefixes it would give and carries a
preference from 0 to 255. The client collects Advertise messages for the
first retransmission timeout, picks the highest preference, and sends a
**Request** to that server. The server confirms with a **Reply** that
carries the lease. A client that asked for **Rapid Commit** can take the
Reply to its Solicit and skip the middle two messages.

What a server grants is held in an **identity association**, an IA. An
IA_NA holds non-temporary addresses, an IA_TA temporary ones, and an
IA_PD a delegated prefix; each has an identifier chosen by the client and
holds IA Address or IA Prefix options with a preferred and a valid
lifetime. A client and a server name each other by a **DUID**, which is a
link-layer address, a link-layer address and a time, an enterprise number,
or a UUID.

A lease has two timers, both measured from the moment it was granted. At
**T1** the client asks the server that granted it to extend the lease with
a **Renew**. At **T2** it asks any server with a **Rebind**. When the
longest valid lifetime ends, the client must stop using the address and
starts again. A server that leaves T1 and T2 at zero leaves them to the
client, which uses half and 0.8 of the shortest preferred lifetime.

| State | The client is | Leaves on |
| --- | --- | --- |
| INIT | without a lease and without an exchange | its next tick |
| STARTING | waiting out the random delay before its first message | the end of the delay |
| SOLICIT | collecting Advertise messages | the end of the first timeout, or a preference of 255 |
| REQUEST | waiting for a Reply to a Request | a Reply, or ten transmissions |
| BOUND | holding a lease | T1 |
| RENEW | asking its server to extend the lease | a Reply, or T2 |
| REBIND | asking any server to extend the lease | a Reply, or the end of the lease |
| CONFIRM | asking whether its addresses are still on the link | a Reply, or ten seconds |
| RELEASE, DECLINE | telling the server it gives the lease up | a Reply, or four transmissions |
| INFORM | asking a stateless server for parameters | a Reply |
| INFORMED | holding parameters | the Information Refresh Time |
| IDLE | finished with its lease | `client_restart` |

| Quantity | Value |
| --- | --- |
| Client and server header | 4 bytes: message type and a 24-bit transaction identifier |
| Relay header | 34 bytes: type, hop count, link address, peer address |
| Client and server UDP ports | 546, 547 |
| Message buffer | 1024 bytes |
| First Solicit delayed by | 0 to 1 second |
| Retransmission timeout | doubles from its initial value (1 s, 10 s for Renew and Rebind) to its maximum (3600 s Solicit, 30 s Request, 600 s Renew and Rebind, 4 s Confirm) |
| Randomisation | the timeout times a factor between -0.1 and +0.1; for the first Solicit between 0 and +0.1 |
| Requests sent before the client starts again | 10 |
| Confirm gives up after | 10 seconds |
| Release and Decline attempts | 4 |
| Lease that never ends | `0xFFFFFFFF` seconds |

## Install

```
novo pkg add dhcpv6-nv
```

## What the package contains

| Module | Contents |
| --- | --- |
| `dhcp6msg` | The message in a `Vec[u8; 1024]`: the client and server header, the relay header and the message it carries, the walk over options, the writers that open and close an option so that options nest, the DUIDs of RFC 8415 and RFC 6355, the IPv6 address as four words, and the numbers RFC 8415 assigns. |
| `dhcp6opt` | The options as values: the IA_NA, IA_TA and IA_PD identity associations with the IA Address and IA Prefix options inside them and the T1 and T2 rules, the Status Code, Preference, Elapsed Time, Option Request, Server Unicast, Rapid Commit, Reconfigure Message and Reconfigure Accept options, SOL_MAX_RT, INF_MAX_RT and the Information Refresh Time, the DNS servers and the Domain Search List of RFC 3646 with the RFC 1035 label form, the NTP Server option of RFC 5908, the User Class, Vendor Class and Vendor-specific Information options, the Interface-Id option, and the Authentication option, read only. |
| `dhcp6client` | The client of section 18.2: the states in the table above, the retransmission schedule of section 15, Advertise selection by preference, Rapid Commit, Renew, Rebind, Release, Decline, Confirm, the Information-request of the stateless client, and the lease with its two timers. |

## How to choose an entry point

**`dhcp6client` is for the device.** It holds the whole exchange in one
`Dhcp6Client` value of 1064 bytes and builds every message the client
sends.

**`dhcp6msg` and `dhcp6opt` are for a program that reads or writes single
messages.** A packet tool, a test that builds a Reply, a server written on
top of this package and a relay agent use the readers and writers
directly. `relay_inner` copies the message a relay message carries into a
buffer of its own, and `begin_relay` with `put_relay_message` wraps one.

## The rules a user needs

1. **The clock is a number of milliseconds, and the caller reads it.**
   Every function that needs time takes `now`. The origin is the caller's.
   The lifetimes in a message are seconds, and the client converts them. A
   clock that goes backwards is not handled.
2. **The caller supplies a random word.** `client_tick` and
   `client_receive` take it. Its low 24 bits are the transaction
   identifier of a new exchange, and bits 8 to 23 give the randomisation
   factor of section 15 and the delay before the first message. A device
   with no random source can pass a hash of its DUID and its uptime. The
   word has to differ between exchanges, or the transaction identifiers
   repeat.
3. **An action says where the buffer goes.** `DHCP6_ACT_SEND` means
   ff02::1:2 port 547 from port 546. `DHCP6_ACT_SEND_UNICAST` means the
   address `client_server_address` answers, port 547, and is used only
   when a server sent the Server Unicast option; a Rebind and a Confirm
   always go to the multicast address. A Reply with the status
   UseMulticast sends the message again to the multicast address.
4. **Call `client_tick` when `client_next_at` comes.** It does nothing
   earlier. For a client that holds a lease that never ends, and for an
   idle client, `client_next_at` is -1.
5. **The first offer is not always taken.** The client collects Advertise
   messages until the first timeout of the Solicit ends and requests from
   the one with the highest preference; an Advertise with preference 255
   is requested at once, and so is one that arrives after the first
   timeout.
6. **Do not use an address after `DHCP6_ACT_REJECTED` or
   `DHCP6_ACT_EXPIRED`.** The client is back in INIT with no address. It
   starts a new exchange on its next tick.
7. **Check the address before using it.** RFC 8415 section 18.2.8 asks
   the client to probe a new address with Neighbor Discovery. If another
   host answers, call `client_decline`, which builds a Decline.
8. **T1 and T2 follow the lease.** The client uses the earliest T1 and T2
   of its IAs. An IA with T1 above T2, both above zero, is discarded, and
   so is an address or prefix whose preferred lifetime is above its valid
   one (sections 21.4, 21.6 and 21.22).
9. **A message that is not an answer is ignored.** The client takes only
   an Advertise or a Reply with its transaction identifier, its own DUID
   in the Client Identifier option and a Server Identifier option. In the
   states that name a server, a Reply from another server is ignored.
10. **A message is 1024 bytes at most.** A datagram the caller received
    that is longer must be dropped or truncated by the caller; the buffer
    cannot hold more. A relay message adds 34 bytes and an option header
    to the message it carries.
11. **A writer that answers false has left a message that is not
    usable.** The writers answer false when the buffer is full. The
    message built so far is discarded.
12. **A reader answers -1 or 0 for what is not there.** An option offset is
    -1 when the option is absent, a field read from a message that is too
    short is 0, and none of them traps. `validate` tells a message that
    cannot be used from one that can.
13. **A Reconfigure message is ignored.** Section 20.4 requires it to be
    authenticated, and this package does not verify authentication. The
    Reconfigure Message and Reconfigure Accept options are read and
    written.
14. **The client keeps one address and one prefix.** It holds the first
    usable IA Address of its IA_NA and the first usable IA Prefix of its
    IA_PD, two DNS servers, the first NTP server address, and the Domain
    Search List up to 128 bytes in label form.

## Running on a microcontroller

All three modules are written for a device with no heap allocator, and the
compiler checks that claim on every build. Every function is
`@tier(embedded)` and `@no_alloc`, so none of them reaches the allocator.
Every function performs no input or output and reads no clock, and none
declares an effect.

| What | Size |
| --- | --- |
| A message, `Vec[u8; 1024]` | 1032 bytes of the caller's stack or static storage |
| A DUID, `Vec[u8; 130]` | 144 bytes |
| `Dhcp6Client` | 1064 bytes |
| `Addr6` | 32 bytes |

The client needs a message buffer to receive into and one to build in, so
a device that is only a client holds about 3.2 KB. The same buffer can
serve for both when the caller is done with the received message before it
builds the reply.

`tests/embedded_probe.nv` is the claim as a program that either builds or
does not. It runs a client through a Solicit, an Advertise, a Request, a
Reply that grants an address, a prefix and two DNS servers, a Renew at T1
and a Release, against messages built in the same image, and prints
`PASS: dhcpv6-embedded`. The image built for `--target=nrf52-qemu` is
25,144 bytes of flash, of which the program is 16,954 bytes and the
runtime, libgcc and tables the rest. The worst-case stack through one call
is 4,432 bytes, and the probe uses no heap.

```bash
cp tests/embedded_probe.nv src/zz_probe_tmp.nv
novo build --target=nrf52-qemu src/zz_probe_tmp.nv
qemu-system-arm -machine mps2-an386 -nographic -semihosting -kernel zz_probe_tmp.elf
rm src/zz_probe_tmp.nv
```

The registry shows `embedded` among the tiers of this package because
every module checks clean at that tier.

## What is not included

- **A relay agent and a server** are NOT IMPLEMENTED. The relay form is
  read and written (`begin_relay`, `put_relay_message`, `relay_inner`), and
  the options a server puts in a Reply have writers, but the decisions of
  a relay agent and of a server, the lease database and the prefix pool
  are not here.
- **Verifying Authentication** is NOT IMPLEMENTED. The option is read:
  `dhcp6opt` answers its protocol, algorithm, replay detection method,
  replay value and the position of the information. Delayed
  authentication and the reconfigure key are not checked, and so a
  Reconfigure message is never acted on.
- **IA_TA in the client.** The IA_TA option is read and written. The
  client asks for non-temporary addresses and delegated prefixes.
- **More than one address or prefix in the client.** The readers and
  writers take any number; the client uses the first.
- **Sockets, Neighbor Discovery and Router Advertisement.** The caller
  sends and receives the datagrams and probes the address.
- **Options outside the list above**, such as the Client FQDN option. They
  can be read with `find_option`, `opt_len` and `u8_at`.
- **A lease database that survives a restart.**

## Related packages

- [dhcp-nv](https://novo-lang.org/packages/dhcp-nv) is the DHCPv4 client
  and server of the same shape.
- [dns-codec-nv](https://novo-lang.org/packages/dns-codec-nv) and
  [ntp-nv](https://novo-lang.org/packages/ntp-nv) are protocol codecs of
  the same shape. A device that has a lease usually asks them next, for a
  name and for the time.

## Tests

```bash
novo test tests/dhcp6_msg_tests.nv       # the header, options, relay form, DUIDs and writers
novo test tests/dhcp6_opt_tests.nv       # the options as values, the names and the lists
novo test tests/dhcp6_client_tests.nv    # the states, the timers and the refusals
novo test tests/dhcp6_capture_tests.nv   # six packets of a published capture
bash tests/coverage.sh                   # line coverage over src/, merged across the suites
```

The message vectors are written out by hand from the diagrams of RFC 8415
section 21 and the layouts of RFC 3646, RFC 5908 and RFC 6355, with the
field names beside the bytes. One suite reads the UDP payloads of packets
2, 5, 7, 8, 11 and 12 of `DHCPv6.pcap` from the Wireshark sample captures
page (a Solicit, an Advertise, a Request, a Reply, a Release and the Reply
to it, for a delegated prefix), rebuilds the Solicit and the Request byte
for byte, and runs the client through the exchange. The client tests set
the clock and the random word and write the server's messages themselves,
so each test says what the client was told.

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->

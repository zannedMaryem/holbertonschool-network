# Localhost, IP Addresses, and Network Interfaces

## Learning objectives

By the end of this guide, you should be able to:

- Explain what `localhost` and `127.0.0.1` mean.
- Explain what `0.0.0.0` means when a program listens for connections.
- Read and safely edit the `/etc/hosts` file.
- Display your machine's active network interfaces and their IP addresses.

The examples use Linux and IPv4 unless stated otherwise.

## 1. `localhost` and `127.0.0.1`

`localhost` is the conventional hostname for the **loopback interface**: a
virtual network interface that lets a machine communicate with itself. It
does not send packets out over Wi-Fi or Ethernet.

`127.0.0.1` is the standard IPv4 loopback address. The entire IPv4 block
`127.0.0.0/8` is reserved for loopback, but `127.0.0.1` is the address most
often used. IPv6 has its own loopback address, `::1`.

```text
Application on this computer
          |
          | connect to localhost (usually 127.0.0.1)
          v
    Loopback interface
          |
          +---- back to an application on this computer
```

For example, if a web server is listening only on `127.0.0.1:8000`, open
`http://127.0.0.1:8000` or `http://localhost:8000` **on that same machine**.
Other computers cannot use that loopback address to reach your server: their
own `127.0.0.1` points back to themselves.

Try resolving the name and requesting a local service:

```bash
getent hosts localhost
curl http://localhost:8000
```

The `curl` command succeeds only if something is listening on that port; an
error can simply mean no local web server is running.

## 2. `0.0.0.0`

`0.0.0.0` is the IPv4 **unspecified address**. Its meaning depends on context.
Most commonly, in a server's *bind/listen* address, it means: listen for
connections addressed to **any IPv4 interface on this machine**.

Suppose a machine has both loopback (`127.0.0.1`) and a LAN address
(`192.168.1.25`):

```text
Server listens on 127.0.0.1:8000
  -> connections to 127.0.0.1:8000 on this machine only

Server listens on 0.0.0.0:8000
  -> connections to 127.0.0.1:8000 and 192.168.1.25:8000
     (subject to firewall and network configuration)
```

For example, Python's development server can be started on loopback or on all
IPv4 interfaces:

```bash
python3 -m http.server 8000 --bind 127.0.0.1
python3 -m http.server 8000 --bind 0.0.0.0
```

Run one command at a time. With `127.0.0.1`, connect locally. With
`0.0.0.0`, a device on the same reachable network can connect using this
machine's LAN IP, for example `http://192.168.1.25:8000`, if firewalls allow.

| Address | Common server-listening meaning | Is it a client destination? |
| --- | --- | --- |
| `127.0.0.1` | Accept connections via this machine's IPv4 loopback | Yes, for a service on the same machine |
| `0.0.0.0` | Accept IPv4 connections on all local interfaces | No; use a specific reachable IP address instead |
| `192.168.1.25` | Accept connections on the interface/address using this LAN IP | Yes, from devices that can reach that LAN |

Listening on `0.0.0.0` can expose a service to more than just your own
computer. Use it only when that access is intended, and consider firewall
rules and the sensitivity of the service. Binding to it does not itself
configure a firewall or guarantee Internet accessibility.

## 3. The `/etc/hosts` file

`/etc/hosts` is a local text file that associates IP addresses with hostnames.
It can provide local name resolution without asking a DNS server. A common
entry maps `localhost` to the loopback address.

View it with:

```bash
cat /etc/hosts
```

Example:

```text
127.0.0.1   localhost
127.0.1.1   my-computer
::1         localhost ip6-localhost ip6-loopback
```

Each non-comment line has an IP address followed by one or more names.
Comments begin with `#`. For example, a local development name can be added
as:

```text
127.0.0.1   my-app.test
```

Then `my-app.test` resolves to this computer's loopback address:

```bash
getent hosts my-app.test
```

This only changes name resolution on this computer; it does not register the
name on the Internet or make a service available to other machines. The
system's name-service configuration determines whether `/etc/hosts` is
consulted before or after DNS. On Linux, `getent hosts` is useful because it
uses the system's configured name-resolution path.

Edit `/etc/hosts` only when needed, using administrator privileges, and
preserve existing entries. For example:

```bash
sudoedit /etc/hosts
```

## 4. Display active network interfaces

A **network interface** is a connection point through which the operating
system can send or receive network traffic. Examples include Ethernet,
Wi-Fi, loopback, and VPN interfaces. Interface names vary, but may look like
`lo`, `eth0`, `enp0s3`, or `wlan0`.

### List interfaces and addresses

Use `ip` (provided by the `iproute2` package) to inspect interfaces:

```bash
ip address
```

For a compact view:

```bash
ip -brief address
```

Illustrative output:

```text
lo       UNKNOWN  127.0.0.1/8 ::1/128
enp0s3   UP       192.168.1.25/24 fe80::1234:5678:abcd:ef01/64
```

Here, `lo` is loopback and `enp0s3` is an example network interface. `UP`
means the interface is administratively enabled; it does not by itself
guarantee Internet connectivity. An address ending in `/24` includes its
network prefix length (subnet information).

Other useful commands:

```bash
ip link show              # Link/interface state and hardware address
ip -brief link            # Compact link/interface state
ip route                  # Routes, including the default route if configured
```

`ifconfig -a` is another command you may encounter, but it belongs to the
older `net-tools` package and may not be installed. Prefer `ip` on modern
Linux systems.

### Quick reference

| What you want to know | Command |
| --- | --- |
| Interface names, states, and IP addresses | `ip -brief address` |
| Detailed interface addresses | `ip address` |
| Link state and hardware addresses | `ip link show` |
| Configured routes | `ip route` |
| How `localhost` resolves | `getent hosts localhost` |

## Key takeaways

- `localhost` is a hostname that normally resolves to a loopback address;
  `127.0.0.1` is the familiar IPv4 loopback address.
- Loopback traffic stays on the same machine.
- A server bound to `0.0.0.0` listens on all of its IPv4 interfaces; clients
  connect using a specific address, not `0.0.0.0`.
- `/etc/hosts` provides local hostname-to-address mappings.
- On Linux, `ip -brief address` is a quick way to see interface states and
  assigned addresses.
  
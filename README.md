# DHCP option injector
[![Build Status](https://travis-ci.org/misje/dhcpoptinj.svg?branch=dev)](https://travis-ci.org/misje/dhcpoptinj) [![Total alerts](https://img.shields.io/lgtm/alerts/g/misje/dhcpoptinj.svg?logo=lgtm&logoWidth=18)](https://lgtm.com/projects/g/misje/dhcpoptinj/alerts/)

Have you ever wanted to intercept DHCP requests and squeeze in a few extra DHCP
options, unbeknownst to the sender? Probably not. However, should the need ever
come, **dhcpoptinj** will (hopefully) help you.

## Why

There can be many a reason to mangle DHCP requests, although chances are you
ought to look for a much better method for solving your problem. Perhaps you do
not have access to the DHCP server/clients and need to modify their DHCP
options, perhaps the DHCP software is difficult to configure (or does not
support what you want to do), perhaps you have a very complex and/or odd setup,
or perhaps you just want to experiment sending exotic or malformed options?
There is a small chance that dhcoptinj might actually be of some use.

## How

dhcpoptinj waits for packets to arrive in a netfilter queue. It will ensure
that a packet is in fact a BOOTP/DHCP packet, and if so proceed to inject
options. It will recalculate the IPv4 header checksum, disable the UDP
checksum (for a simpler implementation) and then give the packet back to
netfilter.

You need an iptables rule in order to intercept packets and send them to
dhcpoptinj. Let us say you have two interfaces bridged together, *eth0* and
*eth1*. Let us say you want to intercept all BOOTP requests coming from *eth0*
and inject the [relay agent information
option](https://tools.ietf.org/html/rfc3046) (82/0x52). Let us make up a silly
payload: An [agent circuit ID
sub-option](https://tools.ietf.org/html/rfc3046#section-3.1) with the value
"Fjas".

Add a rule to the iptables mangle table:`sudo iptables -t mangle -A PREROUTING
-m physdev --physdev-in eth0 -p udp --dport 67 -j NFQUEUE --queue-num 42`.

Then run dhcpoptinj (let us run it in the foreground with extra debug output):
`sudo dhcpoptinj -d -f -q 42 -o'52 01 04 46 6A 61 73'`. Note that dhcpoptinj
must be run by a user with the CAP\_NET\_ADMIN capability. You do not need to,
and you really should not run dhcpoptinj as root. Instead, you can for instance
grant the CAP\_NET\_ADMIN capability to the binary (using *setcap*) and limit
execution rights to only a specific user or group. This is a method used for
running wireshark as non-root, so you will find several guides helping you
accomplish this.

Now send a DHCP packet to the *eth0* interface and watch it (using a tool like
[wireshark](https://www.wireshark.org/)) having been modified when it reaches
the bridged interface. It should have the injected option at the end of the
option list. If you capture the incoming DHCP packet with Wireshark, it will
appear unmodified although it will in fact be mangled.

Note the format of the argument to the *-o* option: It should be a hexadecimal
string starting with the DHCP option code followed by the option payload. The
option length (the byte that normally follows the option code) is automatically
calculated and must not be specified. The hex string can be delimited by
non-hexadecimal characters for readability. All options must have a payload,
except for the special [pad
option](https://tools.ietf.org/html/rfc2132#section-2) (code 0).

The layout of the nonsensical option used in this example (first the [DHCP
option layout](https://tools.ietf.org/html/rfc2132#section-2), then the
specific [relay agent information option sub-option
layout](https://tools.ietf.org/html/rfc3046#section-2.0)) is as follows:

| Code | Length |            Data            |
|------|--------|----------------------------|
|  52  | (auto) | 01 04 46 6A 61 73 ("Fjas") |

| Sub-opt. | Length |         Data         |
|----------|--------|----------------------|
|    01    |   4    | 46 6A 61 73 ("Fjas") |

Note that dhcpoptinj does not care about what you write in the option payloads,
nor does it check whether your option code exists. It does however forbid you
to use the option code 255 (the terminating end option). dhcpoptinj inserts
this option as the last option automatically.

## Screenshot
```
dhcpoptinj -f -d -q42 -r -o'0C 66 6A 61 73 65 68 6F 73 74' -o'52 01 04 46 6A 61 73' -o 320A141E28
```
```
3 DHCP option(s) to inject (with a total of 25 bytes): 12 (0x0C) (Hostname), 82 (0x52) (Relay Agent Information), 50 (0x32) (Address Request)
Existing options will be removed
Initialising netfilter queue
Initialising signal handler
Initialisation completed. Waiting for packets to mangle on queue 42
Received 416 bytes
Inspecting 328-byte DHCP packet from B6:40:FE:41:30:DC to 255.255.255.255:67
Mangling packet
Found option  53 (0x35) (DHCP message type)        DHCPREQUEST (copying)
Found option  54 (0x36) (DHCP Server Id)           with   4-byte payload 0A 14 1E 01 (copying)
Found option  50 (0x32) (Address Request)          with   4-byte payload 0A 14 1E 28 (removing)
Found option  12 (0x0C) (Hostname)                 with  12-byte payload 33 31 36 64 65 39 31 34 64 61 62 34 (removing)
Found option  55 (0x37) (Parameter List)           with  13-byte payload 01 1C 02 03 0F 06 77 0C 2C 2F 1A 79 2A (copying)
Found END option (removing)
Injecting option  12 (0x0C) (Hostname)                 with   9-byte payload 66 6A 61 73 65 68 6F 73 74
Injecting option  82 (0x52) (Relay Agent Information)  with   6-byte payload 01 04 46 6A 61 73
Injecting option  50 (0x32) (Address Request)          with   4-byte payload 0A 14 1E 28
Inserting END option
Padding with 10 byte(s) to meet minimal BOOTP payload size
Sending mangled packet

```

## Installing
[![Packaging status](https://repology.org/badge/vertical-allrepos/dhcpoptinj.svg)](https://repology.org/project/dhcpoptinj/versions)

dhcpoptinj is in Debian/Ubuntu. The deb package is under source control at
[salsa](https://salsa.debian.org/misje-guest/dhcpoptinj). Installing
dhcpoptinj from the deb package is recommended over the following manual
installation procedure, because it also includes a man page, bash completion
rules, example files etc. 

### Prerequisites

You need [cmake](http://www.cmake.org/) and
[libnetfilter\_queue](http://www.netfilter.org/projects/libnetfilter_queue/)
(and a C compiler that supports C99). Hopefully, you are using a Debian-like
system, in which case you can run the following to install them: `sudo apt-get
install cmake libnetfilter-queue-dev`.

### Build

1. Download or clone the source: `git clone git://github.com/misje/dhcpoptinj`
1. Enter the directory: `cd dhcpoptinj`
1. Create a build directory and enter it (optional, but recommended): `mkdir
	build && cd build`
1. Run cmake: `cmake ..` (or `cmake -DCMAKE_BUILD_TYPE=Debug ..` if you want a
	debug build)
1. Run make: `make -j4`
1. Install (optional, but you will benefit from having dhcpoptinj in your
	PATH): `sudo make install`

### Demolish

1. Run `sudo make uninstall` from your build directory

The build directory with all its contents can be safely removed. If you did not
use a build directory, you can get rid of all the cmake rubbish by running `git
clean -dfx`. Note, however, that this removes **everything** in the project
directory that is not under source control.

## Configuration file

dhcptopinj will attempt to parse /etc/dhcpoptinj.conf or the file passed with
-c/--conf-file. The syntax of the configuration file is
* **key=value**, where *key* is the long option name, or
* **key** if the option does not take an argument

Whitespace is optional. Anything after and including the character **#** is
considered a comment. DHCP options are listed one-by-one as *option=01:02:03*.
Quotes around the option hex string is optional, and the bytes may be separated
by any number of non-hexadecimal characters.

The options "options", *version*, *help* and *conf-file* are not accepted in a
configuration file.

Example:
```apache
# Run in foreground:
foreground
# Enable debug output:
debug
# Override hostname to "fjasehost":
option = '0C 66 6A 61 73 65 68 6F 73 74'
# Send agent ID "Fjas":
option = "52:01:04:46:6A:61:73"
# Override address request to ask for 10.20.30.40:
option=320A141E28
# Use queue 12:
queue = 12

remove-existing-opt # Remove options before inserting
```

## Help

This readme should have got you started. Also check out the man page (in the
deb package) and the help output (`dhcpoptinj -h`), which should cover
everything the utility has to offer.

For bugs and suggestions please create an issue.

### Limitations

dhcpoptinj is simple and will hopefully stay that way. Nonetheless, the
following are missing features that hopefully will be added some day:

1. Remove options instead of having to replace them
2. Filter incoming packets by their DHCP message type (code 53) before mangling
	them

### Troubleshooting

1. *Failed to bind queue handler to AF_INET: Operation not permitted*

	Most likely you do not have CAP\_NET\_ADMIN capability or there is another
	process (perhaps another dhcpoptinj instance?) bound to the same netfilter
	queue number.

### Known issues

1. Memory leak on non-normal exit.

	This is not considered a leak. However, there should be no memory leak on a
	normal exit (catching SIGTERM, SIGINT or SIGHUP).

## Useful information

When creating iptables rules to use with dhcpoptinj, the following options can
be useful:

-	`--queue-bypass`

	Do not drop packets, but let them pass through if dhcpoptinj is not running
	(or not listening on the correct queue number).

## Contributing

If you have any suggestions please leave an issue, and I will come back to you.
You are welcome to contribute and pull requests are much appreciated.

If you find dhcpoptinj useful I would love to hear what you are using it for.
Update the [wiki
page](https://github.com/misje/dhcpoptinj/wiki#practical-use-cases) and
describe your use.

## License

I have chosen to use GPL for this project. If that does not suit you, contact
me, and we can agree on a different license.

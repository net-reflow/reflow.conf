# Reflow: Configuration files

A format is specifically designed for reflow, so that you can express whatever you want to do with your network traffic, in an intuitive and terse way.
This repository demonstrates its capabilities, feel free to use it right away and make edits to suit your needs.

This README provides a description of the structure and syntax for your reference.

The first step in taking control of network traffic is to identify and sort them, but you won't want to list thousands of domains and ip addresses in a single configuration file. So they are prefix-matched, and listed in separate files(or directories), one for each `zone`.

# Zone

## Domain names

Domain name lists are maintained in the directory `namezone`, which is a sub-directory in your configuration directory.
To configure a zone "secret-sites" containing domains, create a plain text file named "secret-sites" in the directory `namezone`.

In that file, list domains one per line. Each domain should start with the root, which is the opposite of what you usually see in web browsers. For example:

    com.reddit.www # matches also subdomains
    com.twitter # m.twitter.com will be matched

If there're a lot of domains you want to put in a `zone`, you can also create a directory with the name of a `zone`, and create any number of text files inside it. In this case, what these text files are named don't matter. For example, create a directory for the zone `adservers`, inside it you can have text files `easylist.txt`, `adblock.list`, etc. and they will all be in the zone `adservers`.

The directory structure will be something like

    reflow.conf/namezone/secret-sites
    reflow.conf/namezone/adservers/easylist.txt
    reflow.conf/namezone/adservers/adblock.list

## IP Addresses

They are located in the directory `addrzone`, grouped into text files or directories, in the same way as domains.

In each file, IP addresses are written in CIDR notation in seperate lines, for example:

    192.168.100.0/22

Naturally, they are also prefix-matched.

# config

All other configuration is done in the file `config`.

## Egress

Methods to access the internet are called `egress`, including proxies and VPNs.
One proxy is usually used in more than one cases, to avoid repetition, they are given names, like variables in programming:

    egress privacyproxy = socks5 127.0.0.1:2123

A definition starts with the word egress, the right-hand side is the method (socks5 in this case) followed by its parameter (a socket address 127.0.0.1:2123 here). Another supported method is bind followed by an ip address, it allows you to choose a network interface by its ip, when you have more than one connection to the internet.

The names `direct` and `reset` are special built-in egresses, which means "use the existing default route" and "reject the connection", respectively.

## Rules

A rule is how you express the decision-making process in routing, in the tree-like structure. Because a rule may also be used several times, it's defined as variables as well, in the file `config` you can the definition for the rule "mytrafficrule" :

    rule mytrafficrule = any[
        # rules here omitted
    ]

## Relay

This is the core functionality and where it all comes together. The configuration starts with the word `relay`, followed by options in separate lines, enclosed in brackets.
    
    relay {
        rule=mytrafficrule
        listen=socks5 127.0.0.1:1080
        resolver=udp 127.0.0.1:53
    }

It serves as a socks5 proxy on 127.0.0.1:1080. It use previously defined rule "mytrafficrule" to determine what to do with each connection. A dns server can be set here optionally, by default it uses `8.8.8.8`

## Dns Proxy

DNS is an essential yet insecure part of the internet.
To make life easier, reflow gives you the option to selectively route some dns queries through a proxy. To start the dns proxy, add a `dns` section:

    dns {
        listen=udp 127.0.0.1:5353
        forward= {
            secret-sites => privacyproxy|udp 1.1.1.1:53
            else => udp 192.168.1.1:53
        }
    }

This starts a dns service on 127.0.0.1:5353, when you set it as your DNS server, queries of domains listed in `secret-sites` (and their sub-domains) will be forwarded to 1.1.1.1:53 over the proxy defined as privacyproxy; all other domains use a default server.

# Use as your default gateway

Currently, the relay only provides a socks5 server, to take control of all traffic, it needs to be combined with [tun2socks](https://github.com/ambrop72/badvpn/wiki/Tun2socks).
Create a `tun` device, tell tun2socks to redirect traffic on the `tun` device to the socks5 server defined in the `relay` section of `config`.


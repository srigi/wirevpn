# Wirevpn

A set of [Ansible](https://www.ansible.com/resources/get-started) playbooks & roles to setup a small **VPN** with [Wireguard](https://www.wireguard.com/quickstart/) and [dnsmasq](https://en.wikipedia.org/wiki/Dnsmasq). Totally suited to my personal setup, open sourced for inspiration.

## Motivation

I enjoy the luck of having a **public IPv4 address** from my ISP. With that I can setup a small personal VPN to interconnect my personal devices while on the road. Something, that public VPNs doesn't give you!

I use small, [palm-sized PC](https://www.ecs.com.tw/en/Product/LIVA/LIVA-XE/overview) for the the VPN (and DNS) server. Machine is running a minimal **Ubuntu server 20.04** installation, using [mini.iso](http://archive.ubuntu.com/ubuntu/dists/focal/main/installer-amd64/current/legacy-images/netboot/).

In the previous _incarnation_ of the VPN, I set up everything by hand. This time I want the machine to be controlled by Ansible.

## Network map

```
       Internet
         ┌─┐
         └┼┘
          │
          │
  │ │ │   │      ┌──────────┐
  └─┼─┘   │      │          │
    │     │      │   ┌───┐  │
WiFi│  WAN│      │LAN│   │  │
    │    ┌┼┐    ┌┼┐ ┌┼┐  │  │
    │ ┌──┴─┴────┴─┴─┴─┴┐ │  │
    └─┤ Router         │ │  │
      │                │ │  │
      └────────────────┘ │  │
                         │  │
                    ┌────┘  │
        192.168.0.99│       │
                   ┌┼┐      │
        ┌──────────┴─┴─┐    │
        │ VPN server   │    │
        │ (ecs-liva)   │    │
        └──────────────┘    │
                            │
                            │
   ____________            ┌┼┐
  │  ________  │      ┌────┴─┴──┐
  │ │Work    │ │      │ ┌─────┐ │
  │ │machine │ │      │ │NAS  │ │
  │ │(connor)│ │      │ │(lilith)
  │ │________│ │      │ │     │ │
  │____________│      │ ├─────┤ │
  / [][][][][] \      │ ├─────┤ │
 / [][][][][][] \     │ ├─────┤ │
( [][][____][][] )    └─┴─────┴─┘
 \______________/

Hostnames in (parentheses)
```

Router is configured to reserve local LAN address `192.168.0.99` for VPN server and forward public port `51280` to it. This makes the VPN server accessible from pubic internet using my public IP address.

Use-cases like watching movies or accessing files from the NAS, while outside the apartment, are possible.

## Requirements

### VPN server

- Prepare minimal installation of **Ubuntu server 20.04** on the VPN server machine. Don't create swap partition or swap file during installation (we'll do that via Ansible).
- Install `openssh-server` onto VPN server and setup SSH access using SSH key(s) (`ssh-copy-id` [is your friend](https://www.ssh.com/academy/ssh/copy-id)).

### Work machine

- Ansible (tested v5.2)
- OpenSSL cli, to generate _"[crypto rands](https://en.wikipedia.org/wiki/Cryptographically-secure_pseudorandom_number_generator)"_
- Wireguard cli, to generate VPN client(s) [key-pairs](https://en.wikipedia.org/wiki/Public-key_cryptography)

## VPN design

- Subnet `10.2.0.0/24` (configurable)
- Static assignment of IP addresses to VPN clients (configured & versioned in this repository)
  - VPN server: `10.2.0.1`
  - NAS: `10.2.0.10`
  - work machine: `10.2.0.11`
  - any other device(s) (smartphone, tablet), continuation of this series
- Important devices (work machine, NAS) accessible via assigned DNS names, no need to remember IP addresses 🙇
  - work machine: `connor.wirevpn`
  - NAS: `lilith.wirevpn`

## Setting up the configuration

We will store sensitive data (private keys, pre-shared keys) into Ansible vault. That way, these sensitive data can be versioned safely.

1. Generate [Ansible vault password file](https://docs.ansible.com/ansible/latest/user_guide/vault.html#storing-passwords-in-files):
   ```
   openssl rand -base64 32 > vault_pass
   ```
2. Generate _preshared key_ file(s) for each VPN client (except VPN server)
   ```
   openssl rand -base64 32 > .data/connor.presharedkey
   ```
3. Generate wireguard key-pair for each device (except VPN server)
   ```
   wg genkey | tee .data/connor.privkey | wg pubkey > .data/connor.pubkey
   ```
4. Create & edit Ansible vault
   ```
   ansible-vault create group_vars/wireguard_servers/vault.yml
   ```
   Setup content of the vault by looking on [group_vars/wireguard_servers/vault.yml [example]](group_vars/wireguard_servers/vault.yml). Fill all the values for VPN clients from files generated above ⬆
5. Finish configuration by editing [group_vars/wireguard_servers/vars.yml](group_vars/wireguard_servers/vars.yml)
6. You can now safely delete generated files, except `vault_pass` file!

If you ever need to update contents of Ansible vault, use command bellow (original `vault_pass` must be present!):

```
ansible-vault edit group_vars/wireguard_servers/vault.yml
```

## Execute Ansible playbooks
`//TODO`
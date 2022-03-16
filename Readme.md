# Wirevpn

A set of [Ansible](https://www.ansible.com/resources/get-started) playbooks & roles to setup a small **VPN** with [Wireguard](https://www.wireguard.com/quickstart/) and [dnsmasq](https://en.wikipedia.org/wiki/Dnsmasq). Totally suited to my personal setup, open sourced for inspiration.

## Motivation

I enjoy the luck of having a **public IPv4 address** from my ISP. With that I can setup a small personal VPN to interconnect my personal devices while on the road. Something, that public VPNs doesn't give you!

I use small, [palm-sized PC](https://www.ecs.com.tw/en/Product/LIVA/LIVA-XE/overview) as a VPN (and DNS) server. Machine is running **Ubuntu server 20.04** installation, using "[Manual server installation](https://ubuntu.com/download/server)" method.

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

Router is configured to reserve local LAN address `192.168.0.99` for the VPN server and forward public port `51280` to it. This makes the VPN server accessible from public Internet using my public IP address.

Use-cases like watching movies or accessing files from the NAS, while outside the apartment, are possible.

## Requirements

### VPN server

- Prepare **Ubuntu server 20.04** installation on the VPN server machine. You don't need to setup the swap partition or swap file during the installation (we'll do that via Ansible).
- Install `openssh-server` onto VPN server and setup SSH access using SSH key(s) (`ssh-copy-id` [is your friend](https://www.ssh.com/academy/ssh/copy-id)).

### Work machine

- Ansible (tested v5.2 & v5.3)
- OpenSSL cli, to generate _"[crypto rands](https://en.wikipedia.org/wiki/Cryptographically-secure_pseudorandom_number_generator)"_
- Wireguard cli, to generate VPN client(s) [key-pairs](https://en.wikipedia.org/wiki/Public-key_cryptography)

## VPN design

- Subnet `10.2.0.0/24` (configurable)
- Static assignment of IP addresses to VPN clients (configured & versioned in this repository)
  - VPN server: `10.2.0.1`
  - NAS: `10.2.0.10`
  - work machine: `10.2.0.11`
  - any other device(s) (smartphone, tablet), continuation of this series
- Important devices (work machine, NAS) accessible via assigned DNS names, no need to remember IP addresses 💡
  - VPN server: `ecs-liva.wirevpn`
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

   Setup content of the vault by looking on [`group_vars/wireguard_servers/vault.yml [example]`](group_vars/wireguard_servers/vault.yml%20[example]). Fill all the needed values for VPN clients from credentials generated above ⬆

   **_Don't lose generated `vault_pass` file!_** You will lose your **vault** if you lose `vault_pass` used for creating it via `ansible-vault create` command.

If you ever need to update contents of Ansible vault, use command bellow (original `vault_pass` must be present!):

```
ansible-vault edit group_vars/wireguard_servers/vault.yml
```

5. Finish VPN configuration by editing [`group_vars/wireguard_servers/vars.yml`](group_vars/wireguard_servers/vars.yml)

6. Create the inventory file by copying [`@inventory [example]`](@inventory%20[example]) into `@prod` and editing its content

   Learn more about inventories in [Ansible documentation](https://docs.ansible.com/ansible/latest/user_guide/intro_inventory.html).

## Execute Ansible playbooks

Playbooks must be executed in this particular order:

1. First we execute `postinstall` playbook. It will perform couple of tasks:

   - install latest Ubuntu updates
   - setup a swapfile
   - harden SSH server configuration (disable password authentication)

   ```
   ansible-playbook playbooks/postinstall.yml -i @prod
   ```

   You can run system update on VPN server anytime, by limiting executed postinstall tasks via tag

   ```
   ansible-playbook playbooks/postinstall.yml -i @prod -t upkeep
   ```

2. Install and setup Wireguard

   ```
   ansible-playbook playbooks/wireguard.yml -i @prod
   ```

   Wireguard is now up & running on VPN server machine, ready to accept client(s) connection.

3. Install and setup dnsmasq

   ```
   ansible-playbook playbooks/dnsmasq.yml -i @prod
   ```

   Make sure you configure the main LAN adapter (interface) correctly in [`group_vars/dns_servers.yml`](group_vars/dns_servers.yml)! You can list all network interfaces on VPN server by running `ip addr` command.

After executing all three playbooks, please restart the VPN server machine.

## Gathering client `.conf` files

By executing the playbooks against the inventory file, you setup and configured Wireguard and dnsmasq on VPN server machine. You also need to configure your VPN clients. Part of the configuration is located in [`group_vars/wireguard_servers/vars.yml`](group_vars/wireguard_servers/vars.yml). However you need to create `.conf` files, that can be imported into Wireguard GUI clients, that will be connecting to your VPN server.

// TODO

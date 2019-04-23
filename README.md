# boringtun-vps-config
Setting up wireguard on a tiny VPS can be difficult, especially if you're on OpenVZ with a 2.6 kernel. These instructions will hopefully make things a little easier. This assumes a road warrior configuration with full traffic routed to the internet.

For the entirety of this guide, we'll be using 172.16.1.0/24. Please select a different subnet. Also note that any reference to venet0 is the main interface for the machine (like eth0 would usually be). The IP assigned to that interface shall be shown as 10.1.1.1. Change this to your own IP in the files. Port 13200 will be used as the UDP listening port for boringtun.

# Dependencies
Before you start, you should:

* Install rust with rustup
* Install tmux/byobu and configure it however you like
* Secure SSH and harden the system a bit
* Install iptables-persistent

## Setting up wireguard

```bash
sudo add-apt-repository ppa:wireguard/wireguard
sudo apt-get install wireguard-dkms wireguard-tools
(umask 077 && printf "[Interface]\nPrivateKey = " | sudo tee /etc/wireguard/wg0.conf > /dev/null)
wg genkey | sudo tee -a /etc/wireguard/wg0.conf | wg pubkey | sudo tee /etc/wireguard/publickey
```

Then edit wg0.conf until it looks something like this:

```
[Interface]
PrivateKey = s...=
ListenPort = 13200

[Peer]
PublicKey = R...=
AllowedIPs = 172.16.1.2/32

[Peer]
PublicKey = G...=
AllowedIPs = 172.16.1.3/32
```

## Build and install boringtun
We need to build boringtun from trunk as of 20190422. This may change, at which point installing through cargo should be fine.

```bash
git clone https://github.com/cloudflare/boringtun.git
cd boringtun
cargo build --bin boringtun --release
cargo install --path .
```

### Allow boringtun to TUN

```bash
setcap cap_net_admin+epi boringtun
```

Interestingly this isn't enough to get it to work on OpenVZ. If you look at the systemd script, you need `--disable-multi-queue` as well.

### Systemd and other init
We want all this junk to work on boot.

#### Systemd

Now, add boringtun.service to /etc/systemd/system/ and perform the following:

```bash
systemctl enable boringtun
systemctl start boringtun
```

#### Network Interface

Create /etc/network/interfaces.tail and add this to it, replacing with your wg0 address:

```
auto wg0
iface wg0 inet static
        address 172.16.1.1
        netmask 255.255.255.0
```

This will come up at boot, but for now, just do ifup wg0

## iptables
See rules.v4 for example rules. The important ones are the UDP rule to let traffic in, the nat rules and ssh. Use iptables-persistent to manage the file so it gets restored at boot.

## Create a client configuration
Make a file with this in it:
```
[Interface]
PrivateKey = K...=
Address = 172.16.1.2

[Peer]
PublicKey = v...=
Endpoint = the.wg.server:13200
AllowedIPs = 0.0.0.0/0, ::/0
```
Now qr code it (note: doesn't work well in tmux for some reason):

```bash
qrencode -t ansiutf8 < client.conf
```

You may have to tweak your DNS server values in the wireguard client.

## Connect!
Time to connect.

```
[22:34:54] Resolved the.wg.server to 10.1.1.1
[22:34:54] Blocking standard DNS on all adapters
[22:34:54] Added Route 0.0.0.0/1  =>  172.16.1.1
[22:34:54] Added Route 128.0.0.0/1  =>  172.16.1.1
[22:34:54] Sending handshake...
[22:34:54] Connection established. IP 10.1.1.1
```

Check the server now:

```
interface: wg0
  public key: v...=
  private key: (hidden)
  listening port: 13200

peer: R...=
  endpoint: 1.2.3.4:5216
  allowed ips: 172.16.1.2/32
  latest handshake: 12 seconds ago
  transfer: 76.21 KiB received, 311.02 KiB sent
```
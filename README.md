# GlobalProtect with a client certificate, through NetworkManager

A step by step for connecting to a Palo Alto **GlobalProtect** portal that
requires a **client certificate**, using NetworkManager rather than the vendor
client — so the tunnel is managed like every other connection on the machine,
needs no `sudo`, and hands its DNS to `systemd-resolved`.

There are two ways to do this on Linux, and this guide is about the second.
`gpclient` — the CLI of `globalprotect-openconnect` — handles client
certificates perfectly well (`-c`, `-k`, `-p`, plus `--as-gateway` to skip the
portal round trip), and if you want the shortest path, use it. What it costs is
a `sudo` password on every connect *and* every disconnect, no connection state
beyond a PID file, no autoconnect, and a DNS default that sends every query —
public names included — to your corporate resolvers.

NetworkManager trades that for an administered tunnel: no admin password at
all, DNS scoped to your internal domain, state and autoconnect like any other
connection. It has one limitation `gpclient` does not — see
[Limits](#limits) — so check that before committing to it.

> **The certificate and the password are two independent factors, not
> alternatives.** The certificate operates at the TLS layer of every HTTPS
> request to the portal; the user and password go in the body of the login
> request. The server may return a username derived from the certificate, and
> openconnect deliberately ignores it. Expect to supply both.

## Contents

- [Why NetworkManager](#why-networkmanager)
- [Dependencies](#dependencies)
- [Step by step](#step-by-step)
- [What the settings mean](#what-the-settings-mean)
- [Troubleshooting](#troubleshooting)
- [Limits](#limits)

## Why NetworkManager

Measured against the same portal, with a certificate, on the same machine:

| | `gpclient` | NetworkManager |
|---|---|---|
| client certificate | `-c` / `-k` / `-p` | `usercert` / `userkey` |
| certificate in the **tunnel** phase | **yes** | **no** — see [Limits](#limits) |
| straight to the gateway | `--as-gateway` | `gateway=host/path` |
| admin password | **one to connect, one to disconnect** | **none** |
| portal password | one | one |
| DNS to `systemd-resolved` | yes | yes |
| DNS scoped to your domain | no — `~.`, every query goes to the VPN | yes, via `ipv4.dns-search` |
| connection state | `/var/run/gpclient.lock` holds a PID | `nmcli`, by name |
| autoconnect | no | NetworkManager's own flag |

The DNS row is fixable on the `gpclient` side with a `vpnc-script` wrapper that
sets `CISCO_SPLIT_DNS` and calls `resolvectl default-route <if> no`; its own
script never issues the second call, so the link stays a default route and
public queries race between it and your real one. The admin password is not
fixable: `gpclient` needs root to make the tunnel, and even `gpclient
disconnect` fails silently without it.

So: `gpclient` if you want the shortest path and do not mind the password,
NetworkManager if you want the tunnel administered like everything else.

## Dependencies

### What each piece is for

| Piece | Why it is needed |
|---|---|
| **NetworkManager** ≥ 1.16 | owns the connection, the tun device and the routes |
| **openconnect** ≥ 8.00 | speaks the GlobalProtect protocol; `--protocol=gp` landed in 8.00 (January 2019) |
| **openconnect plugin, service half** | lets NetworkManager drive openconnect. Installs `nm-openconnect-service` and the `.name` file that registers the VPN type |
| **openconnect plugin, GUI half** | the **auth dialog**. This is the piece that presents your certificate and asks for your password. Without it a connection that needs a login has nothing to ask with |
| **systemd-resolved** | optional but recommended: lets the tunnel's resolvers be scoped to your internal domain instead of catching every query |
| **openssl** | only used by `gp-nm-setup` to validate your certificate before writing anything |

> **The plugin is two packages on most distros, and the one people miss is the
> GUI half.** The service half alone gets you a connection that exists, fails
> to activate, and says nothing useful about why.

### Install

| Distro | Command |
|---|---|
| **Arch** | `sudo pacman -S networkmanager openconnect networkmanager-openconnect` |
| **Debian / Ubuntu** | `sudo apt install network-manager openconnect network-manager-openconnect-gnome` |
| **Fedora / RHEL** | `sudo dnf install NetworkManager openconnect NetworkManager-openconnect-gnome` |

On Arch, `networkmanager-openconnect` is the GUI half and pulls in
`networkmanager-vpn-plugin-openconnect`, the service half, as a dependency.
On Debian and Fedora the `-gnome` package is the GUI half and does the same.
In every case, installing the name above gets you both.

Despite the `-gnome` and GTK dependencies, none of this needs a GNOME session.
The auth dialog is a GTK window and runs fine under any desktop.

### Check

```bash
scripts/gp-nm-check
```

Verifies all of the above, names the file it found for each piece, and prints
the install command for whatever is missing. It reads nothing, writes nothing
and asks for no privileges.

```
NetworkManager
  ok    nmcli 1.58.1

openconnect plugin for NetworkManager
  ok    service plugin: /usr/lib/NetworkManager/VPN/nm-openconnect-service.name
  ok    auth dialog: /usr/lib/nm-openconnect-auth-dialog
  warn  plugin does not declare supports-safe-private-file-access

openconnect
  ok    openconnect v9.21
  ok    GlobalProtect protocol supported

DNS
  ok    systemd-resolved is running

certificate tooling
  ok    openssl 3.6.4

ready — with 1 warning(s)
```

That warning is expected and is not a problem — see
[Keep the profile system-wide](#keep-the-profile-system-wide).

## Step by step

### 1. Put the certificate and key somewhere stable

NetworkManager stores **paths**, not contents, and reads them at connect time
as root. Anywhere readable works; the files should not be world-readable.

```bash
mkdir -p ~/.config/gp-vpn
chmod 700 ~/.config/gp-vpn
cp client.crt client.key ~/.config/gp-vpn/
chmod 600 ~/.config/gp-vpn/client.key
```

Certificate and key may live in one PEM file or two. If your certificate came
as PKCS#12 (`.p12` / `.pfx`), split it first:

```bash
openssl pkcs12 -in client.p12 -clcerts -nokeys  -out ~/.config/gp-vpn/client.crt
openssl pkcs12 -in client.p12 -nocerts -nodes   -out ~/.config/gp-vpn/client.key
chmod 600 ~/.config/gp-vpn/client.key
```

`-nodes` leaves the key without a passphrase. A passphrase works too, but you
will be asked for it on every connect, on top of the portal password.

**Include the issuing chain.** The gateway validates the whole path, and a
leaf-only PEM is a common cause of a certificate that the portal accepts and
the gateway rejects. If you have the intermediates, append them to the
certificate file.

### 2. Create the profile

```bash
scripts/gp-nm-setup
```

It asks for the connection name, the portal host and the two file paths; checks
the certificate — expiry, chain, whether the key actually matches, file
permissions — prints the exact `nmcli` command, and runs it once you confirm.
`--dry-run` prints and stops.

Everything can also be given as flags:

```bash
scripts/gp-nm-setup \
  --name "Work VPN" \
  --portal vpn.example.com \
  --cert ~/.config/gp-vpn/client.crt \
  --key  ~/.config/gp-vpn/client.key
```

The equivalent by hand, which is what the script prints:

```bash
nmcli connection add \
  connection.id 'Work VPN' \
  connection.type vpn \
  connection.autoconnect no \
  vpn.service-type org.freedesktop.NetworkManager.openconnect \
  vpn.data 'protocol=gp,gateway=vpn.example.com/gateway,authtype=cert,usercert=/home/you/.config/gp-vpn/client.crt,userkey=/home/you/.config/gp-vpn/client.key,pem_passphrase_fsid=no,enable_csd_trojan=no,disable_udp=no,prevent_invalid_cert=no,reported_os=linux-64,cookie-flags=2,gateway-flags=2,gwcert-flags=2,resolve-flags=2' \
  ipv4.method auto ipv6.method auto
```

Note there is **no username setting**. The openconnect plugin stores none; the
auth dialog asks for it.

#### Keep the profile system-wide

Do not add `connection.permissions`. NetworkManager 1.58 added a protection:
a *private* connection — one restricted to a user — that references
certificates or keys on the filesystem is refused unless the VPN plugin
declares `supports-safe-private-file-access=true` in its `.name` file.
No released openconnect plugin declares it, so a per-user profile fails with:

```
The 'openconnect' plugin doesn't support private connections.
```

Without permissions the profile is system-wide and NetworkManager reads the
files as root. This is also why the same `vpn.data` can work on an older
machine and fail on a newer one.

### 3. Connect

```bash
nmcli connection up 'Work VPN'
```

A window opens, presents the certificate, and asks for your portal user and
password. One prompt — see [the gateway path](#the-gateway-path) for why it is
not two.

### 4. Verify

```bash
nmcli -g GENERAL.STATE,IP4.ADDRESS connection show 'Work VPN'
```

`activated` and an address from the VPN pool. To see the tunnel itself:

```bash
# The interface. A VPN connection reports its PARENT device in
# GENERAL.DEVICES — the wifi or ethernet underneath — so find the interface
# carrying the tunnel address instead.
addr=$(nmcli -g IP4.ADDRESS connection show 'Work VPN' | head -1 | cut -d/ -f1)
ip -4 -o addr show | awk -v a="$addr" '$4 ~ "^"a"/" {print $2}'
```

```bash
ip -4 route show dev <iface> | wc -l     # the split-tunnel routes
resolvectl status <iface>                # the DNS the portal pushed
```

### 5. Scope the DNS

GlobalProtect portals often push resolvers but no search domain. With no
routing domain, `systemd-resolved` marks the link `Default Route: yes` and
sends **every** query — public names included — to your corporate resolvers.

```bash
nmcli connection modify 'Work VPN' ipv4.dns-search internal.example.com
```

**List every internal domain, not just the obvious one.** Scoping is exact: a
name under a domain you left out stops resolving. While the link is a default
route this is invisible, because everything reaches the corporate resolvers
anyway — so the breakage appears at the moment you fix the scoping, which makes
it look like the fix caused it. Separate several with commas:

```bash
nmcli connection modify 'Work VPN' ipv4.dns-search 'corp.example.com,ad.example.com'
```

Reconnect, then confirm the split:

```
$ resolvectl query intranet.corp.example.com   # -> link: <vpn iface>
$ resolvectl query www.google.com              # -> link: <wifi iface>
```

Run the public one a few times. With two links claiming the default route
`systemd-resolved` queries both and keeps whichever answers first, so a single
sample can look right by luck.

## What the settings mean

| Key | Meaning |
|---|---|
| `protocol=gp` | GlobalProtect. The same plugin also speaks anyconnect, pulse, f5, fortinet |
| `gateway=HOST/PATH` | the portal host, and the login path — see below |
| `authtype=cert` | authenticate with a client certificate |
| `usercert` / `userkey` | paths, read by NetworkManager as root at connect time |
| `pem_passphrase_fsid=no` | do not derive the key passphrase from the filesystem ID |
| `enable_csd_trojan` | send a HIP report. `no` unless your gateway enforces it |
| `disable_udp=no` | allow ESP over UDP. Set `yes` where UDP is blocked |
| `prevent_invalid_cert=no` | still prompt on a server certificate you have not trusted yet, rather than failing outright |
| `reported_os=linux-64` | what the portal is told it is talking to. Some portals apply a different policy per OS; `win` is the usual fallback |
| `*-flags=2` | `NM_SETTING_SECRET_FLAG_NOT_SAVED` for `cookie`, `gateway`, `gwcert` and `resolve` |

### The gateway path

GlobalProtect exposes **two interfaces on the same host**: the portal and the
gateway. Given the host alone, openconnect treats it as the portal — it
authenticates there, asks for the list of gateways, and authenticates again.
Two password prompts for one connection.

`openconnect(1)` documents `--usergroup=X` as identical to
`https://host/X`, and networkmanager-openconnect exposes no `usergroup`
setting, so the login path rides along in the gateway field:

```
gateway=vpn.example.com/gateway
```

That is what `--gateway-path` controls, and `gateway` is the usual value. Pass
`--gateway-path ''` to go through the portal instead, which you want if your
portal offers several gateways and you need to pick one.

### The secrets that are not settings

`cookie`, `gateway`, `gwcert` and `resolve` carry `flags=2`,
`NM_SETTING_SECRET_FLAG_NOT_SAVED`. They are not configuration: they are the
*result* of authenticating against the portal. NetworkManager asks a session
secret agent for them at every connect, which is why there is a login window
every time and why nothing is cached between connections.

Your desktop needs to be running such an agent. GNOME and KDE ship one.
`nm-applet` is one. If yours does not, activation fails with:

```
No agents were available for this request.
```

and you will need to run one, or drive the auth dialog yourself.

## Troubleshooting

**`The 'openconnect' plugin doesn't support private connections.`**
The profile has `connection.permissions` and NetworkManager is 1.58 or newer.
Remove them: `nmcli connection modify 'Work VPN' connection.permissions ''`.

**`No agents were available for this request.`**
No secret agent in the session. See above.

**Two password prompts.** The gateway path is missing. Set
`gateway=host/gateway` rather than `gateway=host`.

**`Not authorized to control networking.`**
Your user is not in a group polkit grants network control to — usually
`wheel` — or the session is not local and active. `loginctl session-status`
shows the session state.

**The tunnel is up, routes are right, internal names do not resolve.**
Two causes, in this order. First, the portal pushed no search domain: see
[Scope the DNS](#5-scope-the-dns). Second, something is overriding DNS for
every connection on the machine — look for a `[global-dns]` section:

```bash
grep -rl global-dns /etc/NetworkManager/conf.d/
```

`NetworkManager.conf(5)` is explicit that those settings *"override the
equivalent options defined in individual connections, making them to be
ignored"*. If a distro or a config drop-in sets it, the VPN's DNS is discarded
along with everyone else's. `gp-nm-check` flags an active one.

**The tunnel is up and TCP connections inside it hang.**
GlobalProtect never negotiates an MTU — the server config carries `<mtu>0</mtu>`
— so a path with a smaller MTU goes undetected. This is the most frequently
reported GlobalProtect problem. There is no NetworkManager key for it; lower
the interface MTU after connecting:

```bash
sudo ip link set dev <iface> mtu 1400
```

**It connects, but you cannot reach anything.**
The gateway may require a HIP report. Many portals ask for one, accept
whatever arrives, and silently withhold network access when it is missing. Set
`enable_csd_trojan=yes` (`gp-nm-setup --hip`).

**The certificate is rejected although the portal accepted it.**
Usually an incomplete chain. Append the intermediates to the certificate file.
See also [Limits](#limits) — the tunnel phase may not be getting the
certificate at all.

## Limits

**The certificate reaches authentication, not the tunnel.** The auth dialog
presents your certificate while talking to the portal. The openconnect process
NetworkManager then starts for the tunnel is invoked with the session cookie
and no `-c`/`-k`:

```
openconnect --protocol gp --servercert pin-sha256:... --cookie-on-stdin \
            --script ... --interface vpn0 https://vpn.example.com/
```

If your gateway requires mutual TLS in the tunnel phase too, it will
authenticate and then fail to connect. There is no NetworkManager setting that
fixes this.

**`gpclient` does not have this limitation**, which is the one concrete reason
to prefer it. Its log shows the certificate being installed for the tunnel
phase as well:

```
INFO openconnect::ffi  Setting client certificate: /path/client.crt
INFO openconnect::ffi  POST https://vpn.example.com/ssl-vpn/getconfig.esp
INFO openconnect::ffi  Using client certificate 'name-from-the-cert'
```

So if your gateway wants the certificate in both phases, use `gpclient`.
Portals that require it only to authenticate — the common case — work either
way.

**Every connect is a fresh login.** By design: the cookie is `NOT_SAVED`.
Autoconnect at boot will not work for the same reason.

**HIP reports are minimal.** The plugin's HIP support reports what openconnect
can determine; a gateway that actually verifies HIP content may not accept it.

## Files

```
scripts/gp-nm-check    verify every dependency, name what is missing
scripts/gp-nm-setup    build the profile, after validating the certificate
```

Both are plain bash, need no privileges, and `gp-nm-setup --dry-run` prints the
`nmcli` command without touching anything.

## Verified on

Arch Linux, NetworkManager 1.58.1, networkmanager-openconnect 1.2.10,
openconnect 9.21, systemd 261 — against a GlobalProtect portal requiring
certificate, username and password, with a 113-route split tunnel.

## License

MIT — see [LICENSE](LICENSE).

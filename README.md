# Tailscale Mesh VPN Home Lab

A small lab connecting a Windows VM and an Ubuntu VM over a Tailscale (WireGuard) mesh VPN, then using it for secure remote administration over SSH.

## Environment

| Component | Details |
|---|---|
| Hypervisor | Oracle VirtualBox (both VMs on NAT networking) |
| Node 1 | Windows VM |
| Node 2 | Ubuntu 26.04 LTS VM |
| VPN | Tailscale (WireGuard-based mesh VPN), free Personal plan |
| Remote access | OpenSSH server on Ubuntu, built-in OpenSSH client on Windows |

## What I built

1. Created a Tailscale tailnet and joined both VMs to it.
2. Verified connectivity at three layers:
   - **Tailscale layer:** `tailscale ping` confirmed a **direct peer-to-peer connection over IPv6** (no relay), typically under 35 ms.
   - **IP layer:** standard ICMP `ping` from Ubuntu to Windows: 0% packet loss, ~4–10 ms average. The TTL of 128 confirmed the responder was the Windows host (Linux defaults to 64).
   - **Name resolution:** MagicDNS resolved the short hostname to the full tailnet name (`<host>.<tailnet>.ts.net`).
3. Installed and enabled OpenSSH server on Ubuntu and connected from Windows over the tailnet.
4. Before trusting the connection, **verified the SSH host key fingerprint** on the Ubuntu server against the one presented to the Windows client, which guards against man-in-the-middle attacks.

## Screenshots

**Both machines connected to the tailnet**

![tailscale status](tailscale-status.png)

**Direct peer-to-peer connection confirmed with `tailscale ping`**

![tailscale ping](tailscale-ping.png)

**Standard ICMP ping using the MagicDNS name: 0% packet loss, TTL 128 (Windows)**

![ping test](ping-test.png)

**Verifying the SSH host key fingerprint on the server**

![fingerprint check](fingerprint-check.png)

**SSH session from Windows into Ubuntu over the tailnet**

![ssh session](ssh-session.png)

## Troubleshooting log

| Problem | Diagnosis | Fix |
|---|---|---|
| `apt` reported `Unable to locate package tailscale` after running the official install script | `sudo apt update` output had no line for `pkgs.tailscale.com`, so the repository had never been added. The VM runs Ubuntu 26.04 ("resolute"), which the script didn't set up a repository for | Manually added Tailscale's signing key and the Ubuntu 24.04 ("noble") repository, ran `apt update`, then installed the package |
| `curl: (35) Recv failure: Connection reset by peer` while downloading the signing key | The TLS handshake was being reset. Checked whether the issue was VM-specific or network-wide | Re-ran the download and it completed |
| Commands failed when pasted as one line (`sh: cannot open sudo`) | Two commands were joined, so `sh` treated `sudo` as a script file | Ran each command separately |
| `tailscale` not recognised in PowerShell on Windows | The PowerShell session was opened before installation, so it hadn't picked up the updated PATH | Opened a new session (fallback: call `C:\Program Files\Tailscale\tailscale.exe` directly) |
| `tailscale ping desktop-mohamed` failed with a DNS lookup error | Wrong hostname. `tailscale status` showed the Windows node registered under a different name | Pinged by the 100.x address, then by the correct MagicDNS name |
| After restoring an older VirtualBox snapshot of the Windows VM, the tailnet showed two Windows devices: the original (offline) and a new one with a `-1` suffix | Restoring the snapshot removed Tailscale's stored node identity, so re-joining registered the VM as a new device with a new 100.x address | Removed the stale offline device in the Tailscale admin console and renamed the active nodes clearly |

## Key commands

```bash
# Ubuntu: status and diagnostics
tailscale status
tailscale ip -4
tailscale ping <peer>
ping -c 4 <peer-hostname>

# Ubuntu: SSH server
sudo apt install openssh-server -y
sudo systemctl enable --now ssh

# Ubuntu: show host key fingerprint for verification
ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
```

```powershell
# Windows: connect over the tailnet
ssh <user>@<ubuntu-hostname>
```

## What I learned

- How a mesh VPN differs from a hub-and-spoke VPN, and how Tailscale uses NAT traversal to form direct connections even when both VMs sit behind VirtualBox NAT.
- The difference between testing the overlay network (`tailscale ping`) and testing normal IP traffic (`ping`), and why one can succeed while the other fails, for example when a host firewall blocks ICMP.
- How MagicDNS gives devices stable names within the tailnet.
- Why SSH host key verification matters, and how to check it properly rather than just typing "yes".
- Reading package manager output to find the actual cause of an install failure instead of retrying blindly.

## Next steps

- Use Tailscale ACLs to restrict which devices can reach SSH.
- Switch SSH to key-based authentication and disable password login.
- Configure one VM as a subnet router to reach non-Tailscale devices on the lab network.

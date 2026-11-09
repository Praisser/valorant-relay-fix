# WireGuard VPN on AWS (EC2)

Set up your own private VPN in about 30 to 45 minutes: a small Ubuntu server on AWS running WireGuard, with your phone or laptop connecting through the WireGuard app. All your traffic goes out through the AWS server, so websites see the server's IP instead of yours.

```
Your phone / laptop  --(encrypted, UDP 51820)-->  AWS EC2 (WireGuard)  -->  Internet
```

## Contents

1. [What you need](#what-you-need)
2. [Step 1: AWS account and region](#step-1-aws-account-and-region)
3. [Step 2: Launch the server](#step-2-launch-the-server)
4. [Step 3: Give it a fixed IP](#step-3-give-it-a-fixed-ip)
5. [Step 4: SSH into the server](#step-4-ssh-into-the-server)
6. [Step 5: Install and configure WireGuard](#step-5-install-and-configure-wireguard)
7. [Step 6: Add a device](#step-6-add-a-device)
8. [Step 7: Test it](#step-7-test-it)
9. [Troubleshooting](#troubleshooting)
10. [Cost and cleanup](#cost-and-cleanup)

## What you need

- An email address, a phone number, and a debit/credit card (AWS runs a small refundable verification charge; Indian cards need international/online transactions enabled)
- A laptop with a terminal: PowerShell on Windows 10/11, or Terminal on Mac/Linux
- The free [WireGuard app](https://www.wireguard.com/install/) on every device you want to connect

Anything in a code block gets copy-pasted as is. `SERVER_IP` means "replace with your own IP".

## Step 1: AWS account and region

1. Go to [aws.amazon.com](https://aws.amazon.com) > **Create an AWS account**. Enter email, password, choose **Personal**, add your card and phone.
2. Choose the **Free plan** and **Basic support**.
3. Pick a region at the top right of the console. This decides which country your VPN traffic comes out of:
   - **Asia Pacific (Mumbai) ap-south-1**: fastest from India
   - **Singapore / US East / Frankfurt** etc.: if you want to appear in that country
4. Recommended: search **Budgets** > **Create budget** > **Use a template** > **Zero spend budget**, add your email. AWS emails you the moment anything starts costing real money.
5. Recommended: account name (top right) > **Security credentials** > **Assign MFA device** and set up an authenticator app.

> Everything you create lives in the region you picked. If something "disappears", check the region dropdown first.

## Step 2: Launch the server

Search **EC2** in the top bar > **Launch instance**:

| Setting | Value |
| --- | --- |
| Name | `wireguard-vpn` |
| OS image | **Ubuntu Server 24.04 LTS**, 64-bit (x86) |
| Instance type | `t3.micro` (or `t2.micro` if t3 isn't marked free tier eligible) |
| Key pair | **Create new key pair** > name `wg-key`, type **ED25519**, format **.pem** |
| Storage | Default (8 GB gp3) |

The key file `wg-key.pem` downloads once. Keep it safe; AWS won't let you download it again.

Under **Network settings** click **Edit**:

- **Auto-assign public IP:** Enable
- **Firewall:** Create security group, name `wireguard-sg`
- Rule 1: **SSH**, port 22, source **My IP**
- **Add security group rule**: **Custom UDP**, port **51820**, source **Anywhere (0.0.0.0/0)**

Click **Launch instance** > **View all instances**. Wait until it says *Running* and *2/2 checks passed*.

> If you forget the UDP 51820 rule, everything will look fine but the VPN will never connect.

## Step 3: Give it a fixed IP

Without this, the public IP changes every time the server restarts and all your devices break.

1. EC2 sidebar > **Network & Security** > **Elastic IPs**
2. **Allocate Elastic IP address** > **Allocate**
3. Select it > **Actions** > **Associate Elastic IP address** > pick `wireguard-vpn` > **Associate**
4. Note the IP. This is your `SERVER_IP`.

## Step 4: SSH into the server

**Mac / Linux** (Terminal):

```bash
mv ~/Downloads/wg-key.pem ~/
chmod 400 ~/wg-key.pem
ssh -i ~/wg-key.pem ubuntu@SERVER_IP
```

**Windows** (PowerShell, not as admin):

```powershell
Move-Item $HOME\Downloads\wg-key.pem $HOME\
icacls $HOME\wg-key.pem /inheritance:r
icacls $HOME\wg-key.pem /grant:r "$($env:USERNAME):(R)"
ssh -i $HOME\wg-key.pem ubuntu@SERVER_IP
```

Type `yes` at the first prompt. Then switch to root (Steps 5 and 6 assume this):

```bash
sudo -i
```

The prompt now ends in `#`. If you log out and come back later, run `sudo -i` again.

> **No terminal?** EC2 console > select the instance > **Connect** > **EC2 Instance Connect** opens one in the browser. If it fails, temporarily set the SSH rule source to *Anywhere*, then set it back to *My IP* after.

## Step 5: Install and configure WireGuard

Update and install:

```bash
apt update && apt upgrade -y
apt install -y wireguard qrencode iptables
```

If a purple screen asks which services to restart, just press Enter.

Generate the server keys, write the config, turn on forwarding, and start it:

```bash
cd /etc/wireguard
umask 077
wg genkey | tee server_private.key | wg pubkey > server_public.key
IFACE=$(ip route list default | awk '{print $5}')
cat > wg0.conf << EOF
[Interface]
Address = 10.8.0.1/24
ListenPort = 51820
PrivateKey = $(cat server_private.key)
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -A FORWARD -o wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o $IFACE -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -D FORWARD -o wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o $IFACE -j MASQUERADE
EOF
echo "net.ipv4.ip_forward=1" > /etc/sysctl.d/99-wireguard.conf
sysctl -p /etc/sysctl.d/99-wireguard.conf
systemctl enable --now wg-quick@wg0
wg show
```

`wg show` should print `interface: wg0` and `listening port: 51820`.

## Step 6: Add a device

Run this once per device. Change `NAME` and give each device its own `IP` (`10.8.0.2`, `10.8.0.3`, `10.8.0.4`...).

```bash
NAME=phone
IP=10.8.0.2
cd /etc/wireguard
umask 077
wg genkey | tee ${NAME}_private.key | wg pubkey > ${NAME}_public.key
cat >> wg0.conf << EOF

[Peer]
# ${NAME}
PublicKey = $(cat ${NAME}_public.key)
AllowedIPs = ${IP}/32
EOF
cat > ${NAME}.conf << EOF
[Interface]
PrivateKey = $(cat ${NAME}_private.key)
Address = ${IP}/32
DNS = 1.1.1.1

[Peer]
PublicKey = $(cat server_public.key)
Endpoint = $(curl -s https://checkip.amazonaws.com):51820
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
EOF
systemctl restart wg-quick@wg0
```

**Phone:** show a QR code in the terminal, then in the WireGuard app tap **+** > **Scan from QR code**.

```bash
qrencode -t ansiutf8 < phone.conf
```

**Laptop:** rerun the block above with `NAME=laptop` and `IP=10.8.0.3`, then print the config:

```bash
cat laptop.conf
```

Copy the output, save it as `laptop.conf` on the laptop, and in the WireGuard app choose **Import tunnel(s) from file**.

## Step 7: Test it

1. Turn the tunnel on in the WireGuard app.
2. Google "what is my ip". It should show your `SERVER_IP`.
3. On the server, `wg show` should list the device with a recent **latest handshake**.

## Troubleshooting

| Problem | Fix |
| --- | --- |
| No handshake at all | UDP 51820 rule missing in the security group, or wrong IP in `Endpoint`. College/office Wi-Fi sometimes blocks UDP, so try mobile data. |
| Connects but no internet | `sysctl net.ipv4.ip_forward` should print `= 1`. Then `systemctl restart wg-quick@wg0`. |
| SSH times out | Your home IP changed. Edit the SSH rule source back to **My IP**. |
| `Permission denied (publickey)` | Username must be `ubuntu`. Check the path to `wg-key.pem`. |
| `UNPROTECTED PRIVATE KEY FILE` | Rerun the `chmod` / `icacls` commands from Step 4. |
| Some sites (banks, Netflix) block you | Normal. Many sites flag AWS datacenter IPs. |
| Check service status | `systemctl status wg-quick@wg0` |

## Cost and cleanup

**Cost** (as of Nov 2026, check the [AWS Free Tier page](https://aws.amazon.com/free/) for current terms):

- AWS accounts created on or after 15 July 2025 get a credit-based **Free plan** (up to $200 in credits, up to 6 months). A `t3.micro` and its public IP draw from those credits. Under the Free plan the account closes when the credits or the 6 months run out, unless you upgrade to the Paid plan.
- Older accounts get the legacy 12-month free tier (750 hours/month of `t2.micro` or `t3.micro`).
- After that, expect roughly $7 to $8/month for a `t3.micro` plus a small hourly charge for the public IPv4 address. Prices vary by region.

**Remove one device:** delete its `[Peer]` block from `/etc/wireguard/wg0.conf`, then:

```bash
systemctl restart wg-quick@wg0
```

**Shut everything down:**

1. EC2 > **Instances** > select `wireguard-vpn` > **Instance state** > **Terminate**
2. EC2 > **Elastic IPs** > select the IP > **Actions** > **Release** (an unattached Elastic IP keeps billing)
3. Optional: delete the `wireguard-sg` security group and the `wg-key` key pair

## Security notes

- Each `.conf` file contains a private key. Anyone who has it can use your VPN, so only send each file to the person whose device it's for, and never commit them to a repo.
- Keep the SSH rule on **My IP**, not *Anywhere*.
- Run `apt update && apt upgrade -y` every now and then to keep the server patched.

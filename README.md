# valorant-relay-fix
WireGuard VPN on AWS: Setup Guide
9 Nov 2026 · @Praisser
Overview
By the end you'll have your own private VPN: a small Ubuntu server on AWS running WireGuard, and your phone or laptop connecting to it with the WireGuard app. All your traffic then goes out through the AWS server, so websites see the server's IP, not yours.
What you need before starting
• An email address and a debit/credit card (AWS verifies it with a small refundable charge; Indian cards need international/online transactions enabled)
• A phone number for verification
• A laptop with a terminal: PowerShell on Windows 10/11, Terminal on Mac or Linux
• The free WireGuard app on each device you want to connect (Play Store, App Store, or wireguard.com/install)
• About 30 to 45 minutes
How to use this guide: do the steps in order. Anything in a grey code box gets copy-pasted into the terminal exactly as written. Text like SERVER_IP means "replace this with your own value".
Step 1: Create the AWS account and pick a region
The region you pick decides which country your VPN traffic comes out of, so choose it on purpose.
1. Go to aws.amazon.com and click Create an AWS account. Enter your email, an account name, and verify the email code.
2. Set a strong root password. Choose Personal account type and fill in your address.
3. Add your card, then verify your phone number by SMS or call.
4. When asked to pick a plan, choose the Free plan. When asked for a support plan, choose Basic (free).
5. Sign in to the AWS Console as the root user.
6. Recommended: click your account name (top right) > Security credentials > Assign MFA device, and set up an authenticator app. Losing an AWS account to a leaked password gets expensive fast.
7. Pick your region from the dropdown at the top right, next to your account name:
    ◦ Asia Pacific (Mumbai) ap-south-1: fastest from India, your IP stays Indian
    ◦ Asia Pacific (Singapore), US East (N. Virginia), Europe (Frankfurt) etc.: if you want to appear to be in that country
8. Recommended: search Budgets in the top search bar > Create budget > Use a template > Zero spend budget, and enter your email. AWS will email you the moment anything starts costing real money.
Everything you create from here lives in the region you picked. If something you made "disappears" later, check the region dropdown first.
Step 2: Launch the server (EC2 instance)
You're renting one small Linux machine. Only two settings really matter here: the instance size and the firewall rules.
1. In the top search bar type EC2 and open it. Click Launch instance.
2. Name: wireguard-vpn
3. Application and OS Images: pick Ubuntu, then Ubuntu Server 24.04 LTS, architecture 64-bit (x86). It should say Free tier eligible.
4. Instance type: t3.micro (or t2.micro if t3 isn't marked free tier eligible in your region). Don't pick anything bigger; a VPN for a few devices needs almost nothing.
5. Key pair: click Create new key pair.
    ◦ Name: wg-key
    ◦ Type: ED25519
    ◦ Format: .pem
    ◦ Click Create key pair. A file wg-key.pem downloads. Keep it safe; AWS won't let you download it again, and without it you can't log in.
6. Network settings: click Edit.
    ◦ Auto-assign public IP: Enable
    ◦ Firewall: Create security group, name it wireguard-sg
    ◦ Rule 1 (already there): type SSH, port 22, source type My IP
    ◦ Click Add security group rule for rule 2: type Custom UDP, port range 51820, source type Anywhere (0.0.0.0/0)
7. Configure storage: leave the default (8 GB gp3).
8. Click Launch instance, then View all instances.
9. Wait until Instance state says Running and Status check says 2/2 checks passed (a couple of minutes; hit refresh).
Port 51820/UDP is what WireGuard listens on. If you forget that rule, everything else will look fine but the VPN will never connect.
Step 3: Give the server a fixed IP (Elastic IP)
Without this, the server's public IP changes every time it's stopped and started, and every device you set up stops working.
1. In the EC2 left sidebar, under Network & Security, click Elastic IPs.
2. Click Allocate Elastic IP address, leave the defaults, click Allocate.
3. With the new IP selected, click Actions > Associate Elastic IP address.
4. Instance: pick wireguard-vpn. Click Associate.
5. Write this IP down. The rest of the guide calls it SERVER_IP.
An Elastic IP that isn't attached to a running server still costs money. If you ever delete the server, release the IP too (see the last section).
Step 4: Log in to the server (SSH)
You'll type commands on the server from your own laptop's terminal. Move wg-key.pem from Downloads into your home folder first so the paths below work.
Mac / Linux (Terminal):
mv ~/Downloads/wg-key.pem ~/
chmod 400 ~/wg-key.pem
ssh -i ~/wg-key.pem ubuntu@SERVER_IP
Windows (PowerShell, not as admin):
Move-Item $HOME\Downloads\wg-key.pem $HOME\
icacls $HOME\wg-key.pem /inheritance:r
icacls $HOME\wg-key.pem /grant:r "$($env:USERNAME):(R)"
ssh -i $HOME\wg-key.pem ubuntu@SERVER_IP
The first time, it asks Are you sure you want to continue connecting? Type yes and press Enter. You're in when the prompt looks like ubuntu@ip-172-31-...:~$.
Now switch to the admin (root) user. Every command in Steps 5 and 6 assumes you did this:
sudo -i
The prompt changes to end in #. If you log out and come back later, run sudo -i again.
No terminal handy? In the EC2 console, select the instance > Connect > EC2 Instance Connect > Connect opens a terminal in the browser. It only works if the SSH rule's source allows AWS's connect service, so if it fails, temporarily change the SSH rule source to Anywhere and set it back to My IP when you're done.
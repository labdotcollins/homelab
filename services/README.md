# Services

Currently, I’m hosting the following services:

- Immich
- Nextcloud
- AdGuard
- Caddy
- Vaultwarden
- Wazuh

In the future, I’d like to expand into Home Assistant, though right now my house doesn’t have enough smart devices to fully justify it. I’m also interested in running the full Jellyfin media suite; Jellyfin, Gluetun, Sonarr, Radarr, Prowlarr, and qBittorrent, but that setup would require a paid VPN service, which isn’t an option for me at the moment.

# Proxmox

I ended up choosing Proxmox over TrueNAS Scale because, after testing both, Proxmox felt far more versatile. At one point, I even tried running TrueNAS Scale inside Proxmox, but that quickly proved impractical—I had created a ZFS storage pool in TrueNAS and then passed it through to Proxmox, which felt like a fragile setup just waiting to cause problems. Since I didn’t really see the benefit of running containers or VMs inside TrueNAS on top of Proxmox, I eventually scrapped it. Instead, I set up a Samba share in a container, mounting my ZFS pool directly there.

One thing I learned too late was about RAIDZ1 vs RAIDZ2. It turns out RAIDZ1 is generally not recommended for drives larger than 2TB, and I had four 3TB drives in my pool. This is because in the event of drive failure, the resilvering process takes much longer on a larger drive. During that time, the chance of encountering an issue increases significantly, which would then cause the rebuild to fail and result in data loss. RAIDZ2 provides an extra layer of redundancy by allowing two drives to fail before the pool is at risk, which makes it far safer for larger capacity disks.

LXC vs VM

Setup commands, GPU Passthrough, Backups, and Cron

# Tailscale + Adguard + Caddy

This part of my setup is really the heart of the project, and the piece I’m most proud of. Since I couldn’t find much documentation for this exact configuration, it was a fun challenge to figure things out on my own.

I had some prior experience with Tailscale, which I use as my main way of connecting to devices when I’m outside my home network. I also came across NetBird, an open-source alternative with many of the same features, but decided to save that experiment for later. The main reason I rely on Tailscale is that my ISP uses CGNAT (Carrier-Grade NAT), meaning I don’t get a static IP address. This rules out traditional port forwarding and makes directly exposing services to the internet impractical. Along the way, I did learn about alternatives like Cloudflare DDNS combined with Nginx Proxy Manager, but I thought up something different: I wanted remote access to my services without exposing them publicly.

At first, I experimented with Tailscale’s Split DNS and Serve features, but I ran into the limitation that all domains had to end with .ts.net. I eventually realized this was unnecessary because Tailscale lets you advertise local subnets to your tailnet. In practice, that means as long as my device is connected to Tailscale, it can directly reach any of the services running on my host’s VLAN that I'm advertising, which allows me to use my own local domain instead of being locked into .ts.net.

I chose AdGuard Home over Pi-hole mainly because I had used it before, and I found it more flexible. Caddy, on the other hand, was a different story. I had originally planned to issue internal TLS certificates and run everything behind Nginx Proxy Manager. While that setup might have worked in the end, I ran into a lot of complications with getting certificates and reverse proxy rules working smoothly, especially without being publicy facing. During troubleshooting, I came across Caddy by chance, and immediately switched gears. In the end, I generated my own certificate authorities rather than relying on internal certs, but even then, Caddy made the entire process far smoother than my experience with Nginx Proxy Manager.

# Nextcloud

Immich takes care of my photo backups, but I also wanted to deploy Nextcloud, not just for file storage, but for its potential to enable document collaboration. There’s still plenty for me to experiment with and learn here.

# Vaultwarden

This has been a long time coming. I’d been using KeePassXC for years, but never liked being tied to a local application. Vaultwarden turned out to be the perfect replacement solution with everything I needed from KeePassXC. Now, generating and saving passwords is handled right from my browser through the Bitwarden plugin, without the hassle of managing a separate desktop app. Vaultwarden requires HTTPS, so you definitely want to hold off on deploying this until you've set that up for your network.

# Wazuh

Wazuh is a SIEM. I’ve deployed agents across all my hosts and computers, though I’m still in the process of learning the platform and exploring everything it can do.

# Portainer

I initially deployed Portainer, but I ultimately found it unnecessary since I primarily manage my containers through Docker Compose. For my workflow, Compose feels more straightforward and gives me all the control I need without the extra layer.

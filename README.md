# Homelab

This is the documentation for my homelab project. Here, I’ll explain the reasoning behind my setup decisions, the applications I chose to deploy, and the steps I took to configure them. I’m still learning as I go, so this serves not only as a reference in case something breaks, but also as a resource for anyone else stubborn enough to piece together a functional homelab from scratch.

# Index

- Services
	- Proxmox
		- Immich
		- Tailscale + Adguard + Caddy
		- Nextcloud
		- Vaultwarden
		- Wazuh

The biggest thing I'm proud of with this server is, despite being a homelab, I was actually able to set it up to be remotely accessible from anywhere without being public facing
## Hardware

- Mini PC | Proxmox
	- NAB 6 Lite
		- Intel i5-12600H
			- 12 Cores (4 Performance-cores, 8 Efficient-cores) / 16 Threads
		- 2x16 DDR4-3200MHz SODIMM
		- 512GB M.2 2280 PCIe 4.0 

- DAS
	- SABRENT 4-Bay USB 3.2 Gen 2 SATA DAS
		- x4 WD Red Plus HDD

Before starting, I had a few clear goals: I wanted a low-power machine capable of running either TrueNAS Scale or Proxmox, with enough performance to handle multiple services with GPU hardware acceleration, possibly for multiple users as I wanted to keep my family in mind. Building a $1,000 custom low-power microATX system wasn’t realistic for my budget, so I turned to Mini PCs as an alternative. Many people recommended models with Intel’s N100 chip, but I opted for a refurbished Mini PC with an i5 that I found on eBay for a pretty good price. This gave me more confidence in overhead performance compared to the N100, which, while efficient, was less powerful.

I also considered a few different storage configurations, including adapting the onboard M.2 slot on the NAB 6 Lite into SATA to support four drives. In the end, I went with a simpler, and more expensive, DAS solution. My Mini PC connects to the DAS through a USB 3.1 Gen 2 (10 Gbit) port, which provides more flexibility and performance for my setup.

For storage, I went with the cheapest CMR NAS-grade drives I could find, ultimately landing on WD Red Plus, which have been running without issue. From my research, SMR drives work fine for archival or write-once/read-mostly use, but for a homelab or RAID, especially ZFS configurations, CMR is the more reliable choice.

In the picture, you’ll notice the NUC, which I haven’t set up yet since it was only recently gifted to me by my former employer. I also happen to have a NUC rack mount that I haven’t installed, but my plan is to eventually mount both Mini PCs. The Minisforum unit, however, will be more of a challenge since it doesn’t fit the rack dimensions as neatly.

---
title: Homelab cleanup time!
draft: true
date: 2026-04-05
---
With exams looming in a couple months and precisely zero revision done, now is clearly the perfect time to go down the rabbithole of clearing up the mess that is my current homelab configuration.
As with all good rabbitholes, it started with the simple mistake of having a Thought™ - "how can I centralise authentication instead of having a mess in my password manager?".
And so into the depths of random blog posts I went, gradually growing the scope of what I wanted to have acheived by the end of this...

I'm trying something new with this particular post, and instead of writing it 5 months after the fact when I've half forgotten what I did, I'm actually writing it as I make the changes!!
Let's lay out the current state of my homelab (all hosted on a single Poweredge R720 running Proxmox):
- TrueNAS Scale VM - used for long-term high capacity storage (laptop backups, photo storage, music and film storage, generic network storage)
- macOS VM - spun up manually when I want it to have a faster build server instead of compiling directly on my laptop
- Docker VM:
	- `reprepro` - hosts an apt repo for as yet never mentioned side project
	- `cloudflared` - sets up a half-broken cloudflare zero-trust tunnel instead of relying on port forwarding
	- `ddns` - pushes my public IP to cloudflare since aforementioned zero-trust tunnel is half-broken
	- `immich_*` - various containers to run photo storage
	- `jellyfish` - because apparently I can't spell Jellyfin and never fixed it
	- `portainer` - for a nice frontend to Docker
	- `traefik` - reverse proxy to route requests to the right container
	- `*-hosting-*` - various containers for static and dynamic site hosting for friends and family

# The plan

So let's try to put together a vague plan of what would be nice to achieve, and see how much of this I can stumble through in a random order before getting bored.
1. Reconfigure networking to have proper static IPs allocated and internal domain names instead of having the current mess
2. Spin up a central, backed-up postgres server and eventually migrate other services over to it
3. Setup LLDAP and some OICD system to centralise auth for all services and point all services that require auth (which is probably all of them?) at it
4. Clean up the brokenness of all the publicly exposed web stuff that half uses tunnels and half uses port forwarding
5. Non-manual setup for spinning up hosting instances would be nice
6. As would moving to Jenkins or something for build servers instead of manually cloning and compiling over SSH
7. ~~And hey why not switch to k8s while we're at it too~~
8. ~~Wait ipv6 still doesn't seem to work either~~

# Networking

I'll make a start by reducing the current DHCP pool to only allocate  `.70` to `.250` (that should surely be enough for whatever joins the network), and then planning out some static IPs for all the infrastructure.

| IP                  | Purpose                            |
| ------------------- | ---------------------------------- |
| 192.168.86.1        | Gateway                            |
| 192.168.86.20       | Primary (and only) physical server |
| 192.168.86.21       | ILO for primary server             |
| 192.168.86.30       | Docker VM                          |
| 192.168.86.31       | Postgres server                    |
| 192.168.86.33       | TrueNAS VM                         |
| 192.168.86.34       | macOS build runner                 |
| 192.168.86.35       | Linux build runner                 |
| 192.168.86.60       | My laptop                          |
| 192.168.86.70 - 254 | DHCP pool                          |

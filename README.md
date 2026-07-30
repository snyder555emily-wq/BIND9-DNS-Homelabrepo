# BIND9-DNS-Homelab

I built and deployed a self-hosted, authoritative DNS server using BIND9 in Docker, then broke it (repeatedly, across two different networks) and fixed it. This repo documents the build, the config, and every failure mode I ran into along the way.

## Tools Used

- BIND9 — DNS server software
- Docker — containerization
- zsh / macOS Terminal — CLI environment
- nslookup / dig — DNS query testing and validation
- Git/GitHub — version control

## What I Did

- Deployed a containerized DNS server from scratch and configured it to serve custom hostnames on a local network
- Wrote and edited zone files (`home.lan.zone`) to map internal hostnames to IP addresses
- Configured access control lists (ACLs) in `named.conf` to restrict which network segments could query the server, preventing it from acting as an open recursive resolver
- Rebuilt the entire network configuration after switching physical networks, correctly re-scoping ACLs and zone records to a new subnet instead of leaving stale, insecure config in place
- Diagnosed a fatal BIND startup failure (`undefined ACL 'internal'`) by reading raw daemon logs and tracing it to a naming mismatch between where an ACL was defined and where it was referenced
- Identified and resolved a Docker Desktop networking quirk on macOS, where container-bound queries appear to originate from Docker's internal NAT gateway (`192.168.65.0/24`) rather than the host's actual LAN IP, and adjusted the ACL accordingly instead of just disabling security controls 
- Distinguished between different DNS failure states (`REFUSED` vs `NXDOMAIN` vs connection timeout) to correctly diagnose root cause instead of guessing

## What I Learned

- `REFUSED` means an access control problem, not a DNS problem. I kept checking zone files and hostnames first when the real issue was the ACL blocking the query before it even got there.
- Docker Desktop on Mac routes container traffic through an internal NAT gateway, so the source IP BIND sees isn't the IP the client is actually sending from. The ACLS need to match. 
- `docker compose logs` helped me figure out all of the issues I was having. I stopped guessing and fixed the problem directly instead of restarting the container.
- I could've fixed the denied queries by setting the ACL to `any`, but that turns the server into an open resolver, which is a DNS abuse vector.

## Setup

```
git clone https://github.com/snyder555emily-wq/BIND9-DNS-Homelabrepo.git
cd BIND9-DNS-Homelabrepo
```

Edit `named.conf` to set your trusted subnet(s):

```
acl internal {
    10.0.0.0/24;
    192.168.65.0/24;   # needed if running Docker Desktop on macOS
};
```

Edit `home.lan.zone` to add your hosts:

```
ns1     IN      A       10.0.0.21
```

Start it:

```
docker compose up -d
```

Verify:

```
nslookup ns1.home.lan 10.0.0.21
```

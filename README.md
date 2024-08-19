# A template for modular, secure databases via a compose stack
This repository provides Docker Compose configurations for setting up secure, modular databases using MariaDB, MongoDB, Redis, and PostgreSQL with SSL/TLS, managed by Adminer.
Feel free to make a PR or open an issue if any information has gotten out of data or is not correct. As of yet, this has not been tested in production.

## Prerequisites

- Docker and Docker Compose installed on your system
- (optional - only if you are going to rely on the guide below for generating certificates) - A domain added to Cloudflare

The `docker-compose.yml` includes services configured as follows:

- **MariaDB:** Runs on port 3306, with SSL certificates.
- **MongoDB:** Runs on port 27017, prefers TLS but allows non-TLS.
- **Redis:** TLS on port 6379 and non-TLS on 6380.
- **PostgreSQL:** Runs on port 5432, with enforced SSL.
- **Adminer:** Web database management accessible on port 8530.

## Firewall Configuration

Allow Adminer access to all database addresses (examples are not persistent across reboots, look up how to make iptables or nftables persistent on your specific operating system):

### iptables

```bash
iptables -A INPUT -d 172.18.0.1 -p tcp -m multiport --sports 8530 --dports 3306,27017,6379,5432 -j ACCEPT
```

### nftables

```bash
nft add rule ip filter input ip daddr 172.18.0.1 tcp sport 8530 tcp dport {3306, 27017, 6379, 5432} accept
```

## Generating a Cloudflare API Token
- To generate a token, log into Cloudflare, go to your profile at the top right, and select API Tokens
- Next, click Create Token, and select the Edit zone DNS template. Under Zone Resources, select the domain you want to generate a certificate for.
- Under Client IP Address Filtering, type in your machine's public IP address and hit enter. This will prevent the token from being used on any machine other than your server.
- Hit "Continue to Summary", and save your token somewhere you will remember it, e.g. a password manager or secure note.

## SSL Certificate Generation

Using `acme.sh` with Cloudflare for DNS validation (this example is for mariadb.example.com - simply repeat step 4 for each subdomain you've created an A record for):

1. Set an A record for `mariadb.example.com` pointing to `172.18.0.1`.
2. Install `acme.sh`:

```bash
curl https://get.acme.sh | sh
```
- If it complains about not having a package, install that package and run it again until it no longer complains.

3. Configure Cloudflare API token:

```bash
export CF_Email="your_email@example.com"
export CF_Token="your_cloudflare_token"
```

4. Issue the certificate:

```bash
acme.sh --issue --dns dns_cf -d "mariadb.example.com" --server letsencrypt \
--key-file /etc/letsencrypt/live/mariadb.example.com/privkey.pem \
--fullchain-file /etc/letsencrypt/live/mariadb.example.com/fullchain.pem
```

- If anything went wrong (you messed up one of the export or forgot to create the parent directories ahead of time), simply fix the export or create the directories, then run the command again with --force at the end of it.

## Auto-Renewal

Ensure `acme.sh` has installed a cron job for auto-renewal by checking your crontab:

```bash
crontab -e
```

- If it does not have the line, you likely missed the step above when installing acme.sh:
> If it complains about not having a package, install that package and run it again until it no longer complains.
  - If that is the case, simply install the missing package(s), run the acme.sh installer again, and repeat step 4 with --force appended to the command.

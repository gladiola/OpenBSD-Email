# OpenBSD Email Server

This repository documents how to run an email server on OpenBSD using built-in components:

- `smtpd(8)` (OpenSMTPD) for SMTP receive, transmission (submission), and relay
- `mail.local(8)` for local mailbox delivery (`mbox`)
- `acme-client(1)` + `openssl(1)` for TLS certificates
- `pf(4)` for network exposure control

> Notes:
> - OpenBSD ships OpenSMTPD in base, so no extra SMTP package is required.
> - Running a full Internet-facing mail server also needs DNS records (A/AAAA, MX, PTR, SPF, DKIM, DMARC). See [§ 7 DNS records](#7-dns-records) below.

## 1) Installation / base setup

On a fresh OpenBSD host, update base and set host identity:

```sh
# syspatch
# echo "mail.example.net" > /etc/myname
# hostname mail.example.net
```

OpenSMTPD is already present. Enable it at boot:

```sh
# rcctl enable smtpd
```

## 2) TLS certificate setup

### Option A (public CA via ACME, recommended for Internet-facing hosts)

Configure `/etc/acme-client.conf` for your domain, then request a certificate:

```sh
# acme-client -v mail.example.net
```

Expose the cert and key to OpenSMTPD via a PKI block in `/etc/mail/smtpd.conf`.

### Option B (private CA / lab)

Create a certificate/key pair with `openssl` and reference it in the same PKI block.

## 3) `smtpd` configuration

Edit `/etc/mail/smtpd.conf` (example):

```conf
pki mail.example.net cert "/etc/ssl/mail.example.net.fullchain.pem"
pki mail.example.net key "/etc/ssl/private/mail.example.net.key"

# Optional CA used to verify client certificates when mTLS is enabled.
ca local_ca cert "/etc/ssl/local-ca.pem"

action "local_mbox" mbox alias <aliases>
action "outbound" relay

# Local queue injection
match from local for local action "local_mbox"
match from local for any action "outbound"

# Public SMTP listener with STARTTLS available
listen on egress tls pki mail.example.net

# Submission listener (587): require TLS + AUTH for users
listen on egress port submission tls-require pki mail.example.net auth
match from auth for any action "outbound"
```

Validate and reload:

```sh
# smtpd -n
# rcctl restart smtpd
```

## 4) Operation

Common runtime commands:

```sh
# rcctl check smtpd
# smtpctl show status
# smtpctl show queue
# mailq
```

Send a local test message:

```sh
$ echo "test body" | mail -s "openbsd mail test" localuser
```

Inspect logs:

```sh
# tail -f /var/log/maillog
```

## 5) TLS hardening checklist

- Use valid certificates in `pki` blocks.
- Use `tls-require` (not only `tls`) on user submission ports.
- Keep private keys readable only by root (`/etc/ssl/private`).
- Restrict exposed ports with `pf` (typically 25, 465, 587 as needed).
- Prefer authenticated submission for users; do not run an open relay.

## 6) mTLS with OpenBSD built-ins (laptop submission)

The steps below implement a strict mTLS path from a laptop to your OpenSMTPD submission listener using only built-in OpenBSD tools (`openssl`, `smtpd`, `pf`).

### 6.1 Create a private CA and issue server/client certs

Create a CA for client authentication and keep its private key offline if possible:

```sh
# install -d -m 700 /etc/ssl/my-mail-ca
# openssl genrsa -out /etc/ssl/my-mail-ca/ca.key 4096
# openssl req -x509 -new -nodes -sha256 -days 3650 \
    -key /etc/ssl/my-mail-ca/ca.key \
    -out /etc/ssl/my-mail-ca/ca.crt \
    -subj "/CN=My Mail Client CA"
```

Issue a server certificate for your mail host:

```sh
# openssl genrsa -out /etc/ssl/private/mail.example.net.key 4096
# openssl req -new -key /etc/ssl/private/mail.example.net.key \
    -out /etc/ssl/mail.example.net.csr \
    -subj "/CN=mail.example.net"
# printf "subjectAltName=DNS:mail.example.net\nextendedKeyUsage=serverAuth\n" \
    > /tmp/server-ext.cnf
# openssl x509 -req -in /etc/ssl/mail.example.net.csr \
    -CA /etc/ssl/my-mail-ca/ca.crt -CAkey /etc/ssl/my-mail-ca/ca.key \
    -CAcreateserial -out /etc/ssl/mail.example.net.crt \
    -days 825 -sha256 -extfile /tmp/server-ext.cnf
# cat /etc/ssl/mail.example.net.crt /etc/ssl/my-mail-ca/ca.crt \
    > /etc/ssl/mail.example.net.fullchain.pem
```

Issue a client certificate for the laptop (clientAuth EKU):

```sh
# openssl genrsa -out /etc/ssl/my-mail-ca/laptop01.key 4096
# openssl req -new -key /etc/ssl/my-mail-ca/laptop01.key \
    -out /etc/ssl/my-mail-ca/laptop01.csr \
    -subj "/CN=laptop01"
# printf "extendedKeyUsage=clientAuth\n" > /tmp/client-ext.cnf
# openssl x509 -req -in /etc/ssl/my-mail-ca/laptop01.csr \
    -CA /etc/ssl/my-mail-ca/ca.crt -CAkey /etc/ssl/my-mail-ca/ca.key \
    -CAcreateserial -out /etc/ssl/my-mail-ca/laptop01.crt \
    -days 825 -sha256 -extfile /tmp/client-ext.cnf
```

### 6.2 Require mTLS in OpenSMTPD submission listener

Example `/etc/mail/smtpd.conf` snippet:

```conf
pki mail.example.net cert "/etc/ssl/mail.example.net.fullchain.pem"
pki mail.example.net key "/etc/ssl/private/mail.example.net.key"
ca mtls_clients cert "/etc/ssl/my-mail-ca/ca.crt"

table <mtls_sources> { 198.51.100.44, 10.8.0.0/24 }

action "local_mbox" mbox alias <aliases>
action "outbound" relay

match from local for local action "local_mbox"
match from local for any action "outbound"

# Public inbound SMTP (no client cert required)
listen on egress tls pki mail.example.net

# Submission: TLS required + valid client cert from mtls_clients CA
listen on egress port submission tls-require pki mail.example.net ca mtls_clients verify

# Keep relay tight to expected source(s)
match from src <mtls_sources> for any action "outbound"
```

Validate and reload:

```sh
# smtpd -n
# rcctl restart smtpd
```

### 6.3 Restrict network exposure with `pf`

Only allow submission from expected laptop/VPN sources:

```pf
table <mtls_submit_clients> { 198.51.100.44, 10.8.0.0/24 }

pass in on egress proto tcp from <mtls_submit_clients> to (egress) port 587
block in on egress proto tcp to (egress) port 587
```

Load and verify:

```sh
# pfctl -nf /etc/pf.conf
# rcctl reload pf
```

### 6.4 Install laptop credentials and trust

Copy to laptop:
- `laptop01.crt` (client cert)
- `laptop01.key` (client key, mode `0600`)
- `ca.crt` (for verifying server cert chain)

On OpenBSD laptop:

```sh
# install -d -m 700 /etc/ssl/private
# install -m 600 laptop01.key /etc/ssl/private/laptop01.key
# install -m 644 laptop01.crt /etc/ssl/laptop01.crt
# install -m 644 ca.crt /etc/ssl/my-mail-ca.crt
```

### 6.5 Send an email from laptop with OpenSSL STARTTLS + mTLS

Use OpenSSL interactive SMTP session:

```sh
$ openssl s_client -starttls smtp -crlf -quiet \
    -connect mail.example.net:587 \
    -cert /etc/ssl/laptop01.crt \
    -key /etc/ssl/private/laptop01.key \
    -CAfile /etc/ssl/my-mail-ca.crt \
    -verify_return_error
EHLO laptop01.example.net
MAIL FROM:<user@example.net>
RCPT TO:<dest@example.org>
DATA
Subject: mTLS test

hello from laptop over mTLS
.
QUIT
```

### 6.6 Validate end-to-end

On laptop:
- Ensure TLS verification succeeds (no certificate verify error from `openssl s_client`).

On server:

```sh
# tail -f /var/log/maillog
# smtpctl show queue
# smtpctl show status
```

Confirm logs show successful client certificate verification and the message is accepted/queued/delivered.

### 6.7 Operational hardening

- Issue one client cert per device/user; do not share keys.
- Revoke and reissue certs for lost/retired devices.
- Rotate CA/certs on a defined schedule.
- Keep CA private key offline or heavily restricted.
- Alert on repeated mTLS verification failures and unknown source IPs.
- Combine mTLS with `pf` source restrictions for defense in depth.

## 7) DNS records

Every Internet-facing mail server requires several DNS records before other hosts will reliably accept its mail. The examples below assume the zone is `example.com` and the mail host is `mail.example.com` at `203.0.113.3` / `2001:db8::3`.

### Zone file entries

Add these records to your authoritative zone file (e.g. `/var/nsd/zones/master/example.com` if you are using [OpenBSD-DNSSEC](https://github.com/gladiola/OpenBSD-DNSSEC)):

```dns
; ── Mail host address records ─────────────────────────────────────────────
mail        IN  A       203.0.113.3
mail        IN  AAAA    2001:db8::3

; ── MX — where to deliver mail for example.com ───────────────────────────
@           IN  MX  10  mail.example.com.

; ── SPF — only the MX host may send for example.com ─────────────────────
@           IN  TXT     "v=spf1 mx -all"

; ── DKIM — public key for the "mail" selector ────────────────────────────
mail._domainkey  IN  TXT  "v=DKIM1; k=rsa; p=<base64-public-key>"

; ── DMARC — quarantine failures, send aggregate reports to postmaster ────
_dmarc      IN  TXT     "v=DMARC1; p=quarantine; rua=mailto:postmaster@example.com; ruf=mailto:postmaster@example.com; adkim=s; aspf=s"
```

### PTR record (reverse DNS)

The PTR record is not in your own zone file — it must be set at your hosting provider or ISP for the IP address block they assigned you:

```
3.113.0.203.in-addr.arpa.  IN  PTR  mail.example.com.
```

Many receiving MTAs perform forward-confirmed reverse DNS (FCrDNS): they look up the PTR for the connecting IP, then verify that the name resolves back to the same IP. A missing or mismatched PTR will cause rejections at major providers.

### Record summary

| Record | Purpose |
|--------|---------|
| `A` / `AAAA` | Forward DNS for the mail host; required for FCrDNS |
| `MX` | Tells other MTAs where to deliver mail for the domain |
| `PTR` | Reverse DNS; must resolve to `mail.example.com` |
| `SPF` (TXT) | Authorizes the MX host to send; `-all` hard-fails all other sources |
| `DKIM` (TXT) | Public half of the signing key; authenticates message bodies |
| `DMARC` (TXT) | Policy for receivers when SPF/DKIM fail; start with `p=quarantine`, move to `p=reject` once delivery is confirmed healthy |

### DKIM signing with OpenSMTPD

OpenSMTPD does not sign DKIM natively. Install the filter package and generate a key pair:

```sh
# pkg_add opensmtpd-filter-dkimsign
# openssl genrsa -out /etc/mail/dkim/mail.example.com.key 2048
# openssl rsa -in /etc/mail/dkim/mail.example.com.key \
        -pubout -out /etc/mail/dkim/mail.example.com.pub
# chmod 640 /etc/mail/dkim/mail.example.com.key
# chown root:_smtpd /etc/mail/dkim/mail.example.com.key
```

Add the filter to `/etc/mail/smtpd.conf`:

```conf
filter "dkimsign" proc-exec "filter-dkimsign -d example.com -s mail \
    -k /etc/mail/dkim/mail.example.com.key"

listen on egress tls pki mail.example.com filter "dkimsign"
listen on egress port submission tls-require pki mail.example.com auth filter "dkimsign"
```

Extract the public key to paste into the `mail._domainkey` TXT record:

```sh
# grep -v '^-' /etc/mail/dkim/mail.example.com.pub | tr -d '\n'
```

Wrap that value in `"v=DKIM1; k=rsa; p=..."` in your zone file, re-sign the zone (if using DNSSEC), and reload NSD.

### DNSSEC interaction

If you are signing your zone with DNSSEC (see [OpenBSD-DNSSEC](https://github.com/gladiola/OpenBSD-DNSSEC)), `ldns-signzone` automatically adds RRSIG records for every RRset — including the TXT records for SPF, DKIM, and DMARC — at no additional configuration cost. This prevents an attacker from tampering with your mail policy records in transit.

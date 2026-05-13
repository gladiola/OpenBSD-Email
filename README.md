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

## 6) mTLS availability

OpenSMTPD supports mutual TLS on listeners:

- `listen ... smtps verify`
- `listen ... tls-require verify`

With `verify`, clients must present a valid certificate. Pair this with a `ca` definition (for example `ca local_ca cert "/etc/ssl/local-ca.pem"`) and reference that CA on the listener (`listen ... ca local_ca ...`). This is useful for controlled MTA-to-MTA or device-to-MTA environments.

For general end-user submission, certificate-based mTLS is usually combined with or replaced by SMTP AUTH, because distributing client certs to all users is operationally heavier.

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

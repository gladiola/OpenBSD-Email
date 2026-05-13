# OpenBSD Email Server

This repository documents how to run an email server on OpenBSD using built-in components:

- `smtpd(8)` (OpenSMTPD) for SMTP receive/relay
- `mail.local(8)` for local mailbox delivery (`mbox`)
- `acme-client(1)` + `openssl(1)` for TLS certificates
- `pf(4)` for network exposure control

> Notes:
> - OpenBSD ships OpenSMTPD in base, so no extra SMTP package is required.
> - Running a full Internet-facing mail server also needs DNS records (A/AAAA, MX, PTR, SPF, DKIM, DMARC).

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

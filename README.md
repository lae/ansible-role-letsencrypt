# Ansible role to obtain Let's Encrypt SSL certificates

This role is meant to request SSL certificates from Let's Encrypt,
using the HTTP or the DNS challenge for their ACME API.

Features:
* Installs and configures certbot and the DNS challenge helper script
* Supports both the HTTP and the DNS challenge
* The DNS challenge uses a dedicated zone for AMCE challenge tokens
  only, lowering the security risks of dynamic updates. The concept
  is explained [here](https://www.crc.id.au/using-centralised-management-with-lets-encrypt/)
* Restart of services at certificate renewal using post-hooks
* Permission control to certificates using a dedicated system group

It does the following:

* When invoked with unset/empty variable 'letsencrypt_certs':
  * Installs certbot, registers an account at Let's Encrypt
  * Installs required files/keys for the DNS challenge
  * Creates the system group 'letsencrypt'
* When invoked with filled variable 'letsencrypt_certs':
  * Requests a SSL certificate via the Let's Encrypt ACME API, either
    using the HTTP challenge or using the DNS challenge
  * Optionall sets the post-hook for certificate renewals to restart
    required services afterwards
  * Optionally adds system users to the 'letsencrypt' system group to grant
    them read access to the SSL certificates and their private keys

## Expected structure of variable `letsencrypt_certs`

The variable `letsencrypt_certs` is expected to be a list of
dictionaries:

```
letsencrypt_certs:
  - domains:
      - sub.example.org
      - sub2.example.org
    challenge: http
    services:
      - apache2
      - exim4
    users:
      - Debian-exim
  - domains:
      - sub3.example.org
    challenge: dns
    services:
      - dovecot
```

The dictionaries have the following keys:

* domains: list of domains for the certificate [required]
* challenge: 'http' or 'dns' [required]
* services: list of services to be restarted in the post-hook [optional]
* users: list of users to be added to system group 'letsencrypt' [optional]

## General Preliminaries

The role takes care of installing certbot and requesting SSL certificates
using either the HTTP or the DNS challenge. It doesn't install or configure
the required infrastructure (i.e. the Apache webserver or a DNS server).

The role is tested with Ansible 2.2 only. No guaranties that it runs with
earlier versions of Ansible.

## The HTTP challenge

Requirements:
* Apache needs to be installed (and configured) on the system
* The domain name(s) of the requested certificate should point
  to the system

## The DNS challenge

Requirements:
* A DNS server with a dedicated zone, used for the ACME DNS challenge only.
  This zone has to allow dynamic DNS updates (NSUPDATE) for TXT records
  (see below).
* CNAME records for `_acme-challenge.sub.example.org` for all domain
  names(s) of the requested certificate have to point to
  `sub.example.org._le.example.org` (inside the dedicated zone for the
  ACME DNS challenge).
* The content of the DNS update key and private DNS update keys need to be
  available in the Ansible vars `letsencrypt_ddns_key` and
  `letsencrypt_ddns_privkey` (preferably inside a vault).

This role installs a helper script for the DNS challenge to
`/usr/local/bin/certbot-dns-hook.sh`. This script will add the validation
token to the TXT record at `sub.example.org._le.example.org` during the DNS
challenge and remove it afterwards.

### Configuring bind9 for the DNS challenge
*(Another option would be to use the [acme-dns server](https://github.com/joohoi/acme-dns) for this)*

Generate a key for dynamic updates:

```
cd /etc/bind/keys
dnssec-keygen -a HMAC-SHA512 -b 512 -n USER _le.example.org_ddns_update
chown -R bind:bind /etc/bind/keys
```

Add the key to your bind config (e.g. at `/etc/bind/named.conf.options`):

```
key "_le.example.org_ddns_update" {
        algorithm hmac-sha512;
        secret "...";
};
```

Create the zone for dynamic updates:

```
$ORIGIN .
$TTL 86400	; 1 day
_le.example.org		IN SOA	ns1.example.org. postmaster.example.org. (
				2017061501 ; serial
				86400      ; refresh (1 day)
				3600       ; retry (1 hour)
				2419200    ; expire (4 weeks)
				86400      ; minimum (1 day)
				)
			NS	ns1.example.org.
			NS	ns2.example.org.
			TXT	"v=spf1 -all"
```

and configure it in your bind config (e.g. at `/etc/bind/named.conf.local`):

```
zone "_le.example.org"       { type master; file "/etc/bind/zones/db._le.example.org"; update-policy { grant _le.example.org.org_ddns_update wildcard *._le.example.org.org. TXT; }; };
```

# Ansible variable defaults

```
# Set the email address associated with the Let's Encrypt account
letsencrypt_account_email: ""

# Install the DNS challenge helper script
letsencrypt_dns_challenge: yes
#letsencrypt_ddns_server: ""
#letsencrypt_ddns_zone: ""
#letsencrypt_ddns_key: ""
#letsencrypt_ddns_privkey: ""

# Create system group 'letsencrypt' for access to certificates
letsencrypt_group: yes
```

## License

This Ansible role is licensed under the GNU GPLv2 or later.

## Author

Copyright 2017 systemli.org <admin@systemli.org>
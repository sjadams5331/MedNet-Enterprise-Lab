# Deployment

> **Standing up the osTicket service desk on Debian 12**
> The host, the Apache / PHP / MariaDB stack, and the osTicket application — provisioned as a dedicated single-application server with a least-privilege database account, forming the foundation the AD-authenticated service desk is built on.

---

## Design Intent

The service desk runs on its own dedicated VM rather than sharing a host with other lab services. A single-application server keeps the platform's attack surface small, makes its resource profile predictable, and mirrors how a hospital would isolate an internet-adjacent intake system from back-office infrastructure. `itsm01` does one job — run osTicket — and everything on the host exists to support that.

The application itself holds no identities and no PHI; it is a workflow and audit layer. That separation is deliberate: identity is delegated entirely to Active Directory (see [02-ad-authentication.md](02-ad-authentication.md)), and the only sensitive data the host stores is the operational record of support work itself.

---

## Host & Platform

| Setting | Value |
|---|---|
| Hostname / FQDN | `itsm01` / `itsm01.mednet.lab` |
| IP Address | `192.168.56.120` (static, host-only `vboxnet0`) |
| Operating System | Debian 12 (Bookworm) — headless |
| Role | Dedicated osTicket application server |

The host is headless and reachable only on the lab's host-only network. A static address was assigned directly on the host-only interface so the server's identity in DNS and across the monitoring tier stays stable — dynamic addressing on a server that other components forward alerts to would be a liability.

---

## Application Stack

osTicket is a PHP application backed by a relational database and fronted by a web server. The supported, low-friction stack on Debian is Apache 2, PHP 8.2, and MariaDB, and that is what is deployed here.

| Component | Role | Rationale |
|---|---|---|
| Apache 2 | Web front end | First-class osTicket support; simple per-vhost TLS, hardened in [06-security-hardening.md](06-security-hardening.md) |
| PHP 8.2 | Application runtime | Supported by osTicket 1.18; the `php-ldap` extension is what makes the AD integration possible |
| MariaDB | Persistent storage | Debian 12 default; drop-in for MySQL and the documented osTicket database |

The PHP extensions osTicket requires are installed alongside the runtime — notably `php-imap`, `php-gd`, `php-mbstring`, `php-intl`, `php-mysqli`, and `php-ldap`. `php-ldap` is called out specifically because it is the dependency that enables LDAPS authentication; without it the directory integration in the next document is impossible.

---

## Database Provisioning

osTicket is given its own database and a dedicated MariaDB account whose privileges are scoped to that schema and nothing else:

```sql
CREATE DATABASE osticket CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;
CREATE USER 'osticket'@'localhost' IDENTIFIED BY '<strong-password>';
GRANT ALL PRIVILEGES ON osticket.* TO 'osticket'@'localhost';
FLUSH PRIVILEGES;
```

The grant is deliberately narrow — `osticket.*`, bound to `localhost` only. The application can fully manage its own schema but has no reach into any other database and cannot connect over the network. This is the same least-privilege principle applied to the directory service account in the next document, expressed at the data tier: every identity in the system, human or service, holds only the access its function requires.

---

## osTicket Installation

The application files are deployed under the Apache web root and served by a dedicated virtual host. osTicket 1.18.x is installed from the official release, the web-based installer is run once to initialise the schema against the database account above, and the resulting `config.php` is then locked down.

After the installer completes, two things are done immediately as part of the baseline:

- **`config.php` is set read-only** so the running application cannot rewrite its own configuration. osTicket explicitly warns when this file remains writable, treating it as a misconfiguration.
- **The installer is retired.** The `setup/` directory is a privileged code path that must not remain reachable on a running instance; its removal is the first item of the hardening pass documented in [06-security-hardening.md](06-security-hardening.md).

The agent control panel is served at `/scp` and the user portal at the site root, with both authentication backends pointed at Active Directory in the next phase.

---

## HIPAA Alignment

This phase establishes the platform, not the controls — but two foundations are laid here that the compliance story later depends on:

- **Least privilege at the data tier** — the scoped, localhost-only database account ensures the application cannot be leveraged to reach data outside its own scope (supports §164.308(a)(4), information access management).
- **A defined, isolated system boundary** — a dedicated single-purpose host gives the audit and access controls introduced later a clean, well-understood surface to apply to.

Accountability and the per-ticket audit trail come online with directory authentication in the following document.

---

## Validation

The deployment is confirmed by:

- `systemctl status apache2 mariadb` reporting both services active and enabled at boot.
- The osTicket user portal and `/scp` agent login both rendering over the host-only network.
- The installer (`setup/`) no longer being reachable, and `config.php` reporting as read-only in the admin panel's system check.

| | |
|:---:|:---:|
| ![osTicket service desk dashboard after installation](../screenshots/deployment_01.png) | ![apache2 and mariadb services active](../screenshots/deployment_02.png) |
| *osTicket dashboard following a successful install* | *Apache and MariaDB running and enabled at boot* |

---

*← Back to [MedNet IT Service Desk](../README.md)*

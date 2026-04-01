# Network Security Lab — Defend the Flag

A shell script automating the hardening and certificate provisioning tasks for the ETH Zurich Network Security Defend the Flag (DtF) exercise. The script addresses five discrete tasks: firewall rule remediation, fine-grained port access control, file permission hardening, automated TLS certificate issuance via ACME, and TLS protocol restriction.

---

## Table of Contents

- [Context](#context)
- [Tasks](#tasks)
  - [Task 1 — Firewall Rule Remediation](#task-1--firewall-rule-remediation)
  - [Task 2 — PostgreSQL Port Isolation](#task-2--postgresql-port-isolation)
  - [Task 3 — Sensitive File Permission Hardening](#task-3--sensitive-file-permission-hardening)
  - [Task 4 — Automated TLS Certificate Issuance](#task-4--automated-tls-certificate-issuance)
  - [Task 5 — TLS Protocol Restriction](#task-5--tls-protocol-restriction)
- [Prerequisites](#prerequisites)
- [Usage](#usage)
- [Security Notes](#security-notes)
- [License](#license)


---

## Context

This script was developed as part of the Network Security course at ETH Zurich. The DtF format presents a live, intentionally misconfigured Linux host that must be hardened under time pressure against a set of predefined attack scenarios. All five tasks are executed sequentially and atomically via a single invocation.

The target host runs nftables as its firewall, NGINX as its web server, and PostgreSQL as its database engine. The ACME CA endpoint is the course-internal instance at `acme.dtf.netsec.inf.ethz.ch`.

---

## Tasks

### Task 1 — Firewall Rule Remediation

A misconfigured nftables rule drops all established and related connections, effectively severing any ongoing session on the host. The script removes this rule with an in-place `sed` edit of `/etc/nftables.conf` and restarts the firewall:

```bash
sudo sed -i '/ct state established,related log prefix "DROPPING PACKET: " drop/d' /etc/nftables.conf
sudo systemctl restart nftables
```

### Task 2 — PostgreSQL Port Isolation

The script adds a whitelist rule allowing TCP traffic on port `5432` exclusively from the grader host (`grader.dtf.netsec.inf.ethz.ch`), followed by a blanket drop rule for all other traffic on the same port. Rules are applied directly to the kernel ruleset, then flushed back to disk to survive a firewall restart:

```bash
sudo nft add rule inet filter input ip saddr grader.dtf.netsec.inf.ethz.ch tcp dport {5432} accept
sudo nft add rule inet filter input tcp dport 5432 drop
sudo nft list ruleset > /etc/nftables.conf
```

Rule ordering is significant: the accept rule is inserted before the drop rule, ensuring the grader's traffic is matched and permitted before the catch-all drop is evaluated.

### Task 3 — Sensitive File Permission Hardening

A plaintext password file at `/var/www/secret/passwords` is world-readable in the default configuration. The script restricts access to the owning user only:

```bash
sudo chmod 600 /var/www/secret/passwords
```

### Task 4 — Automated TLS Certificate Issuance

The script provisions a valid TLS certificate for the student subdomain using [acme-tiny](https://github.com/diafygi/acme-tiny), a minimal ACME client, against the course-internal CA. The full provisioning sequence is:

1. Clone `acme-tiny` into a fresh working directory (`../req_cert`).
2. Generate a 4096-bit RSA account key.
3. Generate a 4096-bit RSA domain key and a CSR with the student subdomain as the Common Name.
4. Create the ACME `http-01` challenge directory under the NGINX web root.
5. Run `acme_tiny.py` to perform the `http-01` challenge and obtain a signed certificate chain, written to `signed_chain.crt`.
6. Inject a TLS-enabled `server` block into `/etc/nginx/nginx.conf` anchored on the `gzip on;` directive, configuring the certificate and key paths.
7. Restart NGINX to apply the configuration.

The ACME directory URL used is:

```
https://acme.dtf.netsec.inf.ethz.ch/acme/default/directory
```

### Task 5 — TLS Protocol Restriction

The default NGINX TLS configuration permits TLSv1, TLSv1.1, and TLSv1.2, all of which are deprecated or cryptographically broken. The script replaces the protocol directive to allow TLS 1.3 exclusively:

```bash
sudo sed -i "s/ssl_protocols TLSv1 TLSv1.1 TLSv1.2; .*/ssl_protocols TLSv1.3;/" /etc/nginx/nginx.conf
sudo systemctl restart nginx
```

---

## Prerequisites

The following must be present and accessible on the target host before running the script:

- `nftables` with a base configuration at `/etc/nftables.conf`
- `nginx` installed and configured with a `gzip on;` directive in the http block (used as the injection anchor in Task 4)
- `openssl` available in `PATH`
- `python3` available in `PATH`
- `git` available in `PATH`
- Network access to the course ACME CA and to `grader.dtf.netsec.inf.ethz.ch`
- The NGINX web root at `/var/www/html` and a plaintext password file at `/var/www/secret/passwords`

---

## Usage

```bash
chmod +x setup.sh
./setup.sh
```

The script runs with `set -euo pipefail` and `set -x`: it will abort immediately on any non-zero exit code, treat unset variables as errors, and print each command to stdout as it executes. All steps are idempotent with the exception of Task 4, which clones into a freshly created directory and will fail if `../req_cert` already exists.

---

## Security Notes

- The `sed`-based injection in Task 4 appends a `server` block inline within the `nginx.conf` http block. This is intentionally minimal for the DtF context and is not recommended practice for production NGINX configuration management.
- The account and domain private keys are generated without passphrase protection, as appropriate for an automated, non-interactive provisioning flow.
- Task 2 resolves the grader hostname to an IP at rule insertion time via `nft`. If the grader's IP changes, the ruleset must be updated manually.

---

## License

See [LICENSE](LICENSE).
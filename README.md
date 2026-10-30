# Centralized LDAP Authentication

This repository contains the complete configuration for a centralized Identity and Access Management (IAM) setup using OpenLDAP. It provides a single source of truth for user authentication across multiple Linux servers in the `ozy.cns` domain.

## Overview

Managing local users (`/etc/passwd`) across multiple servers is a security risk and an administrative burden. This project implements a centralized directory using OpenLDAP, allowing servers (clients) to authenticate users against a remote LDAP database.

This setup eliminates the need for local user accounts on individual servers, enabling centralized password policies, access control, and single sign-on for Linux environments.

## Architecture

The infrastructure consists of two primary nodes:

| Role | Hostname | IP Address | Services |
|------|----------|------------|----------|
| LDAP Master | `mail.ozy.cns` | `172.20.10.3` | OpenLDAP Server (`slapd`), phpLDAPadmin |
| LDAP Client | `vic.vic.cns` | `172.20.10.4` | `nslcd`, NSS, PAM |

### Base DN (Distinguished Name)

The LDAP tree is structured under the base DN `dc=ozy,dc=cns`.

```text
dc=ozy,dc=cns
├── ou=people
│   └── uid=netmgt (Network Manager)
└── ou=groups
    └── cn=Student (POSIX Group)
```

## Key Features

- **Centralized Authentication:** Users log in to the client server using credentials stored on the LDAP master.
- **POSIX Attributes:** Supports Linux-specific attributes (`uidNumber`, `gidNumber`, `homeDirectory`, `loginShell`).
- **Automated Home Directory Creation:** Uses PAM (`pam_mkhomedir`) to automatically create the user's `/home/` directory upon their first login.
- **Web-based Administration:** phpLDAPadmin provides a GUI to manage users, groups, and organizational units without touching the command line.
- **NSS and PAM Integration:** Seamlessly integrates with the Name Service Switch (NSS) and Pluggable Authentication Modules (PAM) on Debian clients.

## Deployment Steps

### Part 1: LDAP Master Server Setup

1. **Install Packages:**
   ```bash
   apt update
   apt install slapd ldap-utils -y
   ```

2. **Configure the Domain:**
   ```bash
   dpkg-reconfigure slapd
   ```
   - DNS domain name: `ozy.cns`
   - Organization name: `Network Security Developers Ltd`
   - Administrator password: `ldapims`
   - Choose No for purging the database (to keep it).
   - Choose Yes to remove the old database (if recreating).

3. **Add Organizational Units (OUs):**
   Create `people.ldif` and `groups.ldif` and apply them:
   ```bash
   ldapadd -x -D cn=admin,dc=ozy,dc=cns -W -f people.ldif
   ldapadd -x -D cn=admin,dc=ozy,dc=cns -W -f groups.ldif
   ```

4. **Add a User (`netmgt`):**
   ```bash
   ldapadd -x -D cn=admin,dc=ozy,dc=cns -W -f user.ldif
   ```

5. **Install phpLDAPadmin:**
   ```bash
   apt install apache2 php libapache2-mod-php phpldapadmin -y
   ```
   Configure `/etc/phpldapadmin/config.php` and access it via `http://172.20.10.3/phpldapadmin`.

### Part 2: LDAP Client Configuration (`vic.vic.cns`)

1. **Install Client Packages:**
   ```bash
   apt install libnss-ldapd libpam-ldapd nslcd ldap-utils -y
   ```

2. **Configure NSLCD:**
   Edit `/etc/nslcd.conf`:
   ```conf
   uri ldap://172.20.10.3
   base dc=ozy,dc=cns
   binddn cn=admin,dc=ozy,dc=cns
   bindpw ldapims
   ```

3. **Update NSSwitch:**
   Edit `/etc/nsswitch.conf` to include `ldap`:
   ```conf
   passwd:         files systemd ldap
   group:          files systemd ldap
   shadow:         files ldap
   ```

4. **Configure PAM for Home Directory Creation:**
   Edit `/etc/pam.d/common-session` and add:
   ```conf
   session optional        pam_mkhomedir.so skel=/etc/skel umask=0077
   ```

5. **Restart Services:**
   ```bash
   systemctl restart nslcd
   systemctl restart nscd
   ```

## Verification

To verify the setup works:

1. **Query the LDAP Server from the Client:**
   ```bash
   getent passwd netmgt
   ```
   Expected Output: `netmgt:x:3000:5000:Network Manager:/home/netmgt:/bin/bash`

2. **Attempt a Login:**
   ```bash
   su - netmgt
   ```
   Expected Behavior:
   - You are prompted for the password (`ldapims`).
   - Upon successful login, a new home directory `/home/netmgt` is created.
   - The `whoami` command returns `netmgt`.

## Technologies Used

- OpenLDAP (`slapd`)
- phpLDAPadmin
- NSLCD (Name Service Switch daemon for LDAP)
- PAM (Pluggable Authentication Modules)
- Debian 13 (Trixie)

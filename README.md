# Centralized LDAP Authentication

This repository contains the OpenLDAP server configuration, phpLDAPadmin setup, and Linux client authentication (NSS/PAM) for the `ozy.cns` domain.

## Architecture
- LDAP Master: 172.20.10.3
- LDAP Client: 172.20.10.4
- Base DN: `dc=ozy,dc=cns`

# guiguide
Account Creation Webpage Administration Guide
# Chapter 1: Web Backend Overview
# Web Backend Overview

This section provides an understanding of the various components and mechanisms involved in the backend of the web application.

## 1. **Web Server Setup (Apache)**
- **Apache HTTP Server** serves as the primary web server.
- Configurations are managed through files like `web.conf`.

## 2. **IDM Integration for Authentication**
- Integrates with an **LDAP server** for authenticating users.
- Apache's LDAP module, as configured, handles this process.

## 3. **Account Request and Management**
- Features a **"request" page** for user account requests.
- Submitted user details are stored in the `pending` & `accounts` directories.

## 4. **Review and Approval Workflow**
- Admins review requests on a **"review" page**.
- Approved requests are processed and queued for account creation.

## 5. **Automated Account Creation**
- A **systemd service** monitors the `accountQueue` file.
- Executes `autoCreateUser.sh` to create accounts from the queue.

## 6. **Mattermost Integration**
- Can send notifications to **Mattermost** using webhooks.
- Notifications are sent based on various actions like successful account request submissions, approvals, and denials.
  

## Conclusion

The backend is a sophisticated ecosystem involving multiple components like a web server, LDAP, automated scripts, and security measures, working together to ensure efficient and secure operations of the web application.

# Chapter 2: System Requirements

This chapter outlines the hardware and software requirements necessary for the proper installation and functioning of the web application, including the integration with an Identity Management (IDM) system like FreeIPA.

## Software Dependencies

### Operating System
- **Linux Distribution**: RHEL 8.8.

### Web Server
- **Apache HTTP Server**
  - Version: Minimum required version, e.g., Apache 2.4.
  - Note any specific Apache modules that are required.

### PHP
- **PHP Version**: Minimum required PHP version, e.g., PHP 7.4.
- **Required PHP Modules**: List necessary PHP modules, e.g., `php-ldap`, `php-mysqli`.

### Additional Software
- List other software dependencies, like Git, Composer, or specific libraries.

### Firewall Settings
- 389/tcp
- 443/tcp
- 80/tcp

### Domain Configuration
- Information about domain setup, DNS requirements, and integration with the IPA server.
- **FreeIPA Client**
  - The web server must be configured as an IPA client to integrate with the domain for user management.

### SELinux
- SELinux state is set to Enforcing

## Conclusion

Ensure that all hardware and software requirements, including the web server's role as an IPA client, are met before proceeding with the installation and configuration of the web application. This preparation will facilitate optimal performance and a smooth integration process.

# Chapter 3: Installation and Configuration

This chapter provides detailed steps for installing and configuring the web directory skeleton from a tarball.

## Installation Steps

### Pre-Installation Checks
- Ensure Apache and other dependencies are already installed.
- Verify sufficient disk space in the target directory.
- Backup existing web directories if necessary.

### Downloading the Tarball
- Obtain the tarball containing the web directory skeleton.
- Use `wget` or similar to download: `wget http://example.com/path/to/tarball.tar.gz`.

### Extracting the Tarball
- Navigate to the web root directory: `cd /var/www/html`.
- Extract the tarball: `tar xzvf /path/to/tarball.tar.gz`.

### Creating Missing Directories
- Manually create missing directories (`accounts`, `removed`, `approved`, `pending`):
  ```bash
  mkdir -p /var/www/html/request/accounts
  mkdir -p /var/www/html/request/removed
  mkdir -p /var/www/html/request/approved
  mkdir -p /var/www/html/request/pending
  ```
### Setting Directory Permissions and SELinux File Contexts
- Assign appropriate permissions:
  ```bash
  chown -R apache:apache /var/www/html/request/*
  chmod -R 755 /var/www/html/request/*
  ```
- Set SELinux contexts for directories needing `rw` access:
  ```bash
  semanage fcontext -a -t httpd_sys_rw_content_t "/var/www/html/request/(accounts|removed|approved|pending)(/.*)?"
  restorecon -Rv /var/www/html/request/(accounts|removed|approved|pending)
  ```
- For scripts requiring `bin_t` context, like `autoCreateUser.sh`:
  ```bash
  semanage fcontext -a -t bin_t "/var/www/html/request/scripts/autoCreateUser.sh"
  restorecon -v /var/www/html/request/scripts/autoCreateUser.sh
  ```
## Configuring `rtweb.conf`
The rtweb.conf file is a critical component of the web application, serving as a central place for specific configurations like Acitve Directory LDAP lookups and Mattermost settings. 

### Accessing the Configuration File
- Locate and open `rtweb.conf` for editing: `vim /path/to/config/rtweb.conf`.

### Configuring LDAP Settings for Active Directory
- **Enable LDAP Integration**
  - `enable`: Set to `true` to enable, or `false` to disable.
- **Bind User**
  - `bind_user`: Specify the username for the LDAP bind operation. Use a service account created in Active Directory.
- **Bind DN (Distinguished Name)**
  - `bind_dn`: Enter the full Distinguished Name in Active Directory format, e.g., `CN=BindUser,CN=Users,DC=example,DC=com`.
- **Bind Password**
  - `bind_password`: Enter the password for the bind user.
  - **Security Note**: Ensure this password is securely stored.
- **Search DN**
  - `search_dn`: Provide the base DN for LDAP searches, e.g., `CN=Users,DC=example,DC=com`.

### Configuring Mattermost Integration 
- **Enable Mattermost Integration**
  - `enable`: Set to `true` to activate, or `false` to disable.
- **Channel URL**
  - `channel_url`: Enter the Mattermost channel webhook URL.
- **Direct Message**
  - `direct_msg`: Specify the webhook URL for direct messages.

### Saving the Configuration
- After making the necessary changes, save and close the file.

### Testing the Configuration
- Test LDAP integration by attempting a user login or similar operation.
- If Mattermost integration is enabled, test by sending a notification.

### Final Steps
- Restart any services if required to apply the new configuration.
- Conduct a final review to ensure all settings are correctly implemented.

## Conclusion

Configuring the `rtweb.conf` file is a crucial step in ensuring that LDAP and Mattermost functionalities are properly integrated and operational within the web application. Special attention should be given to the security of sensitive information like bind passwords.

# Chapter 4: Apache Configuration 

This chapter provides detailed insights into configuring the `/etc/httpd/conf.d/web.conf` file for the Apache web server on a RHEL 8 system. The focus is on setting up virtual hosts for `gui.example.com`, configuring LDAP integration, and ensuring directory-level settings.

### Virtual Hosts Configuration Explained
```apache
# Start of the VirtualHost block for HTTPS
<VirtualHost *:443>
    # ServerName specifies the domain name of the server
    ServerName gui.example.com

    # DocumentRoot is the directory where the web files for this host are stored
    DocumentRoot "/var/www/html/request"

    # SSLCertificateFile and SSLCertificateKeyFile point to the SSL certificate and private key
    SSLCertificateFile "/path/to/certificate.crt"
    SSLCertificateKeyFile "/path/to/private.key"

    # The <Directory> block contains specific directives for the web application directory
    <Directory "/var/www/html/request">
        # DirectoryIndex sets the file served as the directory index (default page when no file is specified)
        DirectoryIndex index.php

        # Options -Indexes prevents the server from returning a listing of the directory contents
        Options -Indexes

        # LDAP configuration directives would be placed here (not shown in this snippet)
    </Directory>
</VirtualHost>

# Additional VirtualHost block for the /request/review path
<VirtualHost *:443>
    ServerName gui.example.com
    DocumentRoot "/var/www/html/request/review"

    SSLCertificateFile "/path/to/certificate.crt"
    SSLCertificateKeyFile "/path/to/private.key"

    <Directory "/var/www/html/request/review">
        DirectoryIndex index.php
        Options -Indexes
        # Similar LDAP and access control settings as above
    </Directory>
</VirtualHost>
```

### LDAP Configuration Explained
```apache
# Enables Basic Authentication
AuthType Basic

# Text displayed at the login prompt
AuthName "Protected"

# Specifies LDAP as the authentication provider
AuthBasicProvider ldap

# URL to connect to the LDAP server, including server address, search base, and attribute
# Format: ldap://[server address]:[port]/[search base]?[attribute]
# Example: "ldap://idm.example.com:389/cn=users,cn=accounts,dc=example,dc=com?uid"
AuthLDAPURL "ldap://idm.example.com:389/cn=users,cn=accounts,dc=example,dc=com?uid"

# DN of the user Apache uses to bind to the LDAP server, should have read access
# Example: "CN=BindUser,CN=Users,DC=example,DC=com"
AuthLDAPBindDN "CN=BindUser,CN=Users,DC=example,DC=com"

# Password for the BindDN user, ensure secure storage
AuthLDAPBindPassword "password"

# Restricts access to members of a specified LDAP group
# Example: cn=webapp,cn=groups,cn=accounts,dc=example,dc=local
Require ldap-group cn=webapp,cn=groups,cn=accounts,dc=example,dc=local
```
### Helpful commands for troubleshooting LDAP
Running an `ldapsearch` command that mirrors the LDAP configuration specified in your Apache setup is an excellent way to validate the LDAP settings independently. The `ldapsearch` utility is a command-line tool for searching and querying an LDAP directory.
1. Extracting LDAP Paramters from Apache Config:
* Let's assume your Apache LDAP configuration looks something like this (as per your web.conf file):
   ```apache
   AuthLDAPURL "ldap://idm.example.com:389/cn=users,cn=accounts,dc=example,dc=com?uid"
   AuthLDAPBindDN "CN=BindUser,CN=Users,DC=example,DC=com"
   AuthLDAPBindPassword "password"
   ```
   * Here, `idm.example.com` is your LDAP server, `389` is the port, `cn=users,cn=accounts,dc=example,dc=com` is the search base, and `uid` is the attribute used for searching.
2. Constructing the ldapsearch Command:
* Based on the above configuration, your ldapsearch command would look something like this:
  ```bash
  ldapsearch -x -H ldap://idm.example.com:389 -D "CN=BindUser,CN=Users,DC=example,DC=com" -W -b "cn=users,cn=accounts,dc=example,dc=com" "(uid=yourTestUsername)"
  ```
  
  - **`-x`**: Specifies simple authentication instead of SASL (Simple Authentication and Security Layer).
  - **`-H`**: LDAP server URI. Replace `idm.example.com:389` with your LDAP server's address and port.
  - **`-D`**: 'Bind DN' (Distinguished Name), the username to connect to the LDAP server. Replace with your Bind DN.
  - **`-W`**: Prompts for the bind password for security reasons.
  - **`-b`**: Base DN for the search, should be the same as in your Apache LDAP configuration.









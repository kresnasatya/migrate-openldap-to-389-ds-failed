# Migrate from OpenLDAP to 389 Directory Server Failed
This repo contains steps to reproduce migration from OpenLDAP to 389 Directory Server. Current result is failed.

## Environment

- Ubuntu 22.04 in WSL 2.
- OpenLDAP version 2.5.13 (run `slapd -VV`)
- 389 Directory Server version 2.0.15-1 (https://packages.ubuntu.com/jammy/389-ds)

## Initial setup

1. Download distro Ubuntu 22.04 with command `wsl --install -d Ubuntu-22.04`.
2. Set username and password in distro.
3. Open distro in terminal.
4. Create `wsl.conf` file in `/etc` directory distro Ubuntu 22.04.
    
    ```bash
    # file /etc/wsl.conf
    [boot]
    systemd=true
    ```
5. Close terminal.
6. Turn off wsl in power shell with command `wsl --shutdown`.
7. Open terminal and find distro Ubuntu 22.04.

## Steps to reproduce

1. Run `sudo apt update -y`
2. Run `sudo apt install net-tools -y` to know the IP Address from this distro server in WSL2.
3. Run `sudo apt install slapd ldap-utils`. Setup admin password for LDAP.
4. Run `sudo dpkg-reconfigure slapd` to reconfigure LDAP.
    ```bash
    Administrator password: 1234567890
    Confirm administrator password: 1234567890
    Domain name: example.org
    Organization name: example.org
    ```
5. Run `ldapwhoami -H ldap://localhost -x` to check if result is anonymous. If not then check if slapd is run or not with command `sudo service slapd status`. If doesn’t start then run `sudo service slapd start`.
6. Run `ifconfig` to get IP Address from distro.
7. Open Apache Directory Studio and add hostname with IP Address from distro with default port 389 for LDAP authentication use `dn=admin,dc=example,dc=org` with password setup from LDAP. Here's the image result after connected.
![Screenshot OpenLDAP in Apache Directory Studio](/sc-1-openldap.PNG)
8. Add some sample data and store them as a backup. Please see the backup in folder in this repo.
    ```bash
    # backup command
    sudo slapcat -n 0 -l config.ldif
    sudo slapcat -n 1 -l data.ldif
    # see how to restore them in this guide: https://tylersguides.com/articles/backup-restore-openldap/
    ```
9. Install 389 Directory Server (389 DS) with command `sudo apt install 389-ds -y`.
10. Stop slapd service with `sudo service slapd stop`.
11. Create instance template in `/root` directory with name `instance.inf`.
    ```bash
    # /root/instance.inf
    [general]
    config_version = 2

    [slapd]
    instance_name = localhost
    root_dn = cn=admin,dc=example,dc=org
    root_password = 1234567890
    ```
12. Make instance with command `sudo dscreate from-file /root/instance.inf`.
13. Check if instance already running with command `sudo systemctl status dirsrv@localhost`.
14. Make folder `backup-openldap` and move to that directory. Copy slapd.d directory with command `sudo cp -a /etc/ldap/slapd.d /root/backup-openldap/slapd.d`. Copy data from LDAP already in step 8 with command `sudo slapcat -n 1 -f data.ldif`.
15. Run `sudo openldap_to_ds --confirm localhost slapd.d data.ldif` from `backup-openldap` directory.
16. Migration done and check if config and data from OpenLDAP already move to 389 DS with Apache Directory Studio. The result is empty like the image below.

![Screenshot 389 DS in Apache Directory Studio](/sc-2-389ds.PNG)

17. As I follow from YouTube tutorial **Migrating to 389-ds from openldap2** (https://www.youtube.com/watch?v=qrbtWOXOhtA) it says that data already imported but we cannot see that because it was like that if we run migration with `openldap_to_ds` command.
18. So, I try with make `.dsrc` file in root directory with command `sudo dsctl localhost dsrc create --uri ldapi://%%2fvar%%2frun%%2fslapd-localhost.socket --binddn cn=admin,dc=example,dc=org --basedn dc=example,dc=org`. Here's the result I get.

    ```bash
    [localhost]
    uri = ldapi://%2fvar%2frun%2fslapd-localhost.socket
    basedn = dc=example,dc=org
    binddn = cn=admin,dc=example,dc=org
    saslmech = EXTERNAL
    ```
19. Run command `dsidm localhost client_config ldap.conf` get message **Must provide basedn!**
20. I lost track and no have idea what should I do.

**Note:** I rollback to OpenLDAP with command below.

```bash
sudo dsctl localhost remove --do-it
sudo service slapd start
```
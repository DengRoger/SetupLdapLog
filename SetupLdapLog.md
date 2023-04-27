# LDAP setup log 

- Install ldapServer
    
    0. install basic packakage with setting admin password 
        and set rogerdeng.net to root dn 

        ```
        sudo apt install slapd
        sudo dpkg-reconfigure slapd
        sudo apt install ldap-utils
        ```
    
    1. check if ldap is working

        ```
        ldapsearch -x -b '' -s base '(objectclass=*)' namingContexts
        ```
        ```
        systemctl status slapd
        ```

- Creating a LDAP hierarchical organizational tree 

    0. Creating an LDAP hierarchical organizational tree
        ![](https://i.imgur.com/zDCJ6Ez.png)

    
    1. Create Group (ou)
        ```ldif
        #group.ldif
        dn: ou=Groups,dc=rogerdeng,dc=net
        objectClass: organizationalUnit
        ou: Groups
        ```
        ```
        ldapadd -x -D "cn=admin,dc=rogerdeng,dc=net" -W -f group.ldif
        ```
    2. Create People (ou)
        ```ldif
        #people.ldif
        dn: ou=People,dc=rogerdeng,dc=net
        objectClass: organizationalUnit
        ou: People
        ```
        ```
        ldapadd -x -D "cn=admin,dc=rogerdeng,dc=net" -W -f people.ldif
        ```

    3. Create RogerGroup under Groups
        ```ldif
        #rogergroup.ldif
        dn: cn=RogerGroup,ou=Groups,dc=rogerdeng,dc=net
        objectClass: groupOfNames
        cn: RogerGroup
        member: cn=roger,ou=People,dc=rogerdeng,dc=net
        ```
        ```
        ldapadd -x -D "cn=admin,dc=rogerdeng,dc=net" -W -f rogergroup.ldif
        ```

    4. Create Sudoer Group under Groups
        ```ldif
        #sudoer.ldif
        dn: cn=Sudoer,ou=Groups,dc=rogerdeng,dc=net
        objectClass: posixGroup
        cn: Sudoer
        member : cn=roger,ou=People,dc=rogerdeng,dc=net
        ```
        ```
        ldapadd -x -D "cn=admin,dc=rogerdeng,dc=net" -W -f sudoer.ldif
        ```

    5. Create user roger under People
        ```ldif
        #roger.ldif
        dn: cn=roger,ou=People,dc=rogerdeng,dc=net
        objectClass: inetOrgPerson
        objectClass: posixAccount
        objectClass: shadowAccount
        objectClass: top
        objectClass: person
        cn: Roger
        sn: Deng
        uidNumber: 1000
        gidNumber: 1000
        homeDirectory: /home/roger
        loginShell: /bin/bash
        uid: roger
        mail: rogerdeng92@gmail.com
        memberOf: cn=RogerGroup,ou=Groups,dc=rogerdeng,dc=net
        ```
        
        - !issue
            The membersOf plugin is not loaded by default in LDAP !
            as normally membersOf.la is at /etc/ldap/schema/ but it is not there in my system, and I found it at /usr/lib/ldap/schema/ so I import it.
            
            - import memberOf.la to ldap : 
                - find slapd.conf
                    ```
                    sudo find / -name slapd.conf
                    ```
                - edit slapd.conf 
                    - add `moduleload /usr/lib/ldap/membersOf.la` in slapd.conf
                
                - restart slapd 
                    ```
                    sudo systemctl restart slapd
                    ```
                - check if membersOf loaded 
                    ```
                    ldapsearch -x -LLL -b cn=config -D cn=admin,cn=config -W olcModuleLoad=memberof
                    ```
                    and it should show that : 
                    `dn: cn=config
olcModuleLoad: {0}memberof.la`
            
                
            

- Integrating LDAP into the System

    0. 
        install nscd and nslcd 
    1. 
        Configure LDAP client authentication by editing /etc/nsswitch.conf file and adding the "ldap" option to the "passwd", "group", and "shadow" lines. This tells the system to use LDAP for user and group information:

        ```
        passwd:         compat ldap
        group:          compat ldap
        shadow:         compat ldap
        ```

    2. 
        Configure LDAP client authentication by editing /etc/pam.d/common-auth and adding the following line at the top:

        ```
        auth    sufficient      pam_ldap.so use_first_pass
        ```

        This tells the PAM authentication module to use LDAP as a source of authentication information.

    3. 
        Configure LDAP client account management by editing /etc/pam.d/common-account and adding the following line at the top:

        ```
        account sufficient      pam_ldap.so
        ```

        This tells the PAM account management module to use LDAP as a source of account information.

    4. Configure LDAP client password management by editing /etc/pam.d/common-password and adding the following line at the top:

        ```
        password sufficient      pam_ldap.so use_authtok
        ```

        This tells the PAM password management module to use LDAP to change passwords.

    5. Configure LDAP client session management by editing /etc/pam.d/common-session and adding the following line at the top:

        ```
        session required        pam_ldap.so
        ```

    6. Configure the LDAP client by editing the /etc/ldap/ldap.conf file and adding the following lines:

        ```
        uri ldap://ldap.example.com
        base dc=example,dc=com
        ldap_version 3
        ```

        Replace "ldap.example.com" to "ldap://127.0.0.1" \
        replace "dc=example,dc=com" to "dc=rogerdeng,dc=net"
        
    7. Restart the necessary services:
        ```
        sudo systemctl restart nscd
        sudo systemctl restart nslcd
        ```

        These commands restart the Name Service Cache Daemon (nscd) and the Name Service LDAP Client Daemon (nslcd), which are responsible for caching and querying LDAP information.

        After completing these steps, you should be able to use LDAP for authentication and account information on your Linux system.
        
- LDAP over TLS 
    - instal openssl `sudo apt-get install openssl` 
    - makesure `/etc/hostname` is correct
    - in my case :
        - `hostname -f` :  `ldapServer.rogerdeng.net`
        - `hostname` :     `ldapServer`
    - `cd ~/ssl-ldap`
    - `openssl genrsa -aes128 -out ldapServer.rogerdeng.net.key 4096`
    - `openssl rsa -in ldapServer.rogerdeng.net.key -out ldapServer.rogerdeng.net.key`
        - make sure FDQN are same as `hostname -f`
    - `openssl req -new -days 3650 -key ldapServer.rogerdeng.net.key -out ldapServer.rogerdeng.net.csr `
    - `sudo cp ~/ssl-ldap/*.key /etc/ldap/sasl2/`
    - `sudo cp ~/ssl-ldap/*.crt /etc/ldap/sasl2/`
    - `sudo cp /etc/ssl/certs/ca-certificates.crt /etc/ldap/sasl2/`
    - `sudo groupadd ssl-cert`
    - `sudo usermod -aG ssl-cert openldap`
    - `sudo chown :ssl-cert /etc/ldap/sasl2`
    - make file sslLdap.ldif : 
        ```
        dn: cn=config
        changetype: modify
        add: olcTLSCACertificateFile
        olcTLSCACertificateFile: /etc/ldap/sasl2/ca-certificates.crt
        -
        replace: olcTLSCertificateFile
        olcTLSCertificateFile: /etc/ldap/sasl2/ldapServer.rogerdeng.net.crt
        -
        replace: olcTLSCertificateKeyFile
        olcTLSCertificateKeyFile: /etc/ldap/sasl2/ldapServer.rogerdeng.net.key
        ```
    - `sudo ldapmodify -Y EXTERNAL -H ldapi:/// -f sslLdap.ldif`
    - `sudo vim /etc/default/slapd` add `ldaps:///` in to SLAPD_SERVICES
    - `sudo vim /etc/ldap/ldap.conf` exchange 
        ```
        TLS_CACERT	/etc/ldap/sasl2/ca-certificates.crt
        TLS_REQCERT	allow
        ```
    - `sudo systemctl restart slapd`
    - `sudo systemctl restart slapd.service`
    - `ldapsearch -x -H ldaps://127.0.0.1 =b "dc=rogerdeng,dc=net"` to make sure tls(ldaps) working 

- Forces TLS over LDAP 
    - make sure your database type : 
        `sudo ldapsearch -H ldapi:// -Y EXTERNAL -b "cn=config" -LLL -Q "(olcSuffix=*)" dn olcSuffix`
    - vim ~/forcetls.ldif
        ```
        dn: olcDatabase={1}<change to your database type>,cn=config
        changetype: modify
        add: olcSecurity
        olcSecurity: tls=1
        ```
    - `sudo ldapmodify -H ldapi:// -Y EXTERNAL -f forcetls.ldif`
    - `sudo service slapd force-reload`
    - `ldapsearch -H ldap:// -x -b "dc=rogerdeng,dc=net" -LLL -Z dn`
    
# LDAP setup log 

- Install ldapServer
    
    0. install basic packakage with setting admin password 
        and set rogerdeng.net to root dn 

        ```
        sudo apt -y install slapd ldap-utils libnss-ldap libpam-ldap libpam-ldapd gnutls-bin ssl-cert 
        sudo dpkg-reconfigure slapd
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
        objectClass: groupOfNames
        cn: Sudoer
        member : cn=roger,ou=People,dc=rogerdeng,dc=net
        ```
        ```
        ldapadd -x -D "cn=admin,dc=rogerdeng,dc=net" -W -f sudoer.ldif
        ```

    5. Create user roger under People
        ```ldif
        dn: cn=rogerdeng,ou=People,dc=rogerdeng,dc=net
        objectClass: inetOrgPerson
        objectClass: posixAccount
        objectClass: shadowAccount
        objectClass: organizationalPerson
        objectClass: top
        objectClass: person
        cn: rogerdeng
        sn: Deng
        uidNumber: 1002
        gidNumber: 1002
        homeDirectory: /home/rogerdeng
        loginShell: /bin/bash
        uid: rogerdeng
        userPassword: <yourPassword>
        mail: rogerdeng92@gmail.com
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

    - create CA private key 
        ```
        sudo certtool --generate-privkey --bits 4096 --outfile /etc/ssl/private/mycakey.pem
        ```
    
    - create template of CA 
        - vim /etc/ssl/ca.info 
        ```
        cn = rogerdeng
        ca
        cert_signing_key
        expiration_days = 3650
        ```
        
    - signed the CA 
        ```
        sudo certtool --generate-self-signed \
        --load-privkey /etc/ssl/private/mycakey.pem \
        --template /etc/ssl/ca.info \
        --outfile /usr/local/share/ca-certificates/mycacert.crt
        ```
        
    - let new CA be trusted 
        ```
        sudo update-ca-certificates
        ```
        
    - point 
        ```
        sudo certtool --generate-privkey \
        --bits 2048 \
        --outfile /etc/ldap/ldap01_slapd_key.pem
        ```
        
    - edit ldap01 info  `vim /etc/ssl/ldap01.info`
        ```
        organization = rogerdeng
        cn = ldap.rogerdeng.net
        tls_www_server
        encryption_key
        signing_key
        expiration_days = 365
        ```
        
    - let openldap read the CA 
        ```
        sudo chgrp openldap /etc/ldap/ldap01_slapd_key.pem
        sudo chmod 0640 /etc/ldap/ldap01_slapd_key.pem
        ```
        
    - tell slapd about our TLS work via the slapd-config
        - ldapTls.ldif
        ```
        dn: cn=config
        add: olcTLSCACertificateFile
        olcTLSCACertificateFile: /etc/ssl/certs/mycacert.pem
        -
        add: olcTLSCertificateFile
        olcTLSCertificateFile: /etc/ldap/ldap01_slapd_cert.pem
        -
        add: olcTLSCertificateKeyFile
        olcTLSCertificateKeyFile: /etc/ldap/ldap01_slapd_key.pem
        ```
        `sudo ldapmodify -Y EXTERNAL -H ldapi:/// -f ldapTls.ldif`
        
    - add `ldaps:///` into `/etc/default/slapd`
        - `vim /etc/default/slapd`
            - `SLAPD_SERVICES="ldap:/// ldapi:/// ldaps:///"`
            
    - restart sladp
        - `sudo systemctl restart slapd`
    
    - Test tls 
        `ldapwhoami -x -ZZ -H ldap://ldap.rogerdeng.net`
    
    - Test ldaps
        `ldapwhoami -x -H ldaps://ldap.rogerdeng.net`

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
    - `sudo systemctl restat`
    - `ldapsearch -H ldap:// -x -b "dc=rogerdeng,dc=net" -LLL -Z dn`
    
- reference document:
    - https://blueskyson.github.io/2021/06/15/LDAP-setting/
    - https://linux.vbird.org/somepaper/20050817-ldap-3.pdf
    - https://ubuntu.com/server/docs/service-ldap-with-tls
    - https://www.openldap.org/doc/admin24/tls.html
    - https://www.digitalocean.com/community/tutorials/how-to-encrypt-openldap-connections-using-starttls#setting-up-the-client-machines
    - https://idmoim.blogspot.com/2014/05/ldapadd-insufficient-access-50-openldap.html

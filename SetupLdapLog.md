# LDAP setup log 

- Install ldapServer
    -

    0. install basic packakage with setting admin password 
        and set rogerdeng.net to root dn 

        ```
        sudo apt -y install slapd ldap-utils libnss-ldap libpam-ldapd gnutls-bin ssl-cert 
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
    -
    
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
        homeDirectory: /mnt/home/rogerdeng
        loginShell: /bin/bash
        uid: rogerdeng
        mail: rogerdeng92@gmail.com
        gidNumber: 2001
        uidNumber: 2001
        userPassword: <yourPassword>
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
    -
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
    - 
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
        
- Setup client CA 
    -
     - generate ca pair 
        ```
        mkdir ldapconsumer-ssl
        cd ldapconsumer-ssl
        certtool --generate-privkey \
        --bits 2048 \
        --outfile ldapconsumer_slapd_key.pem
        ```
    
    - create ca info for ldapconsumer
        ```
        organization = rogerdeng
        cn = ldapconsumer.rogerdeng.net
        tls_www_server
        encryption_key
        signing_key
        expiration_days = 365
        ```
    - create consumer's ca 
        ```
        sudo certtool --generate-certificate \
        --load-privkey ldapconsumer_slapd_key.pem \
        --load-ca-certificate /etc/ssl/certs/mycacert.pem \
        --load-ca-privkey /etc/ssl/private/mycakey.pem \
        --template ldapconsumer.info \
        --outfile ldapconsumer_slapd_cert.pem
        ```
    - send ldapconsumer-ssl folder to ldapconusmer and repeat LDAP over TLS
    
- Forces TLS over LDAP 
    -
    
    - both of consumer and provider have to setup  
    
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

- Setup workstation(client) 
    -
    `sudo apt -y install ldap-utils libnss-ldap libpam-ldapd gnutls-bin ssl-cert`
    - set u're base dn 
    - set server to provider or consumer or both 
    - set send u're ca to workstation 
        - send provider:/usr/local/share/ca-certificates/mycacert.crt to workstation:/ /etc/ssl/certs/ldapCA.crt
        - set `/etc/ldap/ldap.conf`
            ```
            URI         <ldapprovider domainname>
            BINDDN      cn=admin,dc=rogerdeng,dc=net
            BINDPW      <your password>
            
            # and also set u're ca at here and start tls 
            TLS_CACERT      /etc/ssl/certs/ldapCA.crt
            ssl start_tls
            ```
        - repeat Integrating LDAP into the System and restart all of packages 

- HA with mirror mode 
    - 
    - vim syncprov.ldif  #import syncprov and setup syncprov 
        - both of provider and consumer have to setup 
        ```
        dn: cn=module,cn=config
        objectClass: olcModuleList
        cn: module
        olcModulePath: /usr/lib/ldap
        olcModuleLoad: syncprov.la
        
        dn: olcOverlay=syncprov,olcDatabase={1}mdb,cn=config
        objectClass: olcOverlayConfig
        objectClass: olcSyncProvConfig
        olcOverlay: syncprov
        olcSpCheckpoint: 100 10
        olcSpSessionLog: 100
        ```
    
    - vim HA.ldif (for provider)
        ```
        dn: cn=config
        changetype: modify
        replace: olcServerID
        olcServerID: 1 

        dn: olcDatabase={1}mdb,cn=config
        changetype: modify
        replace: olcSyncRepl
        olcSyncRepl: rid=010
                     provider=ldap://ldapconsumer.rogerdeng.net:389 
                     bindmethod=simple
                     binddn="cn=admin,dc=rogerdeng,dc=net"
                     credentials=<ldapconsumer admin pw>
                     searchbase="dc=rogerdeng,dc=net"
                     filter="(objectClass=*)"
                     scope=sub
                     schemachecking=on
                     attrs="*,+"
                     type=refreshAndPersist
                     retry="5 5 300 +"
                     interval=00:00:00:05
                     starttls=critical tls_reqcert=demand
                     
        -
        add: olcMirrorMode
        olcMirrorMode: TRUE
        -
        add: olcDbIndex
        olcDbIndex: entryUUID eq
        -
        add: olcDbIndex
        olcDbIndex: ent
        ```
        
    - vim HA.ldif (for consumer)
    
        ```
        dn: cn=config
        changetype: modify
        replace: olcServerID
        olcServerID: 2

        dn: olcDatabase={1}mdb,cn=config
        changetype: modify
        replace: olcSyncRepl
        olcSyncRepl: rid=010
                     provider=ldap://ldapprovider.rogerdeng.net:389 
                     bindmethod=simple
                     binddn="cn=admin,dc=rogerdeng,dc=net"
                     credentials=<ldapprovider admin pw>
                     searchbase="dc=rogerdeng,dc=net"
                     filter="(objectClass=*)"
                     scope=sub
                     schemachecking=on
                     attrs="*,+"
                     type=refreshAndPersist
                     retry="5 5 300 +"
                     interval=00:00:00:05
                     starttls=critical tls_reqcert=demand
                     
        -
        add: olcMirrorMode
        olcMirrorMode: TRUE
        -
        add: olcDbIndex
        olcDbIndex: entryUUID eq
        -
        add: olcDbIndex
        olcDbIndex: entryCSN eq
        ```
        
    - modify ldapserver (both)
        `ldapmodify -Y EXTERNAL -H ldapi:/// -f HA.ldif -W`
    
    - check if hierarchical organizational tree are same 

- permissions setup 
    -
    
    - add sudoers at root 
    
        - vim sudoers.ldif 
            ```
            dn: ou=Sudoers,dc=rogerdeng,dc=net
            objectClass: organizationalUnit
            objectClass: top
            ou: Sudoers
            ```
            
        - vim sudoPermissions.ldif
            ```
            dn: cn=%sudoer,ou=Sudoers,dc=rogerdeng,dc=net
            cn: %sudoer
            objectClass: sudoRole
            objectClass: top
            sudoCommand: ALL
            sudoHost: ALL
            sudoUser: %sudoer
            ```
            
        `ldapadd -x -D "cn=admin,dc=rogerdeng,dc=net" -W -f ?.ldif
`
        
        - add sudo schema : vim Sudoschema.ldif 
            ``` 
            dn: cn=sudo,cn=schema,cn=config
            objectClass: olcSchemaConfig
            cn: sudo
            olcAttributeTypes: ( 1.3.6.1.4.1.15953.9.1.1 NAME 'sudoUser' DESC 'User(s) who may  run sudo' EQUALITY caseExactIA5Match SUBSTR caseExactIA5SubstringsMatch SYNTAX 1.3.6.1.4.1.1466.115.121.1.26 )
            olcAttributeTypes: ( 1.3.6.1.4.1.15953.9.1.2 NAME 'sudoHost' DESC 'Host(s) who may run sudo' EQUALITY caseExactIA5Match SUBSTR caseExactIA5SubstringsMatch SYNTAX 1.3.6.1.4.1.1466.115.121.1.26 )
            olcAttributeTypes: ( 1.3.6.1.4.1.15953.9.1.3 NAME 'sudoCommand' DESC 'Command(s) to be executed by sudo' EQUALITY caseExactIA5Match SYNTAX 1.3.6.1.4.1.1466.115.121.1.26 )
            olcAttributeTypes: ( 1.3.6.1.4.1.15953.9.1.4 NAME 'sudoRunAs' DESC 'User(s) impersonated by sudo (deprecated)' EQUALITY caseExactIA5Match SYNTAX 1.3.6.1.4.1.1466.115.121.1.26 )
            olcAttributeTypes: ( 1.3.6.1.4.1.15953.9.1.5 NAME 'sudoOption' DESC 'Options(s) followed by sudo' EQUALITY caseExactIA5Match SYNTAX 1.3.6.1.4.1.1466.115.121.1.26 )
            olcAttributeTypes: ( 1.3.6.1.4.1.15953.9.1.6 NAME 'sudoRunAsUser' DESC 'User(s) impersonated by sudo' EQUALITY caseExactIA5Match SYNTAX 1.3.6.1.4.1.1466.115.121.1.26 )
            olcAttributeTypes: ( 1.3.6.1.4.1.15953.9.1.7 NAME 'sudoRunAsGroup' DESC 'Group(s) impersonated by sudo' EQUALITY caseExactIA5Match SYNTAX 1.3.6.1.4.1.1466.115.121.1.26 )
            olcObjectClasses:  ( 1.3.6.1.4.1.15953.9.2.1 NAME 'sudoRole' SUP top STRUCTURAL DESC 'Sudoer Entries' MUST ( cn ) MAY ( sudoUser $ sudoHost $ sudoCommand $ sudoRunAs $ sudoRunAsUser $ sudoRunAsGroup $ sudoOption $ description ) )
            ```
            `ldapmodify -Y EXTERNAL -H ldapi:/// -f ?.ldif -W`
        
- Create NFS server
    -
    - install package 
        `sudo apt-get install nfs-kernel-server`
    - allow ID mapping 
        `vim /etc/default/nfs-common`
        `NEED_IDMAPD=yes`
    - NFS server setup 
        `vim /etc/exports`
        ```
        /mnt/data 10.1.1.0/24(rw,nohide,insecure,sync,no_root_squash)
        ```
        - no_root_squash explain (https://linux.vbird.org/linux_server/centos6/0330nfs.php)

- mount NFS server by autofs 
    -
    - install autofs
        `sudo apt-get install autofs`
    
    - automount 
        - vim /etc/auto.master
            `/mnt  /etc/auto.nfsdb  --timeout=180`
        - vim etc/auto.nfsdb
            `home  -fstype=nfs,rw,soft,intr  nfs.rogerdeng.net:/mnt/data/home`
        - restart and enable 
            `sudo systemctl restart autofs`
            `sudo systemctl enable autofs`
            
- autoCreateuser script
    -
    ```sh
    #!/bin/bash
    # add user to ldap
    admin="cn=admin,dc=rogerdeng,dc=net"
    ldap_server="ldapprovider.rogerdeng.net"

    # input ldap password to ldap_passwd
    read -s -p "Enter LDAP Password: " ldap_passwd
    echo

    #check if ldap_passwd correct , if not exit
    ldapsearch -x -w $ldap_passwd -D $admin -H ldap://$ldap_server -b "dc=rogerdeng,dc=net" -s base > /dev/null 2>&1

    if [ $? -ne 0 ];
    then
        echo "Invalid LDAP Password"
        exit 1
    else
        read -p "Enter User Name: " user_name
        if [ -z "$user_name" ];
        then
            echo "user_name is empty"
            echo
            exit 1
        fi

        result=$(ldapsearch -x -H ldap://$ldap_server -b "ou=People,dc=rogerdeng,dc=net" "uid=$user_name")
        last_line=$(echo "$result" | tail -n 1)
        if [ "$last_line" = "# numEntries: 1" ];
        then
            echo "User $user_name already exists"
            exit 1
        else
            # get user password
            read -s -p "Enter User Password: " user_passwd
            echo

            if [ -z "$user_passwd" ];
            then
                echo "user_passwd is empty"
                echo
                exit 1
            fi

            # get user email
            read -p "Enter User Email: " user_email
            echo

            cnt=$(ldapsearch -x -H ldap://$ldap_server -b "ou=People,dc=rogerdeng,dc=net" "(uidNumber>=2001)" uidNumber | grep "uidNumber:" | awk '{print $2}' | wc -l)
            uidNumber=$(($cnt+2000))
            gidNumber=$(($cnt+2000))

            # add user to ldap
            ldapadd -x -D "cn=admin,dc=rogerdeng,dc=net" -w $ldap_passwd -H ldap://$ldap_server <<EOF 2>/dev/null
    dn: cn=$user_name,ou=People,dc=rogerdeng,dc=net
    objectClass: inetOrgPerson
    objectClass: posixAccount
    objectClass: shadowAccount
    objectClass: organizationalPerson
    objectClass: top
    objectClass: person
    cn: $user_name
    sn: $user_name
    homeDirectory: /mnt/home/$user_name
    loginShell: /bin/bash
    uid: $user_name
    mail: $user_email
    userPassword: $user_passwd
    gidNumber: $gidNumber
    uidNumber: $uidNumber
    EOF
            if [ $? -ne 0 ];
            then
                echo "LDAP add failed!"
                exit 1
            fi
            # create user home directory if can't create home directory echo error message and exit
            sudo mkdir -p /mnt/home/$user_name
            # set permission to user home directory
            sudo chown -R $user_name: /mnt/home/$user_name
            sudo chmod 700 /mnt/home/$user_name

        fi
    fi
    ```
- ACL
    -

- reference document:
    - https://blueskyson.github.io/2021/06/15/LDAP-setting/
    - https://linux.vbird.org/somepaper/20050817-ldap-3.pdf
    - https://ubuntu.com/server/docs/service-ldap-with-tls
    - https://www.openldap.org/doc/admin24/tls.html
    - https://www.digitalocean.com/community/tutorials/how-to-encrypt-openldap-connections-using-starttls#setting-up-the-client-machines
    - https://idmoim.blogspot.com/2014/05/ldapadd-insufficient-access-50-openldap.html

### Install slapd and ldap-utils:
	sudo apt-get install slapd ldap-utils

### Create database, by creating a ldif-file and add content to it: 
sudo vim /etc/ldap/slapd.d/db.ldif

	# Create database for domain danetest.se dc=danetest,dc=se
	
	# Load modules for database type
	dn: cn=module,cn=config
	objectclass: olcModuleList
	cn: module
	olcModuleLoad: back_bdb.la

	# Create directory database
	dn: olcDatabase=bdb,cn=config
	objectClass: olcDatabaseConfig
	objectClass: olcBdbConfig
	olcDatabase: bdb
	
	# Domain name (e.g. home.local)
	olcSuffix: dc=danetest,dc=se
	
	# Location on system where database is stored
	olcDbDirectory: /var/lib/ldap
	
	# Manager of the database
	olcRootDN: cn=admin,dc=danetest,dc=se
	
	# LDAP password, chose a good one
	olcRootPW: test                                                           
	
	# Indices in database to speed up searches
	olcDbIndex: uid pres,eq
	olcDbIndex: cn,sn,mail pres,eq,approx,sub
	olcDbIndex: objectClass eq
	
	# Allow users to change their own password
	# Allow anonymous to authenciate against the password
	# Allow admin to change anyones password
	olcAccess: to attrs=userPassword
	  by self write
	  by anonymous auth
	  by dn.base="cn=admin,dc=danetest,dc=se" write
	  by * none
	
	# Allow users to change their own record
	# Allow anyone to read directory
	olcAccess: to *
	  by self write
	  by dn.base="cn=admin,dc=danetest,dc=se" write
	  by * read

### Add the database in ldap

	cd /etc/ldap/slapd.d
	sudo ldapadd -x -D cn=admin,dc=danetest,dc=se -W -f db.ldif

### Create ldif-file to add people (users) to it
sudo vim /etc/ldap/slapd.d/people.ldif 

	# Create top-level object in domain
	dn: dc=danetest,dc=se
	objectClass: top
	objectClass: dcObject
	objectclass: organization
	o: danetest.se
	dc: danetest
	description: Danetest network 

	dn: ou=People,dc=danetest,dc=se
	objectClass: organizationalUnit
	ou: People

	dn: ou=Groups,dc=danetest,dc=se
	objectClass: organizationalUnit
	ou: Groups

	dn: cn=miners,ou=Groups,dc=danetest,dc=se
	objectClass: posixGroup
	cn: miners
	gidNumber: 5000

	dn: uid=john,ou=People,dc=danetest,dc=se
	objectClass: inetOrgPerson
	objectClass: posixAccount
	objectClass: shadowAccount
	uid: john
	sn: Doe
	givenName: John
	cn: John Doe
	displayName: John Doe
	uidNumber: 10000
	gidNumber: 5000
	userPassword: password
	gecos: John Doe
	loginShell: /bin/bash
	homeDirectory: /home/john
	mail: john.doe@example.com


### Add the people (users) in the people.ldif-file to the database

	cd /etc/ldap/slapd.d
	sudo ldapadd -x -D cn=admin,dc=danetest,dc=se -W -f people.ldif

### LDAPSearch

	ldapsearch -x -H ldap://danetest.se -b dc=danetest,dc=se

	ldapsearch -x -b dc=danetest,dc=se

	ldapsearch -x -LLL -b dc=danetest, dc=se 'uid=john' cn gidNumber


### Add new user to the database, by creating a new tempfile, lets call it new_temp_file.ldif and add a user within the file
sudo vim /etc/ldap/slapd.d/new_temp_file.ldif 
	
	dn: uid=adam,ou=People,dc=danetest,dc=se
	objectClass: inetOrgPerson
	objectClass: posixAccount
	objectClass: shadowAccount
	uid: adam
	sn: Brown
	givenName: Adam
	cn: Adam Brown
	displayName: Adam Brown
	uidNumber: 10002
	gidNumber: 5000
	userPassword: adamldap
	gecos: Adam Brown
	loginShell: /bin/bash
	homeDirectory: /home/adam
	mail: adam.brown@adambrown.com

### Add the new user to the database 
	sudo ldapadd -x -D cn=admin,dc=danetest,dc=se -W -f new_temp_file.ldif 


### Can come in handy, maybe 

##### Setup LDAP Server
dpkg-reconfigure slapd

	Omit OpenLDAP server configuration? ... No
	DNS domain name: ... danetest.se
	Name of your organization: ... danetest.se.internal
	Admin Password: XXXXX
	Confirm Password: XXXXX
	OK
	BDB
	Do you want your database to be removed when slapd is purged? ... No
	Move old database? ... Yes
	Allow LDAPv2 Protocol? ... No

##### LDAPSearch for danetest.se
	ldapsearch -x -b dc=danetest,dc=se

##### Deleting LDAP and if ldap_add: Insufficient access (50) occur

	apt-get --purge remove slapd ldap-utils
	apt-get -y install slapd ldap-utils
	ls /etc/ldap/schema/*.ldif | xargs -I {} sudo -Y EXTERNAL -H ldapi:/// -f {}

##### Delete LDAP databases

	 1. sudo service slapd stop
	 2. sudo rm -f /var/lib/ldap/*
	 3. sudo service slapd start


##### Links
	http://ubuntuforums.org/showthread.php?p=8161118
	https://help.ubuntu.com/11.10/serverguide/C/openldap-server.html
	http://www.debuntu.org/ldap-server-and-linux-ldap-clients
	http://ubuntuforums.org/showthread.php?t=1313472&page=4

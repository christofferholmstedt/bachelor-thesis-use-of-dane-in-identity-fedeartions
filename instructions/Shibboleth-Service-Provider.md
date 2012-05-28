## Guide on how to install Shibbboleth Identity Provider on Ubuntu
### Some background
This guide is originally from Rickard Bellgrim (Certezza AB, Sweden) and released under Creative Commons Attribution-ShareAlike 3.0 Unported (CC BY-SA 3.0). For more information about the license please visit http://creativecommons.org/licenses/by-sa/3.0/

These instructions have been tested on a Virtual Machine of Ubuntu 12.04 LTS the 28th of May 2012.

### Requirements

 * Internet connectivity.
 * A {sub}domain to use for the Service Provider.

### Install Apache HTTP Server
	sudo apt-get install apache2

### Install Shibboleth SP
First step is to install all dependencies aswell as Shibboleth Service Provider software. At time of writing these instructions the versions mentioned below are the latest stable that we have used. Before downloading please visit respective page to confirm that those are the latest versions available.

    # Install build tools
	sudo apt-get install build-essential

    # Download source packages for all dependencies
    wget http://www.shibboleth.net/downloads/log4shib/latest/log4shib-1.0.4.tar.gz
	wget http://apache.dataphone.se/xerces/c/3/sources/xerces-c-3.1.1.tar.gz
	wget http://apache.dataphone.se/santuario/c-library/xml-security-c-1.6.1.tar.gz
	wget http://www.shibboleth.net/downloads/c++-opensaml/latest/xmltooling-1.4.2.tar.gz
	wget http://www.shibboleth.net/downloads/c++-opensaml/latest/opensaml-2.4.3.tar.gz
	wget http://www.shibboleth.net/downloads/service-provider/latest/shibboleth-sp-2.4.3.tar.gz

    # Unpackge
	tar -xzf log4shib-1.0.4.tar.gz
	tar -xzf xerces-c-3.1.1.tar.gz
	tar -xzf xml-security-c-1.6.1.tar.gz
	tar -xzf xmltooling-1.4.2.tar.gz
	tar -xzf opensaml-2.4.3.tar.gz
	tar -xzf shibboleth-sp-2.4.3.tar.gz
	
    # Install log4shib
    cd log4shib-1.0.4
	./configure --disable-static --disable-doxygen --prefix=/opt/shibboleth-sp
	make
	sudo make install

    # Install xerces
	cd ../xerces-c-3.1.1
	./configure --prefix=/opt/shibboleth-sp --disable-netaccessor-libcurl
	make
	sudo make install
	
    # Install XML Security
    cd ../xml-security-c-1.6.1
	sudo apt-get install libssl-dev pkg-config
	./configure --without-xalan --disable-static --prefix=/opt/shibboleth-sp --with-xerces=/opt/shibboleth-sp
	make
	sudo make install
	
    # Install XML Tooling
    cd ../xmltooling-1.4.2
	sudo apt-get install libcurl3 libcurl3-dev
	./configure --with-log4shib=/opt/shibboleth-sp --prefix=/opt/shibboleth-sp -C
	make
	sudo make install

    # Install OpenSAML
    cd ../opensaml-2.4.3
	./configure --with-log4shib=/opt/shibboleth-sp --prefix=/opt/shibboleth-sp -C
	make
	sudo make install
	
    # Install Shibboleth
    cd ../shibboleth-2.4.3
	sudo apt-get install apache2-threaded-dev
	./configure --with-log4shib=/opt/shibboleth-sp --prefix=/opt/shibboleth-sp
	make
	sudo make install

### Apache2

#### Create a self-signed certificate. TODO if you want CA signed: Send CSR below and change the crt. 

##### Create new directory to hold the keys within:
	sudo mkdir /etc/apache2/keys

##### Go to the new directory

cd /etc/apache2/keys
	
	* sudo openssl genrsa -des3 -out server.key 2048
	* sudo cp server.key server.key.orig
	* sudo openssl rsa -in server.key.orig -out server.key
	* sudo openssl req -out server.csr -key server.key -subj "/CN=federera.iis.se/OU=IT/O=Internet Infrastructure Foundation/L=Stockholm/ST=Stockholm/C=SE" -new
	* sudo openssl req -x509 -nodes -sha256 -key server.key -out server.crt -days 365 -subj "/CN=federera.iis.se/OU=IT/O=Internet Infrastructure Foundation/L=Stockholm/ST=Stockholm/C=SE" -new

##### Set the rights for the keys
	sudo chmod -R 640 /etc/apache2/keys/

<!---( ##### Edit the ports to listen to 
sudo vim /etc/apache2/ports.conf

	# Change to: 
	"Listen 192.168.238.11:80"
	
	# Change to: 
	"Listen 192.168.238.11:443"

	# Add:  
	"Listen [fc00::1]:80"
	
	# Add:  
	"Listen [fc00::1]:443"
)--->
##### Edit /opt/shibboleth-sp/etc/shibboleth/apache22.config
sudo vim /opt/shibboleth-sp/etc/shibboleth/apache22.config
	
	# Remove the tag:
	<Location /secure>

##### Edit /etc/apache2/httpd.conf
sudo vim /etc/apache2/httpd.conf

	# Add: 
	Include /opt/shibboleth-sp/etc/shibboleth/apache22.config

	<VirtualHost _default_:80>
		RewriteEngine on
		RewriteCond %{SERVER_PORT} !^443$
		RewriteRule ^.*$ https://%{SERVER_NAME}%{REQUEST_URI} [L,R]
	</VirtualHost>

##### Activate mudules
	sudo a2enmod rewrite ssl


### The webpage

##### Remove default webpage
	sudo rm /etc/apache2/sites-enabled/000-default

##### Create config file (danetest.se) for the webpage
	sudo vim /etc/apache2/sites-enabled/danetest.se

##### Add to danetest.se
	<VirtualHost *:443>
		ServerName sp1.danetest.se
		ServerAdmin hostmaster@danetest.se
		UseCanonicalName On

		DocumentRoot /var/www/sp1.danetest.se/

		<Directory />
			Options FollowSymLinks
			AllowOverride None
		</Directory>

		<Directory /var/www/sp1.danetest.se/>
			Options Indexes FollowSymLinks MultiViews
			AllowOverride None
			Order allow,deny
			allow from all
		</Directory>

		<Directory /var/www/sp1.danetest.se/sp1>
			Options +Indexes FollowSymLinks +ExecCGI
			DirectoryIndex printenv.cgi
			AddHandler cgi-script .cgi
		</Directory>

		# SSL.
		SSLEngine on
		SSLProtocol all -SSLv2
		SSLHonorCipherOrder on
		SSLCipherSuite ECDHE-RSA-AES256-SHA384:AES256-SHA256:RC4:HIGH:!MD5:!aNULL:!EDH:!AESGCM
		SSLCertificateFile      /etc/apache2/keys/server.crt
		SSLCertificateKeyFile   /etc/apache2/keys/server.key
		# Header add Strict-Transport-Security "max-age=7776000"

		<Location /sp1>
			AuthType           shibboleth
			ShibRequireSession On
			require            valid-user

			ShibRequestSetting applicationId default
			# TODO Kontrollera entityID nedanför
			ShibRequestSetting entityID https://idp1.danetest.se/idp/shibboleth
		</Location>
	</VirtualHost>

### Create testpage

##### Create directory to sp1.danetest.se 

	sudo mkdir /var/www/sp1.danetest.se

##### Create directory to sp1.danetest/sp1

	sudo mkdir /var/www/sp1.danetest/sp1

##### Create file printenv.cgi:

	sudo vim /var/www/sp1.danetest.se/sp1/printenv.cgi

##### Add to perl-sctipt to the file: 
	#!/usr/bin/perl

	my @entitlements = split(';', $ENV{'SAML_eduPersonEntitlement'});
	my $entitlementAA;
	my $entitlementIdP;
	foreach my $entitlement (@entitlements) {
	  if ($entitlement eq "http://skolfederation.se/NS/entitlement/accredited-teacher") {
	    $entitlementAA = $entitlement;
	  } else {
	    if (!$entitlementIdP) {
	      $entitlementIdP = $entitlement;
	    } else {
	      $entitlementIdP = join(';', $entitlementIdP, $entitlement);
	    }
	  }
	}

	print "Content-type: text/html; charset=UTF-8\n\n";
	print "<html>\n";
	print "  <head>\n";
	print "    <title>.SE Test Service Provider</title>\n";
	print "    <link rel=\"stylesheet\" type=\"text/css\" href=\"/page.css\"/>\n";
	print "  </head>\n";
	print "  <body id=\"homepage\">\n";
	print "    <div id=\"page\">\n";
	print "      <div id=\"header\">\n";
	print "        <a href=\"https://www.iis.se/en/\"><img src=\"/skolfed/logo.png\"\n";
	print "           width=\"100\" height=\"43\" alt=\".SE\" class=\"logo\" /></a>\n";
	print "      </div>\n";
	print "      <div id=\"content\">\n";
	print "        <h2>Successful login</h2><br/>Variables sent to the application:<br/><br/>\n";

	print "        <h3>Subject NameID</h3>\n";
	print "        <table>\n";
	&print_row("transientId", $ENV{'SAML_transientId'});
	&print_row("persistentId", $ENV{'SAML_persistentId'});
	print "        </table>\n";

	print "        <h3>Attributes</h3>\n";
	print "        <table>\n";
	print "          <tr>\n";
	print "            <th>Attribute</th>\n";
	print "            <th>Value</th>\n";
	print "          </tr>\n";
	&print_row("eduPersonPrincipalName", $ENV{'SAML_eduPersonPrincipalName'});
	&print_row("norEduPersonNIN", $ENV{'SAML_norEduPersonNIN'});
	&print_row("sn", $ENV{'SAML_sn'});
	&print_row("givenName", $ENV{'SAML_givenName'});
	&print_row("displayName", $ENV{'SAML_displayName'});
	&print_row("mail", $ENV{'SAML_mail'});
	&print_row("personalIdentityNumber", $ENV{'SAML_personalIdentityNumber'});
	&print_row("postalAddress", $ENV{'SAML_postalAddress'});
	&print_row("street", $ENV{'SAML_street'});
	&print_row("postOfficeBox", $ENV{'SAML_postOfficeBox'});
	&print_row("postalCode", $ENV{'SAML_postalCode'});
	&print_row("l", $ENV{'SAML_l'});
	&print_row("c", $ENV{'SAML_c'});
	&print_row("telephoneNumber", $ENV{'SAML_telephoneNumber'});
	&print_row("mobile", $ENV{'SAML_mobile'});
	&print_row("eduPersonOrgDN", $ENV{'SAML_eduPersonOrgDN'});
	&print_row("norEduOrgUniqueIdentifier", $ENV{'SAML_norEduOrgUniqueIdentifier'});
	&print_row("eduPersonOrgUnitDN", $ENV{'SAML_eduPersonOrgUnitDN'});
	&print_row("norEduOrgUnitUniqueIdentifier", $ENV{'SAML_norEduOrgUnitUniqueIdentifier'});
	&print_row("eduPersonEntitlement", $entitlementIdP);
	&print_row("eduPersonAffiliation", $ENV{'SAML_eduPersonAffiliation'});
	print "        </table>\n";

	print "        <h3>Attribute from test AA</h3>\n";
	print "        <table>\n";
	print "          <tr>\n";
	print "            <th>Attribute</th>\n";
	print "            <th>Value</th>\n";
	print "          </tr>\n";
	&print_row("eduPersonEntitlement", $entitlementAA);
	print "        </table>\n";

	print "        <h3>Information about the session</h3>\n";
	print "        <table>\n";
	print "          <tr>\n";
	print "            <th>Variable</th>\n";
	print "            <th>Value</th>\n";
	print "          </tr>\n";

	foreach $var (sort(keys(%ENV))) {
	  $val = $ENV{$var};
	  $val =~ s|\n|\\n|g;
	  $val =~ s|"|\\"|g;
	  if ($var =~ m/^(Shib_|REMOTE_)/){
	    &print_row(${var}, ${val});
	  }
	}

	print "        </table>\n";
	print "      </div>\n";
	print "      <div id=\"footer\">\n";
	print "        .SE - The Internet Infrastructure Foundation -\n";
	print "        <a href=\"https://www.iis.se/en/om-se/kontakt\">Addresses and contact information</a>\n";
	print "      </div>\n";
	print "    </div>\n";
	print "  </body>\n";
	print "</html>\n";

	sub print_row {
	  print "          <tr>\n";
	  print "            <td>$_[0]</td>\n";
	  print "            <td>$_[1]</td>\n";
	  print "          </tr>\n";
	}


##### Make the script runnable
	sudo chmod +x /var/www/skolfed/sp1/printenv.cgi	

### Create CSS for the webpage

##### Create page.css file
	sudo vim /var/www/page.css

##### Add to the page.css file
	html,body { margin: 0; padding: 0; font-size: 12px; }
	body { background-color: #eee; font-family: helvetica,arial,sans-serif; line-height: 18px; }
	#page { width: 930px; margin: 0 auto; }
	#header { position: relative; width: 870px; height: 73px; margin: 0 0 30px 0;
		  padding: 30px 30px 0 30px; background-color: #fff; }
	#content { display: inline; float: left; width: 900px; margin: 0 0 30px 0; padding: 15px;
		 background-color: #fff; }
	#footer { display: inline; float: left; width: 900px; padding: 15px;
		  background-color: #fff; font-size: 12px; }
	a:hover { text-decoration: underline; }
	a { outline: none; color: #0C8EAE; text-decoration: none; }
	table { border: 1px solid #ddd; padding: 5px; margin-bottom: 10px; border-collapse:collapse;
		border-right: 1px solid #999; border-bottom: 1px solid #999; }
	table th { font-size: 12px; padding: 5px; background-color: #eee; border-left: 1px solid #ccc; }
	table td { font-size: 12px; border-top: 1px solid #ddd; padding: 5px; padding-right: 10px;
		   border-left: 1px solid #ccc; margin: 0; }



### shibboleth2.xml

##### Open the shibboleth2.xml file
	sudo vim /opt/shibboleth-sp/etc/shibboleth/shibboleth2.xml
	
##### Edit the shibboleth2.xml file
	
 	<ApplicationDefaults 
                        id="default" 
                        entityID="https://sp1.danetest.se/sp1/shibboleth"
                        homeURL="https://sp1.danetest.se/sp1/"
                        REMOTE_USER="SAML_eduPersonPrincipalName SAML_persistentId">
	
 	<Sessions 
                lifetime="28800" 
                timeout="3600" 
                checkAddress="false" 
                cookieProps="; path=/; secure"
                relayState="cookie" 
                handlerSSL="true">
	
		<SSO 
                 discoveryProtocol="SAMLDS" discoveryURL="https://ds.danetest.se/">
              		SAML2 SAML1
            	</SSO>
        
	<Errors supportContact="christoffer.holmstedt@gmail.com"
            logoLocation="/shibboleth-sp/logo.jpg"
            styleSheet="/shibboleth-sp/main.css"/>
	
	<MetadataProvider 
                type="XML" 
                uri="http://md.danetest.se/md/danetestPretty.xml"
                backingFilePath="/opt/shibboleth-sp/var/xml/idp1-metadata.xml" 
                reloadInterval="7200"
                disregardSslCertificate="true">
        </MetadataProvider>
	



### attribute-map.xml

##### Open attribute-map.xml
	sudo vim /opt/shibboleth-sp/etc/shibboleth/attribute-map.xml
	
##### Edit attribute-map.xml
	<Attribute name="urn:mace:dir:attribute-def:mail" id="SAML_mail"/>
	<Attribute name="urn:oid:0.9.2342.19200300.100.1.3" id="SAML_mail"/>

	<Attribute name="urn:mace:dir:attribute-def:manager" id="SAML_manager"/>
	<Attribute name="urn:oid:0.9.2342.19200300.100.1.10" id="SAML_manager"/>

	<Attribute name="urn:mace:dir:attribute-def:mobile" id="SAML_mobile"/>
	<Attribute name="urn:oid:0.9.2342.19200300.100.1.41" id="SAML_mobile"/>

	<Attribute name="urn:oid:1.2.752.29.4.13" id="SAML_personalIdentityNumber"/>

	<Attribute name="urn:mace:dir:attribute-def:norEduPersonNIN" id="SAML_norEduPersonNIN"/>
	<Attribute name="urn:oid:1.3.6.1.4.1.2428.90.1.5" id="SAML_norEduPersonNIN"/>

	<Attribute name="urn:mace:dir:attribute-def:norEduOrgUniqueIdentifier" id="SAML_norEduOrgUniqueIdentifier"/>
	<Attribute name="urn:oid:1.3.6.1.4.1.2428.90.1.7" id="SAML_norEduOrgUniqueIdentifier"/>

	<Attribute name="urn:mace:dir:attribute-def:norEduOrgUnitUniqueIdentifier" id="SAML_norEduOrgUnitUniqueIdentifier"/>
	<Attribute name="urn:oid:1.3.6.1.4.1.2428.90.1.8" id="SAML_norEduOrgUnitUniqueIdentifier"/>

	<Attribute name="urn:mace:dir:attribute-def:eduPersonAffiliation" id="SAML_eduPersonAffiliation">
		<AttributeDecoder xsi:type="StringAttributeDecoder" caseSensitive="false"/>
	</Attribute>
	<Attribute name="urn:oid:1.3.6.1.4.1.5923.1.1.1.1" id="SAML_eduPersonAffiliation">
		<AttributeDecoder xsi:type="StringAttributeDecoder" caseSensitive="false"/>
	</Attribute>

	<Attribute name="urn:mace:dir:attribute-def:eduPersonNickname" id="SAML_eduPersonNickname"/>
	<Attribute name="urn:oid:1.3.6.1.4.1.5923.1.1.1.2" id="SAML_eduPersonNickname"/>

	<Attribute name="urn:mace:dir:attribute-def:eduPersonOrgDN" id="SAML_eduPersonOrgDN"/>
	<Attribute name="urn:oid:1.3.6.1.4.1.5923.1.1.1.3" id="SAML_eduPersonOrgDN"/>

	<Attribute name="urn:mace:dir:attribute-def:eduPersonOrgUnitDN" id="SAML_eduPersonOrgUnitDN"/>
	<Attribute name="urn:oid:1.3.6.1.4.1.5923.1.1.1.4" id="SAML_eduPersonOrgUnitDN"/>

	<Attribute name="urn:mace:dir:attribute-def:eduPersonPrimaryAffiliation" id="SAML_eduPersonPrimaryAffiliation">
		<AttributeDecoder xsi:type="StringAttributeDecoder" caseSensitive="false"/>
	</Attribute>
	<Attribute name="urn:oid:1.3.6.1.4.1.5923.1.1.1.5" id="SAML_eduPersonPrimaryAffiliation">
		<AttributeDecoder xsi:type="StringAttributeDecoder" caseSensitive="false"/>
	</Attribute>

	<Attribute name="urn:mace:dir:attribute-def:eduPersonPrincipalName" id="SAML_eduPersonPrincipalName">
		<AttributeDecoder xsi:type="ScopedAttributeDecoder"/>
	</Attribute>
	<Attribute name="urn:oid:1.3.6.1.4.1.5923.1.1.1.6" id="SAML_eduPersonPrincipalName">
		<AttributeDecoder xsi:type="ScopedAttributeDecoder"/>
	</Attribute>

	<Attribute name="urn:mace:dir:attribute-def:eduPersonEntitlement" id="SAML_eduPersonEntitlement"/>
	<Attribute name="urn:oid:1.3.6.1.4.1.5923.1.1.1.7" id="SAML_eduPersonEntitlement"/>

	<Attribute name="urn:mace:dir:attribute-def:eduPersonPrimaryOrgUnitDN" id="SAML_eduPersonPrimaryOrgUnitDN"/>
	<Attribute name="urn:oid:1.3.6.1.4.1.5923.1.1.1.8" id="SAML_eduPersonPrimaryOrgUnitDN"/>

	<Attribute name="urn:mace:dir:attribute-def:eduPersonScopedAffiliation" id="SAML_eduPersonScopedAffiliation">
		<AttributeDecoder xsi:type="ScopedAttributeDecoder" caseSensitive="false"/>
	</Attribute>
	<Attribute name="urn:oid:1.3.6.1.4.1.5923.1.1.1.9" id="SAML_eduPersonScopedAffiliation">
		<AttributeDecoder xsi:type="ScopedAttributeDecoder" caseSensitive="false"/>
	</Attribute>

	<Attribute name="urn:mace:shibboleth:1.0:nameIdentifier" id="SAML_transientId">
		<AttributeDecoder xsi:type="NameIDAttributeDecoder" formatter="$Name" defaultQualifiers="true"/>
	</Attribute>
	<Attribute name="urn:oasis:names:tc:SAML:2.0:nameid-format:transient" id="SAML_transientId">
		<AttributeDecoder xsi:type="NameIDAttributeDecoder" formatter="$Name" defaultQualifiers="true"/>
	</Attribute>

	<Attribute name="urn:oid:1.3.6.1.4.1.5923.1.1.1.10" id="SAML_persistentId">
		<AttributeDecoder xsi:type="NameIDAttributeDecoder" formatter="$Name" defaultQualifiers="true"/>
	</Attribute>
	<Attribute name="urn:oasis:names:tc:SAML:2.0:nameid-format:persistent" id="SAML_persistentId">
		<AttributeDecoder xsi:type="NameIDAttributeDecoder" formatter="$Name" defaultQualifiers="true"/>
	</Attribute>

	<Attribute name="urn:mace:dir:attribute-def:eduPersonAssurance" id="SAML_eduPersonAssurance"/>
	<Attribute name="urn:oid:1.3.6.1.4.1.5923.1.1.1.11" id="SAML_eduPersonAssurance"/>

	<Attribute name="urn:mace:dir:attribute-def:isMemberOf" id="SAML_isMemberOf"/>
	<Attribute name="urn:oid:1.3.6.1.4.1.5923.1.5.1.1" id="SAML_isMemberOf"/>

	<Attribute name="urn:mace:dir:attribute-def:eduCourseOffering" id="SAML_eduCourseOffering"/>
	<Attribute name="urn:oid:1.3.6.1.4.1.5923.1.6.1.1" id="SAML_eduCourseOffering"/>

	<Attribute name="urn:mace:dir:attribute-def:eduCourseMember" id="SAML_eduCourseMember"/>
	<Attribute name="urn:oid:1.3.6.1.4.1.5923.1.6.1.2" id="SAML_eduCourseMember"/>

	<Attribute name="urn:mace:dir:attribute-def:cn" id="SAML_cn"/>
	<Attribute name="urn:oid:2.5.4.3" id="SAML_cn"/>

	<Attribute name="urn:mace:dir:attribute-def:sn" id="SAML_sn"/>
	<Attribute name="urn:oid:2.5.4.4" id="SAML_sn"/>

	<Attribute name="urn:mace:dir:attribute-def:c" id="SAML_c"/>
	<Attribute name="urn:oid:2.5.4.6" id="SAML_c"/>

	<Attribute name="urn:mace:dir:attribute-def:l" id="SAML_l"/>
	<Attribute name="urn:oid:2.5.4.7" id="SAML_l"/>

	<Attribute name="urn:mace:dir:attribute-def:st" id="SAML_st"/>
	<Attribute name="urn:oid:2.5.4.8" id="SAML_st"/>

	<Attribute name="urn:mace:dir:attribute-def:street" id="SAML_street"/>
	<Attribute name="urn:oid:2.5.4.9" id="SAML_street"/>

	<Attribute name="urn:mace:dir:attribute-def:o" id="SAML_o"/>
	<Attribute name="urn:oid:2.5.4.10" id="SAML_o"/>

	<Attribute name="urn:mace:dir:attribute-def:ou" id="SAML_ou"/>
	<Attribute name="urn:oid:2.5.4.11" id="SAML_ou"/>

	<Attribute name="urn:mace:dir:attribute-def:title" id="SAML_title"/>
	<Attribute name="urn:oid:2.5.4.12" id="SAML_title"/>

	<Attribute name="urn:mace:dir:attribute-def:description" id="SAML_description"/>
	<Attribute name="urn:oid:2.5.4.13" id="SAML_description"/>

	<Attribute name="urn:mace:dir:attribute-def:businessCategory" id="SAML_businessCategory"/>
	<Attribute name="urn:oid:2.5.4.15" id="SAML_businessCategory"/>

	<Attribute name="urn:mace:dir:attribute-def:postalAddress" id="SAML_postalAddress"/>
	<Attribute name="urn:oid:2.5.4.16" id="SAML_postalAddress"/>

	<Attribute name="urn:mace:dir:attribute-def:postalCode" id="SAML_postalCode"/>
	<Attribute name="urn:oid:2.5.4.17" id="SAML_postalCode"/>

	<Attribute name="urn:mace:dir:attribute-def:postOfficeBox" id="SAML_postOfficeBox"/>
	<Attribute name="urn:oid:2.5.4.18" id="SAML_postOfficeBox"/>

	<Attribute name="urn:mace:dir:attribute-def:physicalDeliveryOfficeName" id="SAML_physicalDeliveryOfficeName"/>
	<Attribute name="urn:oid:2.5.4.19" id="SAML_physicalDeliveryOfficeName"/>

	<Attribute name="urn:mace:dir:attribute-def:telephoneNumber" id="SAML_telephoneNumber"/>
	<Attribute name="urn:oid:2.5.4.20" id="SAML_telephoneNumber"/>

	<Attribute name="urn:mace:dir:attribute-def:facsimileTelephoneNumber" id="SAML_facsimileTelephoneNumber"/>
	<Attribute name="urn:oid:2.5.4.23" id="SAML_facsimileTelephoneNumber"/>

	<Attribute name="urn:mace:dir:attribute-def:seeAlso" id="SAML_seeAlso"/>
	<Attribute name="urn:oid:2.5.4.34" id="SAML_seeAlso"/>

	<Attribute name="urn:mace:dir:attribute-def:givenName" id="SAML_givenName"/>
	<Attribute name="urn:oid:2.5.4.42" id="SAML_givenName"/>

	<Attribute name="urn:mace:dir:attribute-def:initials" id="SAML_initials"/>
	<Attribute name="urn:oid:2.5.4.43" id="SAML_initials"/>

	<Attribute name="urn:mace:dir:attribute-def:carLicense" id="SAML_carLicense"/>
	<Attribute name="urn:oid:2.16.840.1.113730.3.1.1" id="SAML_carLicense"/>

	<Attribute name="urn:mace:dir:attribute-def:departmentNumber" id="SAML_departmentNumber"/>
	<Attribute name="urn:oid:2.16.840.1.113730.3.1.2" id="SAML_departmentNumber"/>

	<Attribute name="urn:mace:dir:attribute-def:employeeNumber" id="SAML_employeeNumber"/>
	<Attribute name="urn:oid:2.16.840.1.113730.3.1.3" id="SAML_employeeNumber"/>

	<Attribute name="urn:mace:dir:attribute-def:employeeType" id="SAML_employeeType"/>
	<Attribute name="urn:oid:2.16.840.1.113730.3.1.4" id="SAML_employeeType"/>

	<Attribute name="urn:mace:dir:attribute-def:preferredLanguage" id="SAML_preferredLanguage"/>
	<Attribute name="urn:oid:2.16.840.1.113730.3.1.39" id="SAML_preferredLanguage"/>

	<Attribute name="urn:mace:dir:attribute-def:displayName" id="SAML_displayName"/>
	<Attribute name="urn:oid:2.16.840.1.113730.3.1.241" id="SAML_displayName"/>
	



### attribute-policy.xml

##### Open attribute-policy.xml
	sudo vim /opt/shibboleth-sp/etc/shibboleth/attribute-policy.xml
	
##### Set right names to the attributes
	
	# Remove:
	"affiliation"
	
	# Remove: 
	"unscoped-affiliation"
	
	# Remove:
	"primary-affiliation"
	
	# Change name to: 
	"SAML_eduPersonPrincipalName"
	
	# Remove:
	"targeted-id"
	
	# Change name to:
	"SAML_persistent"
	


### Start Shibboleth
	sudo /opt/shibboleth-sp/sbin/shibd -f &


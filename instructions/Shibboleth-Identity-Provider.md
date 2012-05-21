## Guide on how to install Shibbboleth Identity Provider on Ubuntu, demands internet access.



### Dependencies

##### Install dependencies: 
	sudo apt-get install unzip default-jre

##### Open file to set environment variables:  sudo vim /etc/profile 

	# Set the environment variables to:  
	export JAVA_HOME=/usr/lib/jvm/java-6-openjdk/jre/

##### Save JAVA_HOME as sudo-command by adding to file:  sudo vim /etc/sudoers  
	Defaults        env_keep += "JAVA_HOME"

###### Restart ther terminal to recieve new environment variables.



### The Shibboleth IdP software

##### Install wget: 
	sudo apt-get install wget

##### Fetch Shibboleth IdP zip:  
	wget http://www.shibboleth.net/downloads/identity-provider/2.3.5/shibboleth-identityprovider-2.3.6-bin.zip

##### Unzip the Shibboleth IdP:  
	sudo unzip -d /opt shibboleth-identityprovider-2.3.6-bin.zip

##### Change mode on the software:  
	sudo chmod -R 755 /opt/shibboleth-identityprovider-2.3.6

##### Create symbolic link so that /opt/shibboleth-identityprovider-2.3.6 is the same as  /opt/shibboleth-identityprovider: 
	sudo ln -s /opt/shibboleth-identityprovider-2.3.6 /opt/shibboleth-identityprovider




### Tomcat
##### Install tomcat6:  
	sudo apt-get install tomcat6

##### Copy tomcat6 library to shibboleth-indentityprovider/lib:  
	sudo cp /usr/share/tomcat6/lib/servlet-api.jar /opt/shibboleth-identityprovider/lib/




### Install Shibboleth IdP

	# Go to shibboleth-identityprovider catalog:  
	cd /opt/shibboleth-identityprovider

	# Run:  
	sudo env IdPCertLifetime=3 ./install.sh -Didp.home.input="/opt/shibboleth-idp" -Didp.hostname.input="idp1.danetest.se" -Didp.keystore.pass="123456"




### Tomcat continue

##### Remove default tomcat application:  
	sudo mv /var/lib/tomcat6/webapps/ROOT /opt/disabled.tomcat6.webapps.ROOT

##### Create directory to copy tomcat files to:  
	sudo mkdir /usr/share/tomcat6/endorsed

##### Copy files from the shibboleth-identityprovider to the new tomcat directory: 
	sudo cp /opt/shibboleth-identityprovider/endorsed/* /usr/share/tomcat6/endorsed/

##### Configure tomcat by adding to file:  sudo vim /etc/default/tomcat6 
	JAVA_OPTS="${JAVA_OPTS} -Djava.endorsed.dirs=/usr/share/tomcat6/endorsed"
	AUTHBIND=yes

##### Fetch dependency to /usr/share/tomcat6/lib/:  
	sudo wget -O /usr/share/tomcat6/lib/tomcat6-dta-ssl-1.0.0.jar http://shibboleth.internet2.edu/downloads/maven2/edu/internet2/middleware/security/tomcat6/tomcat6-dta-ssl/1.0.0/tomcat6-dta-ssl-1.0.0.jar




### Create SSL certficates
	# Run: 
	sudo keytool -genkey -keysize 2048 -keyalg RSA -alias tomcat -keystore /opt/shibboleth-idp/credentials/https.jks

	# CN = idp1.danetest.se, OU = IT, O = Exjobb, L = Stockholm, ST = Stockholm, C = SE





### Set rights
* sudo chown -R tomcat6 /opt/shibboleth-idp/metadata/
* sudo chown -R tomcat6 /opt/shibboleth-idp/logs/
* sudo chown -R tomcat6 /opt/shibboleth-idp/credentials/
* sudo chmod -R 640 /opt/shibboleth-idp/credentials/
* sudo chmod +x /opt/shibboleth-idp/credentials/




### Tomcat continue

##### Make sure tomcat starts the IdP service by adding to file: sudo vim /var/lib/tomcat6/conf/Catalina/localhost/idp.xml

	<Context docBase="/opt/shibboleth-idp/war/idp.war" 
		privileged="true" 
		antiResourceLocking="false" 
		antiJARLocking="false" 
		unpackWAR="false" 
		swallowOutput="true" /></code></pre>

##### Make sure tomcat listens to the rigt ports by commenting out other connectors and adding new to file:  
sudo vim /etc/tomcat6/server.xml  

	<Connector port="443" 
		protocol="org.apache.coyote.http11.Http11Protocol" 
		SSLEnabled="true" 
		maxThreads="150" 
		scheme="https"
		secure="true" 
		clientAuth="false" 
		sslProtocol="TLS" 
		keystoreFile="/opt/shibboleth-idp/credentials/https.jks" 
		keystorePass="123456" />

	<Connector port="8443" 	
		protocol="org.apache.coyote.http11.Http11Protocol" 
		SSLImplementation="edu.internet2.middleware.security.tomcat6.DelegateToApplicationJSSEImplementation"
		scheme="https" 
		SSLEnabled="true" 
		clientAuth="true" 
		keystoreFile="/opt/shibboleth-idp/credentials/idp.jks" 
		keystorePass="123456" />
	



### Login handler

##### Open and edit file: 
sudo vim /opt/shibboleth-idp/conf/handler.xml

	# Comment out the RemoteUser section:	
	<ph:LoginHandler xsi:type="ph:RemoteUser"> 
	
	# Comment in the UsernamePassword section:	
	<ph:LoginHandler xsi:type="ph:UsernamePassword">




### Relying party

##### Open and edit file:  
sudo vim /opt/shibboleth-idp/conf/relying-party.xml

	# Below   <!-- Relying Party Configuration --> edit/add so that it says 
	
	<rp:AnonymousRelyingParty 	provider="https://idp1.danetest.se/idp/shibboleth" 
					defaultSigningCredentialRef="IdPCredential"/>
	
	<rp:DefaultRelyingParty 	provider="https://idp1.danetest.se/idp/shibboleth" 
					defaultSigningCredentialRef="IdPCredential"
					nameIDFormatPrecedence="urn:oasis:names:tc:SAML:2.0:nameid-format:persistent urn:oasis:names:tc:SAML:2.0:nameid-format:transient" >

	# Change encryptAssertions="conditional" to "never" in all ProfileConfiguration that is between what is mentioned above and </rp:DefaultRelyingParty>

	# Example:
 
	<rp:ProfileConfiguration 	xsi:type="saml:SAML2ArtifactResolutionProfile"  
					signResponses="never" 
					signAssertions="always"  
					encryptAssertions="never" 
					encryptNameIds="never"/>

	# Below <!-- Metadata Configuration --> and above <!-- Security Configuration --> edit so that this is what is inbetween

	<metadata:MetadataProvider id="ShibbolethMetadata" xsi:type="metadata:ChainingMetadataProvider">
		
		<metadata:MetadataProvider id="URLMD" 
			xsi:type="metadata:FileBackedHTTPMetadataProvider" 
			metadataURL="http://md.danetest.se/md/danetest.xml" 
			backingFile="/opt/shibboleth-idp/metadata/md.danetest.se.xml"   
			disregardSslCertificate="true">
			
			<metadata:MetadataFilter xsi:type="metadata:ChainingFilter">
				<metadata:MetadataFilter xsi:type="metadata:RequiredValidUntil"  
							maxValidityInterval="P7D" />
				<metadata:MetadataFilter xsi:type="metadata:SignatureValidation"   
							trustEngineRef="shibboleth.MetadataTrustEngine"  
							requireSignedMetadata="true" />
			</metadata:MetadataFilter>		
		
		</metadata:MetadataProvider>    
	
	</metadata:MetadataProvider>

	# Below <! Security Configuration --> and  <!-- DO NOT EDIT BELOW THIS POINT --> edit/add so that it says

	<security:Credential id="IdPCredential" xsi:type="security:X509Filesystem">                                                           
		 <security:PrivateKey>/opt/shibboleth-idp/credentials/idp.key</security:PrivateKey>
		 <security:Certificate>/opt/shibboleth-idp/credentials/idp.crt</security:Certificate>
	</security:Credential>

	<security:TrustEngine id="shibboleth.MetadataTrustEngine" xsi:type="security:StaticExplicitKeySignature">
		<security:Credential id="DanetestFederationCredentials" xsi:type="security:X509Filesystem">
	       <security:Certificate>/opt/shibboleth-idp/credentials/danetest.crt</security:Certificate>
	       </security:Credential>
	</security:TrustEngine>




### Attribute filter

##### Open and edit file: 
sudo vim /opt/shibboleth-idp/conf/attribute-filter.xml

	# Below the attribute filer policy for transient ID delete all and add

	<!-- Release persistent ID to anyone -->
	<afp:AttributeFilterPolicy id="releasePersistentIdToAnyone">
		<afp:PolicyRequirementRule xsi:type="basic:ANY"/>
	   <afp:AttributeRule attributeID="persistentId">
		<afp:PermitValueRule xsi:type="basic:ANY"/>
	   </afp:AttributeRule>
	</afp:AttributeFilterPolicy>

	<!-- Danetest Policy requirement rule that indicates this policy can be used for request from any sp-->
	<afp:AttributeFilterPolicy id="danetest"> <afp:PolicyRequirementRule xsi:type="basic:ANY"/>   

	<!-- Attributes -->
	<afp:AttributeRule attributeID="mail">
		<afp:PermitValueRule xsi:type="basic:ANY" />
	</afp:AttributeRule>

	<afp:AttributeRule attributeID="postalAddress">
		<afp:PermitValueRule xsi:type="basic:ANY" />
	</afp:AttributeRule>

	<afp:AttributeRule attributeID="street">
		<afp:PermitValueRule xsi:type="basic:ANY" />
	</afp:AttributeRule>

	<afp:AttributeRule attributeID="postOfficeBox">
		<afp:PermitValueRule xsi:type="basic:ANY" />
	</afp:AttributeRule>

	<afp:AttributeRule attributeID="postalCode">
		<afp:PermitValueRule xsi:type="basic:ANY" />
	</afp:AttributeRule>

	<afp:AttributeRule attributeID="l">
		<afp:PermitValueRule xsi:type="basic:ANY" />
	</afp:AttributeRule>

	<afp:AttributeRule attributeID="c">
		<afp:PermitValueRule xsi:type="basic:ANY" />
	</afp:AttributeRule>

	<afp:AttributeRule attributeID="norEduOrgUnitUniqueIdentifier">
		<afp:PermitValueRule xsi:type="basic:ANY" />
	</afp:AttributeRule>

	afp:AttributeRule attributeID="SAML_street">
		<afp:PermitValueRule xsi:type="basic:ANY" />
	</afp:AttributeRule>

	afp:AttributeRule attributeID="SAML_mail">
		<afp:PermitValueRule xsi:type="basic:ANY" />
	</afp:AttributeRule>



### Attribute resolver

##### Open and edit file: 
sudo pico /opt/shibboleth-idp/conf/attribute-resolver.xml

	# Delete and add between Attribute definition and Data connectors
	
	<resolver:DataConnector id="staticAttributes" xsi:type="dc:Static">
		<dc:Attribute id="postalAddress">
			<dc:Value>.SE Box 7399 SE-10391 Stockholm Sweden</dc:Value>
		</dc:Attribute> 

		<dc:Attribute id="street">
			<dc:Value>just ordinary street</dc:Value>
		</dc:Attribute> 

		<dc:Attribute id="SAML_street">
			<dc:Value>my saml street</dc:Value>
		</dc:Attribute>         
		
		<dc:Attribute id="postOfficeBox">
			<dc:Value>Box 7399</dc:Value>
		</dc:Attribute> 
		 
		<dc:Attribute id="postalCode">
			<dc:Value>10391</dc:Value>         
		</dc:Attribute> 

		<dc:Attribute id="c">
			<dc:Value>SE</dc:Value>
		</dc:Attribute> 

		<dc:Attribute id="l">
			<dc:Value>ELL ELLE ELLE</dc:Value>         
		</dc:Attribute> 
		 
		<dc:Attribute id="norEduOrgUniqueIdentifier">             
			<dc:Value>802405-0190</dc:Value>         
		</dc:Attribute>         
		 
		<dc:Attribute id="mail">             
			<dc:Value>mail attribute @danetest.se</dc:Value>         
		</dc:Attribute>          
		 
		<dc:Attribute id="SAML_mail">     
			<dc:Value>SAML_mail attribute @danetest.se</dc:Value>         
		</dc:Attribute> 
	</resolver:DataConnector>

	<!-- Name Identifier related attributes -->
     	<resolver:AttributeDefinition id="transientId" xsi:type="ad:TransientId">
        	<resolver:AttributeEncoder xsi:type="enc:SAML1StringNameIdentifier" nameFormat="urn:mace:shibboleth:1.0:nameIdentifier"/>
         	<resolver:AttributeEncoder xsi:type="enc:SAML2StringNameID" nameFormat="urn:oasis:names:tc:SAML:2.0:nameid-format:transient"/>
     	</resolver:AttributeDefinition> 



	
### Authenticate User

##### Open and edit so that ShibUserAuth is as below in file: 
sudo vim /opt/shibboleth-idp/conf/login.config

		ShibUserPassAuth {
        		edu.vt.middleware.ldap.jaas.LdapLoginModule required
			host="danetest.se"
			port="389"
			base="dc=danetest,dc=se"
			ssl="false"
			userField="uid"
			serviceUser="cn=admin,dc=danetest,dc=se"
			serviceCredential="test"                                                                                                           
			subtreeSearch="true";
		};

##### Logging can be found here
/var/lib/tomcat6/logs and /opt/shibboleth-idp/logs

##### Restart tomcat
* sudo service tomcat6 stop
* sudo service tomcat6 start

##### Re-run install script for IdP (answer no on configuration question)
* cd /opt/shibboleth-identityprovider
* sudo ./install.sh -Didp.home.input="/opt/shibboleth-idp"

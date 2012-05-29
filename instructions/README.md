Overall structure of these instructions
=======================================
The instructions available in this directory is for you to be able to follow our steps on setting up the test environment we have used in our project.
This is not a complete guide on how to get Shibboleth and the other software used fully operational.
Therefore we prefer to give links to the official documentation.

#### What's the purpose with each software?

##### Bind9, OpenDNSSEC, SoftHSM
To be able to use "DANE" in its current form DNS and DNSSEC are prerequisites.
Bind9 is for basic DNS functionality.
OpenDNSSEC is to sign the unsigned zone-files.
SoftHSM is used by OpenDNSSEC to store the keys used for signing.

http://www.isc.org/software/bind (Bind)
http://www.opendnssec.org/ (OpenDNSSEC and SoftHSM)

##### Federation Operator (Metadata aggregator)
It's possible to set up a test environment without the federation operator but for us we wanted to simulate how it works in most federations here in Sweden.
The common approach is with a federation operator that collects the metadata from all entities, signes it and publish it.

To be able to parse and sign XML files some scripts were used. More about that in the "federationOperator-MetadataAggregator" guide.

##### Shibboleth Identity Provider
Self-explanatory, to set up an identityfederation you need at least one IdP.

https://wiki.shibboleth.net/confluence/display/SHIB2/Home

##### OpenLDAP
Used as credential storage by the Shibboleth Identity Provider software.

http://www.openldap.org/

##### Shibboleth Service Provider
Self-explanatory, to set up an identityfederation you need at least one SP.

https://wiki.shibboleth.net/confluence/display/SHIB2/Home

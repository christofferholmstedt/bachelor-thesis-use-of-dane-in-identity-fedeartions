Federation Operator or Metadata Aggregator
==========================================
The role of the federation operator is somewhat limited but it has one very important technical role, to fetch and sign the metadata of all entities within the federation. As a second step also publish the signed aggregated metadata.

Publishing the metadata file is often done over HTTP webserver of some kind.
The tricky part is to sign the XML files with suitable time inbetween.

### Signing of the XML files
We got help from Leif Johansson (SUNET) with this one and linked us to the scripts used by the identity federation "skolfederation.se" in Sweden.

http://registry.skolfederation.se/?p=skolfederation-metadata.git;a=tree

The signing part is within the "Makefile".
When we got the signing working we put it in a cronjob to do a new signing procedure every two hours.

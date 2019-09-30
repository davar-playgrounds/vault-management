# Latest Version
2.7-00006

# Version History

## Version 2.7-00006 (devon-thyne)
### Enhancement - ssh_development accessible to all ldap users by default
* Edited REDACTED-default policy to expose access to ssh_development secrets engine to all ldap users by default

### Bug fix - root user environment changes not taking effect during vault restoration
* Moved logic to modify the root user environment to initialization task that will not be skipped during a restoration of Vault from an existing consul snapshot stored in AWS S3

### Revision - changed consul s3 backup file naming conventions
* Moved to better convention for naming consul snapshot that properly order alphabetically and chronologically at the same time when listed in a file viewer

## Version 2.7-00005 (devon-thyne)
### Enhancement - Vault consul snapshot restore in place
* Ability to live restore an active Vault in place from an existing consul snapshot stored in stored in AWS S3 bucket

## Version 2.7-00004 (devon-thyne)
### Enhancement - Vault consul backup to S3
* Created hourly cron job on vault server to snapshot the consul backend and back it up to S3
* Passable parameter for s3 key to remote consul snapshot to automatically restore newly created vault from

### Enhancement - Install from newest vault and consul version
* Uses hardcoded download links to newest versions of vault (v1.2.1) and consul (1.5.3) at the time

### Enhancement - Overhaul role task stucture
* Broke role out into more module tasks that can more easily be called from main task without having to use large ansible blocks within each task

## Version 2.7-00003 (devon-thyne)
### Feature - Configure Vault to use https
* Correctly self signs x509 SSL certificate to use for https communication

## Version 2.7-00002 (devon-thyne)
### Feature - Adding Vault Configuration Tasks
* Enables vault initialization with a single key
* Enables the configuration of vault post-installation for items such as local users, kv store, vault policy, aws auth, bashrc, vault cli autocomplete

## Version 2.7-00001
### Initial version established with the following features
* Role enables operators to prepare install a single instance of Hashicorp Vault
* Non-persistent backend

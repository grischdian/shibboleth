# ADFS and ownCloud - Shibboleth and Shibboleth-DS
This Repo contains everything you need to install and configure Shibboleth for SSO with ADFS.
ownCloud is the protected Application

## Prerequisits
- Install and configure apache with SSL 

## More information
[Shibboleth Docu](https://wiki.shibboleth.net/confluence/display/SP3/)
[Shibboleth-DS Docu](https://wiki.shibboleth.net/confluence/display/EDS10/)
[ownCloud Shibboleth](https://doc.owncloud.com/server/admin_manual/enterprise/user_management/)

## Shibboleth SSO

### Debugging

```
occ shibboleth:mode ssoonly #Will activate SSO in ownCloud
occ shibboleth:mode notactive #Will deactivate SSO in ownCloud

```

### Create Shibboleth Repo

```
[shibboleth]
name=Shibboleth (CentOS_7)
# Please report any problems to https://issues.shibboleth.net
type=rpm-md
mirrorlist=https://shibboleth.net/cgi-bin/mirrorlist.cgi/CentOS_7
gpgcheck=1
gpgkey=https://shibboleth.net/downloads/service-provider/RPMS/repomd.xml.key
        https://shibboleth.net/downloads/service-provider/RPMS/cantor.repomd.xml.key
enabled=1
```
### Install Shibboleth

```
yum install shibboleth
```

### Configuration
#### Create new Encryption Keys
```
cd /etc/shibboleth
./keygen.sh
```
#### Download ADFS Metadata
For ownCloud you need to filter ADFS Metadata

```
php apps/user_shibboleth/tools/adfs2fed.php \
   https://<ADFS server FQDN>/FederationMetadata/2007-06/FederationMetadata.xml \
   <AD-Domain> > /etc/shibboleth/filtered-metadata.xml
```
#### Configure shibd
In /etc/shibboleth/shibboleth2.xml look for the following Sections
```
<ApplicationDefaults entityID="https://<owncloud server FQDN>/login/saml" REMOTE_USER="eppn upn">

<!-- <ADFS server FQDN>/<URI>/ is at the top of filtered-metadata.xml -->
<SSO entityID="https://<ADFS server FQDN>/<URI>/">
  SAML2
</SSO>

<MetadataProvider type="XML" file="/etc/shibboleth/filtered-metadata.xml"/>
```
In /etc/shibboleth/attribute-map.xml
```
<Attributes xmlns="urn:mace:shibboleth:2.0:attribute-map" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
   <Attribute name="http://schemas.xmlsoap.org/ws/2005/05/identity/claims/upn" id="upn"/>
</Attributes>
```

#### Configure ADFS
##### Add Relying Party Trust
![](https://github.com/grischdian/shibboleth/raw/master/screenshots/1.png)
![](https://github.com/grischdian/shibboleth/raw/master/screenshots/2.png)
![](https://github.com/grischdian/shibboleth/raw/master/screenshots/3.png)
![](https://github.com/grischdian/shibboleth/raw/master/screenshots/4.png)
![](https://github.com/grischdian/shibboleth/raw/master/screenshots/5.png)
![](https://github.com/grischdian/shibboleth/raw/master/screenshots/6.png)
![](https://github.com/grischdian/shibboleth/raw/master/screenshots/7.png)
##### Edit Claim Issuance Policy
![](https://github.com/grischdian/shibboleth/raw/master/screenshots/8.png)
##### Add Rule ...
![](https://github.com/grischdian/shibboleth/raw/master/screenshots/9.png)
![](https://github.com/grischdian/shibboleth/raw/master/screenshots/10.png)
##### Set Useridentifier to UPN (like in <ApplicationDefaults> in shibboleth2.xml)
![](https://github.com/grischdian/shibboleth/raw/master/screenshots/11.png)

#### Configure Apache
In /etc/httpd/conf.d/shib.conf
```
<Location />
    AuthType shibboleth
    ShibRequestSetting requireSession false
    Require shibboleth
</Location>

<Location ~ "^(/index.php)?/login">
  AuthType shibboleth
  ShibRequestSetting requireSession true
  require valid-user
</Location>

<Location ~ "/.*\.(css|js|woff)">
    AuthType None
    Require all granted
</Location>
```
### Restart apache and shibd
### Done

## Shibboleth Discovery Services
This is needed if you have more then one ADFS

### Test normal Shibboleth Config with both ADFS Servers to ensure Everything is fine

### Install Shibboleth-DS

```
yum install shibboleth-embedded-ds
```

### Configure Shibboleth-DS
```
mkdir /var/www/html/owncloud/shibboleth-ds
cd /etc/shibboleth-ds
cp *.js *.css *.html *.gif /var/www/html/owncloud/shibboleth-ds/
cp shibboleth-ds.conf /etc/httpd/conf.d/shibboleth-ds.conf
```
### Configure Apache
In /etc/httpd/conf.d/shibboleth-ds.conf adjust path of Aliase
```
<IfModule mod_alias.c>
  <Location /shibboleth-ds>
    <IfVersion >= 2.4>
      Require all granted
    </IfVersion>
    <IfVersion < 2.4>
      Allow from all
    </IfVersion>
    <IfModule mod_shib.c>
      AuthType shibboleth
      ShibRequestSetting requireSession false
      require shibboleth
    </IfModule>
  </Location>
  Alias /shibboleth-ds/idpselect_config.js /var/www/html/owncloud/shibboleth-ds/idpselect_config.js
  Alias /shibboleth-ds/idpselect.js /var/www/html/owncloud/shibboleth-ds/idpselect.js
  Alias /shibboleth-ds/idpselect.css /var/www/html/owncloud/shibboleth-ds/idpselect.css
  Alias /shibboleth-ds/index.html /var/www/html/owncloud/shibboleth-ds/index.html
  Alias /shibboleth-ds/blank.gif /var/www/html/owncloud/shibboleth-ds/blank.gif
</IfModule>
```
### Configure Shibboleth-DS
In /var/www/html/owncloud/shibboleth-ds/idpselect_config.js
```
    this.returnWhiteList = [ "^https:\/\/owncloud.grischdian\.de.*$" ];
```
### Configure Shibboleth
In /etc/shibboleth/shibboleth2.xml change to
```
<SSO discoveryProtocol="SAMLDS" discoveryURL="https://owncloud.grischdian.de/shibboleth-ds">
   SAML2 SAML1
</SSO>
<!-- Add a MetadataProvider for each ADFS Server -->
<MetadataProvider type="XML" lagacyOrgNames="true" reloadInterval="7200" path="/etc/shibboleth/filtered-metadata.xml"/>
```
### Restart apache and shibd
### Done



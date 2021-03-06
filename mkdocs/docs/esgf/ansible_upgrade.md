ESGF-Ansible node upgrade
=========================

* Version: 0.0.3
* Date: 15/07/2019
* Authors: Sébastien Gardoll
* Keywords: esgf ansible upgrade node

## Description

!!! note
    This procedure describes how to upgrade ESGF-Ansible nodes from the version 4.0.2

* ESGF-Ansible nodes version: 4.0.2 and higher
* License: as ESFG
* Repository: [github](https://github.com/ESGF/esgf-ansible)
* Documentation: [ESGF-Ansible wiki](https://esgf.github.io/esgf-ansible/)

## Hosts & access

### Hosts

Any ESGF nodes (idp, index and data).

### Access

!!! warning
    This procedure assumes that you can establish ssh connexions (PKI) to your ESGF 4.0.x nodes.

* Protocol: ssh
* Permission: superuser
* Login: root

## Environment

### Requirements

* An already deployed ESGF node 4.0.x or higher
* An already deployed ESFG-Ansible repository (local or remote)

### Dependencies

For running ESGF-Ansible:

* Python = 3.6 and higher
* Ansible = 2.7.8 and higher

## Procedure

### Settings

!!! note
    Custom and run these commands every time you execute the code of the procedure, from the machine which will run esgf-ansible. At the time of this writing, esgf-ansible is only installed on esgf-monitoring at ipsl.
    You must set the TAG_VERSION value at least.


```bash hl_lines="3" 
export GROUP='dev' # The group of idp/index and data nodes (e.g.: dev nodes).
export PARENT_DIR="${HOME}" # The parent directory that contains ESGF-Ansible repository.
export TAG_VERSION='XXX' # Checkout the new version of ESGF-Ansible to be upgraded to.
```

### Node preparation

#### Stop the all nodes:

Run the following command on the machine where the ESGF-Ansible repository lives.

```bash
# The above command is only available on 
# esgf-watch-dog@esgf-monitoring.ipsl.upmc.fr (a VM at IPSL).
esgf ${GROUP} stop # This stops idp, index and data nodes of the specified group.
```

#### Make a snapshot of the VM that will receive the upgrade

This is fairly self-explanatory, as if the upgrade failed for some reason you can get back to the last snapshot.

At IPSL, this is made through the vSphere interface.

#### Data node

!!! note
    Run these commands on esgf-watch-dog@esgf-monitoring.ipsl.upmc.fr, according to the node that you want to upgrade.

```bash
# Cleaning commands:
rm -fr /tmp/esgf-config # Mandatory only for upgrade to version 4.0.3 .
rm -fr /usr/local/src
rm -fr /etc/tempcerts
rm -f /var/lib/pgsql/data/pg_hba.conf.bak
rm -fr /esg/config.bak
rm -fr /etc/certs.bak /etc/esgfcerts.bak /etc/grid-security.bak

# Certificates backup commands:
cp -rp /etc/certs /etc/certs.bak
cp -rp /etc/esgfcerts /etc/esgfcerts.bak
cp -rp /etc/grid-security /etc/grid-security.bak

# PostgreSQL connexion configuration backup:
cp -p /var/lib/pgsql/data/pg_hba.conf /var/lib/pgsql/data/pg_hba.conf.bak

# ESGF configuration backup: 
cp -rp /esg/config /esg/config.bak
```

#### Index/IDP node

```bash
# Cleaning commands:
rm -fr /tmp/esgf-config # Mandatory only for upgrade to version 4.0.3 .
rm -fr /usr/local/src
rm -fr /etc/tempcerts
rm -fr /esg/config.bak
rm -fr /etc/certs.bak /etc/esgfcerts.bak /etc/grid-security.bak /var/lib/globus/simple_ca.bak

# Certificates backup commands:
cp -rp /etc/certs /etc/certs.bak
cp -rp /etc/esgfcerts /etc/esgfcerts.bak
cp -rp /etc/grid-security /etc/grid-security.bak
cp -rp /var/lib/globus/simple_ca /var/lib/globus/simple_ca.bak

# ESGF configuration backup: 
cp -rp /esg/config /esg/config.bak
```

### Disable notifications from the nagios administration interface

!!! warning
    This step is optional

If you have configured nagios alerts to warn you when a node is down or certain services are down, you will receive notifications during the upgrade because the node has to be shut down to upgrade. This can be prevented by disabling nagios notifications.

Log into the nagios administration : https://nagios-ng.ipsl.upmc.fr/nagios/

Then you can disable the tests such as : HTTP SSL certificate, HTTPS service, HTTP service.

### Set globus_user and globus_pass

At IPSL, GridFTP isn’t used on the ESGF machines anymore : it takes a lot of space and uses pretty much the entirety of the CPU. Hence, it’s on a separated, physical machine (not a virtual machine). This means that the globus services are not mandatory and should normally not be installed.

The ESGF-Ansible doc says that if you disable the GridFTP steps at installation (like it’s done here), you don’t need to set values to the variables globus_user and globus_pass. This is a lie.

However, you don’t need to assign real values, and for security reasons it’s recommended to set a bogus value instead :

```bash
cd ${PARENT_DIR}/esgf-ansible/host_vars/
nano <host variables file for the host you want>
```

Then uncomment these lines and add any string you want at the end :

```bash
#globus_user:
#globus_pass:
```

Then save and quit.

### Upgrade

Run these commands from the machine that holds your ESGF-Ansible repository.

!!! warning
    add --limit "data, index" when upgrading ESGF CDS nodes

```bash
script ${PARENT_DIR}/esgf-ansible/upgrade.log # Keep a log of the upgrade.
cd ${PARENT_DIR}/esgf-ansible
git checkout ${TAG_VERSION}
git status
source activate ansible # See ESGF-Ansible installation.
export ANSIBLE_NOCOLOR=true # Make the log readable.

# Change the path of the directory that contains your hosts inventory file.
INVENTORY="${PARENT_DIR}/scripts/esgf-ansible/hosts.${GROUP}"

ansible-playbook -i ${INVENTORY} -u root --skip-tags gridftp install.yml
```

### Post installation

Upgrading may revert some features of your node. Follow this procedure according to the features that have been reverted.

#### Robots exclusion standard

Follow this procedure if the Apache web site configuration of your node has been reverted and lost the robots exclusion standard.

* Create the robots.txt file, **once for all**

```bash
cat > "/var/www/html/robots.txt" <<EOF
User-agent: *
Disallow: /
EOF
```

* Modify the Apache configuration file

Open /etc/httpd/conf/httpd.ssl.conf and add the highlighted instruction at line 14:

```apache hl_lines="5"
...
<VirtualHost *:443>
        SSLEngine on
        SSLProxyEngine On
        Alias /robots.txt /var/www/html/robots.txt
...
```

* Reload the Apache configuration

```bash
service httpd reload
```

#### Data node root web page redirection

The root page of a **data node** web site usually points to the generic 'unkown service' page (code 503). This procedure describes how to redirect the root page to the thredds page.

* esgf-httpd-local.conf et esgf-httpd-local**s**.conf must exist in /etc/httpd/conf (see Annex).
* Decomment the following lines:
    - line 196 in /etc/httpd/conf/httpd.conf: `Include /etc/httpd/conf/esgf-httpd-local.conf`
    - line 33 in /etc/httpd/conf/httpd.**ssl**.conf: `Include /etc/httpd/conf/esgf-httpd-locals.conf`
* Comment the lines 65 et 66 in /etc/httpd/conf/httpd.ssl.conf:

```apache
ProxyPass / http://localhost:8889/
ProxyPassReverse / http://localhost:8889/
```

* Reload the Apache configuration

```bash
service httpd reload
```

* Test

The page http or https://FQN/ should redirect to http or https://FQN/thredds/catalog/catalog.html

#### Annex

* Content of the file esgf-httpd-local.conf (modify the highlighted line):

```apache hl_lines="1"
RedirectMatch ^/$ http://FQN/thredds/catalog/catalog.html

<LocationMatch ".*\/fileServer\/.*\.(?:png|jpg|jpeg|gif|txt|html|css|js|card)$">
  Header set Content-Disposition inline
</LocationMatch>

<LocationMatch ".*\/fileServer\/[^.]+$">
  Header set Content-Disposition inline
  Header set Content-Type text
</LocationMatch>

<LocationMatch ".*\/fileServer\/.*\.(?:png|jpg|jpeg|gif)$">
  Header set Content-Type image
</LocationMatch>

<LocationMatch ".*\/fileServer\/.*\.js$">
  Header set Content-Type text/javascript
</LocationMatch>

<LocationMatch ".*\/fileServer\/.*\.css$">
  Header set Content-Type text/css
</LocationMatch>

<LocationMatch ".*\/fileServer\/.*\.card$">
  Header set Content-Type text
</LocationMatch>

RewriteCond %{QUERY_STRING} ^[^.]+$|\.(?!(nc|grib)$)([^.]+$)
RewriteCond %{QUERY_STRING} ^dataset=.*\/(.*)$
RewriteRule ^(.*)\/catalog\/(.*)\/catalog.html$ $1\/fileServer\/$2\/%1? [R,L]
```

* Content of the file esgf-httpd-local**s**.conf (modify the highlighted line):

```apache hl_lines="1"
RedirectMatch ^/$ https://FQN/thredds/catalog/catalog.html

<LocationMatch ".*\/fileServer\/.*\.(?:png|jpg|jpeg|gif|txt|html|css|js|card)$">
  Header set Content-Disposition inline
</LocationMatch>

<LocationMatch ".*\/fileServer\/[^.]+$">
  Header set Content-Disposition inline
  Header set Content-Type text
</LocationMatch>

<LocationMatch ".*\/fileServer\/.*\.(?:png|jpg|jpeg|gif)$">
  Header set Content-Type image
</LocationMatch>

<LocationMatch ".*\/fileServer\/.*\.js$">
  Header set Content-Type text/javascript
</LocationMatch>

<LocationMatch ".*\/fileServer\/.*\.css$">
  Header set Content-Type text/css
</LocationMatch>

<LocationMatch ".*\/fileServer\/.*\.card$">
  Header set Content-Type text
</LocationMatch>

RewriteCond %{QUERY_STRING} ^[^.]+$|\.(?!(nc|grib)$)([^.]+$)
RewriteCond %{QUERY_STRING} ^dataset=.*\/(.*)$
RewriteRule ^(.*)\/catalog\/(.*)\/catalog.html$ $1\/fileServer\/$2\/%1? [R,L]
```

### Login

```bash
ssh esgf-watch-dog@esgf-monitoring.ipsl.upmc.fr
```

### Deployment/Install

See the ESGF-Ansible installation [page](ansible_installation.md).

### Configuration

The configuration of the ESGF-Ansible node is out of scope. See ESGF-Ansible documentation [here](https://esgf.github.io/esgf-ansible/usage/usage.html#quick-configuration).

### Tests

See ESGF-test-suite page. Running the status recipe of ESGF-Ansible is also a good starting point.

### Monitoring

Run status recipe of ESGF-Ansible.

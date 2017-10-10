
# Manual Gluu Server Clustering

## Introduction
If you have requirements for high availability (HA) or failover, you can configure your Gluu Server for multi-master replication by following the instructions below.

My server configurations are like so:

```
45.55.232.15    c4.gluu.org (NGINX server)
159.203.126.10  c5.gluu.org (Gluu 3.1.1 server/Ubuntu 14)
138.197.65.243  c6.gluu.org (Gluu 3.1.1 server/Ubuntu 14)
```

## Prerequisites

Some prerequisites are necessary for setting up Gluu with delta-syncrepl MMR:

- A minimum of three (3) servers or VMs--two (2) for Gluu Servers and one (1) for load balancing (in our example, NGINX);
- To create the following instructions we used Ubuntu 14 Trusty, but the process should not be OS specific;
- To create the following instructions we used an Nginx load balancer/proxy, however if you have your own load balancer, like F5 or Cisco, you should use that instead and disregard the bottom instructions about configuring Nginx.
- Gluu Server 3.x using OpenLDAP.

## Concept

Multi-master replication with OpenLDAP through delta-syncrepl by creating an accesslog database and configuring synchronization by means the slapd.conf file. The ldap.conf file for all the servers will allow the self-signed certs that Gluu creates and configuring the symas-openldap.conf to allow external connections for LDAP. There are also some additional steps that are required to persist Gluu functionality across servers. This is where a load-balancer/proxy is required.

## Instructions

### 1. [Install Gluu](https://gluu.org/docs/ce/3.1.0/installation-guide/install/) on one of the servers making sure to use a separate NGINX server FQDN as hostname.

- A separate NGINX server is necessary, since replicating a Gluu server to a different hostname breaks the functionality of the Gluu web page, when using a hostname other than what is in the certificates. For example, if I used c1.gluu.info as my host and copied that to a second server (e.g. c2.gluu.info), the process of accessing the site on c2.gluu.info, even with replication, will fail authentication, due to hostname conflict. So if c1 failed, you couldn't access the Gluu web GUI anymore.

- Now for the rest of the servers in the cluster, [Download the Gluu packages](https://gluu.org/docs/ce/3.1.0/installation-guide/install/), but don't run `setup.py` yet.

- We want to copy the `/install/community-edition-setu/setup.properties.last` file from this first install to the other servers as `setup.properties` so we have the exact same configurations. (Here I have ssh access to my other server outisde the Gluu chroot)

```

scp /opt/gluu-server-3.1.1/install/community-edition-setup/setup.properties.last root@c6.gluu.org:/opt/gluu-server-3.1.1/install/community-edition-setup/setup.properties

```

- Once you have the `setup.properties` file in place on the **other** server, run setup.py inside the Gluu chroot:

```

Gluu.Root # cd /install/community-edition
Gluu.Root # ./setup.py

```

- The configurations for the install should be automatically loaded and all you need to do here is press `Enter`

### 4. There needs to be primary server to replicate from initially for delta-syncrepl to inject data from. After the initial sync, all servers will be exactly the same, as delta-syncrepl will fill the newly created database.

- So choose one server as a base and then on every **other** server:

```
Gluu.Root # rm /opt/gluu/data/main_db/*.mdb
```

- Now make accesslog directories on **every server** (including the primary server) and give ldap ownership:

```
Gluu.Root # mkdir /opt/gluu/data/accesslog_db
Gluu.Root # chown -R ldap. /opt/gluu/data/
```

### 5. Now is where we will set servers to associate with each other for MMR by editing the slapd.conf, ldap.conf and symas-openldap.conf files.

- Creating the slapd.conf file is relatively easy, but can be prone to errors if done manually. Attached is a script and template files for creating multiple slapd.conf files for every server. Download git and clone the necessary files on **one** server:

```
Gluu.Root # apt-get update && apt-get install git && cd /tmp/ && git clone https://github.com/GluuFederation/cluster-mgr.git && cd /tmp/cluster-mgr/manual_install/slapd_conf_script/
```

- We need to change the configuration file for our own specific needs:

```
Gluu.Root # vi syncrepl.cfg
```

- Here we want to change the `ip_address`, `fqn_hostname`, `ldap_password` to our specific server instances. For example:

```

[server_1]
ip_address = 159.203.126.10
fqn_hostname = c5.gluu.org
ldap_password = (your password)
enable = Yes

[server_2]
ip_address = 138.197.65.243
fqn_hostname = c6.gluu.org
ldap_password = (your password)
enable = Yes

[server_3]
...
[nginx]
fqn_hostname = c4.gluu.org

```

 - Include the FQDN's and IP addresses of your Gluu servers and NGINX server (if you want the NGINX configuration file to be created automatically)

- If required, you can change the `/tmp/cluster-mgr/manual_install/slapd_conf_script/ldap_templates/slapd.conf` to fit your specific needs to include different schemas, indexes, etc. Avoid changing any of the `{#variables#}`.

- Now run the python script `create_slapd_conf.py` (Built with python 2.7) in the `/tmp/cluster-mgr/manual_install/slapd_conf_script/` directory :

```

Gluu.Root # python create_slapd_conf.py

```

- There is also a 2.6 Python script included.

- This will output multiple `.conf` files in `/tmp/cluster-mgr/manual_install/slapd_conf_script/` named to match your server FQDN:

```

Gluu.Root #  ls

... c5_gluu_org.conf  c6_gluu_org.conf ... nginx.conf

```

- Move each .conf file to their respective server replacing the `slapd.conf`:

```

Gluu.Root # mv /tmp/cluster-mgr/manual_install/slapd_conf_script/c5_gluu_org.conf /opt/symas/etc/openldap/slapd.conf

```

and for the other servers

```
Gluu.Root # logout
scp /opt/gluu-server-3.1.1/tmp/cluster-mgr/manual_install/slapd_conf_script/c6_gluu_org.conf root@c6.gluu.org:/opt/gluu-server-3.1.1/opt/symas/etc/openldap/slapd.conf

```

- Now create and modify the ldap.conf **on every server**:

```

Gluu.Root # vi /opt/symas/etc/openldap/ldap.conf

```

- Add these lines (it's an empty file)

```

TLS_CACERT /etc/certs/openldap.pem
TLS_REQCERT never

```

- Modify the HOST_LIST entry of symas-openldap.conf **on every server**:

```

vi /opt/symas/etc/openldap/symas-openldap.conf

```


- Replace:

```

HOST_LIST="ldaps://0.0.0.0:1636/"

```

- With:

```

HOST_LIST="ldaps://0.0.0.0:1636/ ldaps:///"

```

- **On all your servers**, inside the chroot, modify `/etc/gluu/conf/ox-ldap.properties` replacing:

`servers: localhost:1636`

With (obviously use your own FQDN's):

`servers: c5.gluu.org:1636,c6.gluu.org:1636,...`

Placing all servers in your cluster topology in this config portion.

### 6. It is important that our servers times are synchronized so we must install ntp outside of the Gluu chroot and set ntp to update by the minute (necessary for delta-sync log synchronization). If time gets out of sync, the entries will conflict and their could be issues with replication.

```
GLUU.root@host:/ # logout
# apt install ntp
# crontab -e
```

- Select your preferred editor and add this to the bottom of the file:

```
* * * * * /usr/sbin/ntpdate -s time.nist.gov
```
 
- This synchronizes the time every minute.

- Force-reload solserver on every server
```
# service gluu-server-3.1.0 login
# service solserver force-reload
```

- Delta-sync multi-master replication should be initializing and running. Check the logs for confirmation. It might take a moment for them to sync, but you should end up see something like the following:

```
# tail -f /var/log/openldap/ldap.log | grep sync

Aug 23 22:40:29 dc4 slapd[79544]: do_syncrep2: rid=001 cookie=rid=001,sid=001,csn=20170823224029.216104Z#000000#001#000000
Aug 23 22:40:29 dc4 slapd[79544]: syncprov_matchops: skipping original sid 001
Aug 23 22:40:29 dc4 slapd[79544]: syncrepl_message_to_op: rid=001 be_modify
```

### 7. **If you have your own load balancer these NGINX configuration files are merely a guide for how to interact with the Gluu server.** Let's configure our NGINX server for oxTrust and oxAuth web failover.

- We need the httpd.crt and httpd.key certs from one of the Gluu servers.   

- From the NGINX server:  

```

mkdir /etc/nginx/ssl/

scp root@server1.com:/opt/gluu-server-3.1.0/etc/certs/httpd.key /etc/nginx/ssl/
scp root@server1.com:/opt/gluu-server-3.1.0/etc/certs/httpd.crt /etc/nginx/ssl/

```

- From the Gluu server:

```
scp /opt/gluu-server-3.1.1/etc/certs/httpd.key root@c4.gluu.org:/etc/nginx/ssl/
scp /opt/gluu-server-3.1.1/etc/certs/httpd.crt root@c4.gluu.org:/etc/nginx/ssl/
```

### 8. Next we install NGINX

- On c4.gluu.org 

```

apt-get install nginx -y

```

- And from the server we created our nginx.conf file (c5.gluu.org in my case), to the NGINX server (c4.gluu.org)

- Put the following template in it's place. Make sure to change the `{serverX_ip_or_FQDN}` portion to your servers IP addresses or FQDN under the upstream section. Add as many servers as exist in your replication setup. The `server_name` needs to be your NGINX servers FQDN.    

```
events {
        worker_connections 768;
}

http {
  upstream backend_id {
    ip_hash;
    server {server1_ip_or_FQDN}:443;
    server {server2_ip_or_FQDN}:443;
  }
  upstream backend {
    server {server1_ip_or_FQDN}:443;
    server {server2_ip_or_FQDN}:443;
        
  }
  server {
    listen       80;
    server_name  {NGINX_server_FQDN};
    return       301 https://{NGINX_server_FQDN}$request_uri;
   }
  server {
    listen 443;
    server_name {NGINX_server_FQDN};

    ssl on;
    ssl_certificate         /etc/nginx/ssl/httpd.crt;
    ssl_certificate_key     /etc/nginx/ssl/httpd.key;

    location ~ ^(/)$ {
      proxy_pass https://backend;
    }
    location /.well-known {
        proxy_pass https://backend/.well-known;
    }
    location /oxauth {
        proxy_pass https://backend/oxauth;
    }
    location /identity {
        proxy_pass https://backend_id/identity;
    }

  }
}

```
- For web GUI to persist logins we must change some setting's on the Gluu page itself:
  - Access your website by entering your load-balancer or NGINX FQDN
  - Login with admin and use your LDAP password
  - go to the `Configuration` drop down on the left
  - `> JSON Configuration`
  - `Memcached configuration` tab
  - Change the drop down of `cacheProviderType` from `IN_MEMORY` to `MEMCACHED`
  - Add every server from your replication set-up, using spaces for separations and **not** commas. e.g `c1.gluu.org:11211 c2.gluu.org:11211`

- Now all traffic for the Gluu web GUI will route through one address e.g. `nginx.gluu.info`. This gives us failover redundancy for our Gluu web GUI if any server goes down, as NGINX automatically does passive health checks.   

## Support
If you have any questions or run into any issues, please open a ticket on [Gluu Support](https://support.gluu.org).

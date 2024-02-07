# Wazuh

What will this do for me / infrastructure / Retro setup

## Endpoint Security
- Configuration Assessment
- Malware Detection
- File Integrity Monitoring

## Threat Intelligence
- Threat Hunting
- Log Data Analysis
- Vulnerability Detection

## Security Operations
- Incident Response
- Regulatory Compliance
- IT Hygiene

## Cloud Security
- Container Security
- Posture Management
- Workload Protection

A good example is protect your server from brute force attacks, mitigate DDOS with the use of the Cloudflare API, custom security scripts etc. etc.

# Deploy Wazuh Docker in single node configuration

This deployment is defined in the `docker-compose.yml` file with one Wazuh manager containers, one Wazuh indexer containers, and one Wazuh dashboard container. It can be deployed by following these steps: 

```
mkdir /docker
cd docker
git clone https://git.krews.org/duckietm/wazuh-siem.git
cd wazuh-siem
```

1) Increase max_map_count on your host (Linux). This command must be run with root permissions:
```
$ sysctl -w vm.max_map_count=262144
```
2) Run the certificate creation script:
```
$ docker-compose -f generate-indexer-certs.yml run --rm generator
```
2.a)
If you run Wazuh behind a Proxy server, add the following to the generate-indexer-certs.yml:
```
    environment:
      - HTTP_PROXY=YOUR_PROXY_ADDRESS_OR_DNS
``` 
a quick example:
```
services:
  generator:
    image: wazuh/wazuh-certs-generator:0.0.1
    hostname: wazuh-certs-generator
    volumes:
      - ./config/wazuh_indexer_ssl_certs/:/certificates/
      - ./config/certs.yml:/config/certs.yml
    environment:
      - HTTP_PROXY=10.10.0.1
```

3) Start the environment with docker-compose:

- In the background:
```
$ docker-compose up -d
```

The environment takes about 1 minute to get up (depending on your Docker host) for the first time since Wazuh Indexer must be started for the first time and the indexes and index patterns must be generated.


### URL 

https://#Your IP#:5601  # Default username:admin Default password:SecretPassword 

# Change the password of Wazuh users
To improve security, you can change the default password of the Wazuh users. There are two types of Wazuh users:

- Wazuh indexer users
- Wazuh API users

To change the password of the default admin and kibanaserver users, do the following. <b>You can only change one at a time.<b>

1. Stop the deployment stack if it’s running:
```
docker-compose down
```

2. Run this command to generate the hash of your new password. Once the container launches, input the new password and press Enter.
```
docker run --rm -ti wazuh/wazuh-indexer:4.6.0 bash /usr/share/wazuh-indexer/plugins/opensearch-security/tools/hash.sh
```
3. Copy the generated hash to a notepad.

4. Open the config/wazuh_indexer/internal_users.yml file in your docker location.
Locate the block for the user you are changing password for.

5. Replace the hash.

* admin user !!! Remember only 1 user at the time !!!
```
admin:
  hash: "$2y$12$K/SpwjtB.wOHJ/Nc6GVRDuc1h0rM1DfvziFRNPtk27P.c4yDr9njO"
  reserved: true
  backend_roles:
  - "admin"
  description: "Demo admin user"
```
* kibanaserver user !!! Remember only 1 user at the time !!!
```
kibanaserver:
  hash: "$2a$12$4AcgAt3xwOWadA5s5blL6ev39OXDNhmOesEoo33eZtrq2N0YrU3H."
  reserved: true
  description: "Demo kibanaserver user"
```
## Setting the new password

Open the docker-compose.yml file. Change all occurrences of the old password with the new one. For example:

admin user
```
services:
  wazuh.manager:
    ...
    environment:
      - INDEXER_URL=https://wazuh.indexer:9200
      - INDEXER_USERNAME=admin
      - INDEXER_PASSWORD=SecretPassword
      - FILEBEAT_SSL_VERIFICATION_MODE=full
      - SSL_CERTIFICATE_AUTHORITIES=/etc/ssl/root-ca.pem
      - SSL_CERTIFICATE=/etc/ssl/filebeat.pem
      - SSL_KEY=/etc/ssl/filebeat.key
      - API_USERNAME=wazuh-wui
      - API_PASSWORD=MyS3cr37P450r.*-
  ...
  wazuh.dashboard:
    ...
    environment:
      - INDEXER_USERNAME=admin
      - INDEXER_PASSWORD=SecretPassword
      - WAZUH_API_URL=https://wazuh.manager
      - DASHBOARD_USERNAME=kibanaserver
      - DASHBOARD_PASSWORD=kibanaserver
      - API_USERNAME=wazuh-wui
      - API_PASSWORD=MyS3cr37P450r.*-
```

kibanaserver user

```
services:
  wazuh.dashboard:
    ...
    environment:
      - INDEXER_USERNAME=admin
      - INDEXER_PASSWORD=SecretPassword
      - WAZUH_API_URL=https://wazuh.manager
      - DASHBOARD_USERNAME=kibanaserver
      - DASHBOARD_PASSWORD=kibanaserver
      - API_USERNAME=wazuh-wui
      - API_PASSWORD=MyS3cr37P450r.*-
```

## Applying the changes

Start the deployment stack.
```
docker-compose up -d
```

Run docker ps and note the name of the first Wazuh indexer container. For example, single-node-wazuh.indexer-1.

Run ```docker exec -it <WAZUH_INDEXER_CONTAINER_NAME> bash``` to enter the container. For example:

```
docker exec -it single-node-wazuh.indexer-1 bash
```
Set the following variables from command line:

```
export INSTALLATION_DIR=/usr/share/wazuh-indexer
CACERT=$INSTALLATION_DIR/certs/root-ca.pem
KEY=$INSTALLATION_DIR/certs/admin-key.pem
CERT=$INSTALLATION_DIR/certs/admin.pem
export JAVA_HOME=/usr/share/wazuh-indexer/jdk
```

Wait for the Wazuh indexer to initialize properly.
<b>The waiting time can vary from two to five minutes. But just get a coffee and wait 5 min!<b>
It depends on the size of the cluster, the assigned resources, and the speed of the network.

now after 5 min. run the following command:
```
bash /usr/share/wazuh-indexer/plugins/opensearch-security/tools/securityadmin.sh -cd /usr/share/wazuh-indexer/opensearch-security/ -nhnv -cacert  $CACERT -cert $CERT -key $KEY -p 9200 -icl
```

# Wazuh API users
The wazuh-wui user is the user to connect with the Wazuh API by default. Follow these steps to change the password.

Note The password for Wazuh API users must be between 8 and 64 characters long. It must contain at least one uppercase and one lowercase letter, a number, and a symbol.

1. Open the file config/wazuh_dashboard/wazuh.yml and modify the value of password parameter.

```
hosts:
  - 1513629884013:
      url: "https://wazuh.manager"
      port: 55000
      username: wazuh-wui
      password: "MyS3cr37P450r.*-"
      run_as: false
...
```

2. Open the docker-compose.yml file. Change all occurrences of the old password with the new one.

```
services:
  wazuh.manager:
    ...
    environment:
      - INDEXER_URL=https://wazuh.indexer:9200
      - INDEXER_USERNAME=admin
      - INDEXER_PASSWORD=SecretPassword
      - FILEBEAT_SSL_VERIFICATION_MODE=full
      - SSL_CERTIFICATE_AUTHORITIES=/etc/ssl/root-ca.pem
      - SSL_CERTIFICATE=/etc/ssl/filebeat.pem
      - SSL_KEY=/etc/ssl/filebeat.key
      - API_USERNAME=wazuh-wui
      - API_PASSWORD=MyS3cr37P450r.*-   # Change this one
  ...
  wazuh.dashboard:
    ...
    environment:
      - INDEXER_USERNAME=admin
      - INDEXER_PASSWORD=SecretPassword
      - WAZUH_API_URL=https://wazuh.manager
      - DASHBOARD_USERNAME=kibanaserver
      - DASHBOARD_PASSWORD=kibanaserver
      - API_USERNAME=wazuh-wui
      - API_PASSWORD=MyS3cr37P450r.*- # Change this one
```

3. Recreate the Wazuh containers:

```
docker-compose down
docker-compose up -d
```
Now have a lot off funn with it !

# Custom script

In the Custom_scripts directory you can find all the scripts you can you use to protect your server.

# resources

Documentation: https://documentation.wazuh.com/current/index.html

Original github: https://github.com/wazuh/wazuh-docker

# Upgrade

Change the version number in the docker-compose.yml
Latest Docker version is 4.7.0 (keep this otherwise things will break!)

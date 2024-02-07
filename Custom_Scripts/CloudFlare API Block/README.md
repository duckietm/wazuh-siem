Place this file in /var/ossec/active-response/bin : On the Client
Edit the following:
```
TOKEN='Cloudflare API Token'
USER='cloudflareuser@email.com'
```
Make sure you have got a API user in Cloudflare 

Sample config (Edit /docker/wazuh-docker/config/wazuh_cluster/wazuh_manager.conf this is the shared config that will install to all clients):
  <command>
     <name>cloudflare-ban</name>
     <executable>cloudflare-ban.sh</executable>
     <timeout_allowed>yes</timeout_allowed>
     <expect>srcip</expect>
  </command>

  <active-response>
     <command>cloudflare-ban</command>
     <location>server</location>
     <rules_id>31151,31152,31153,31154,31163</rules_id>
     <timeout>43200</timeout>
  </active-response>
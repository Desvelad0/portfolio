---
title: "Upgrading Wazuh components"
slug: "upgrading_wazuh_components"
tags: [wazuh,linux,ubuntu]
date: 2025-02-23T22:03:00+02:00
draft: false
---

I thought upgrading Wazuh along with its several components would be as simple as running `sudo apt update && sudo apt upgrade -y`, but last time when I attempted this a few months ago, the setup stopped working, so I'm documenting the steps required to update the setup in a [proper manner](https://documentation.wazuh.com/current/upgrade-guide/upgrading-central-components.html) to the tunes of Virtual Riot, one of the most talented EDM artists out there.

First I obviously will run `apt update`  on both the Wazuh Indexer and Wazuh Server servers, as I chose to install the components in Ubuntu. I then apparently need to install the ``gnupg apt-transport-https`` before continuing, and install the Wazuh GPG key with:

```  
# curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | gpg --no-default-keyring --keyring gnupg-ring:/usr/share/keyrings/wazuh.gpg --import && chmod 644 /usr/share/keyrings/wazuh.gpg
```

Next, I need to make sure the Wazuh repository is available on all the relevant servers:

```
# echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" | tee -a /etc/apt/sources.list.d/wazuh.list && apt update
```

After running this I noticed a warning, so this wasn't exactly needed and I had to remove the duplicate entry:

```
W: Target Packages (main/binary-amd64/Packages) is configured multiple times in /etc/apt/sources.list.d/wazuh.list:1 and /etc/apt/sources.list.d/wazuh.list:2
W: Target Packages (main/binary-all/Packages) is configured multiple times in /etc/apt/sources.list.d/wazuh.list:1 and /etc/apt/sources.list.d/wazuh.list:2
W: Target Translations (main/i18n/Translation-en) is configured multiple times in /etc/apt/sources.list.d/wazuh.list:1 and /etc/apt/sources.list.d/wazuh.list:2
W: Target Packages (main/binary-amd64/Packages) is configured multiple times in /etc/apt/sources.list.d/wazuh.list:1 and /etc/apt/sources.list.d/wazuh.list:2
W: Target Packages (main/binary-all/Packages) is configured multiple times in /etc/apt/sources.list.d/wazuh.list:1 and /etc/apt/sources.list.d/wazuh.list:2
W: Target Translations (main/i18n/Translation-en) is configured multiple times in /etc/apt/sources.list.d/wazuh.list:1 and /etc/apt/sources.list.d/wazuh.list:2
```

Next, stopping ``filebeat`` and `wazuh-dashboard` with `systemctl stop` on the server containing them, which if following best practises suggested by Wazuh themselves, these should be on a dedicated server (and they are for me.)

## Upgrading Wazuh indexer

Now is the time to upgrade the Wazuh indexer by running this:

```
# curl -X PUT "https://<WAZUH_INDEXER_IP_ADDRESS>:9200/_cluster/settings" \
-u <USERNAME>:<PASSWORD> -k -H "Content-Type: application/json" -d '
{
   "persistent": {
      "cluster.routing.allocation.enable": "primaries"
   }
}'
```

The username and password have to indeed be in the format ``admin:password`` without any brackets, sometimes I find this part confusing in some documentation, though then again I'm not sure you would actually use brackets anywhere. After running the json above, I get a ``{"acknowledged":true,"persistent":{"cluster":{"routing":{"allocation":{"enable":"primaries"}}}},"transient":{}}`` , confirming that the command was successful.

Next up:

```
# curl -X POST "https://<WAZUH_INDEXER_IP_ADDRESS>:9200/_flush" -u <USERNAME>:<PASSWORD> -k
```

Output:

```
{"_shards":{"total":889,"successful":889,"failed":0}}
```

Perfect.

In my sort of setup where I have just one Wazuh manager node, I now need to run ``systemctl stop wazuh-manager`` .

These following steps need to be ran on every Wazuh indexer node, which for me is just one; I think I'll have to install at least one more of these to have some high-availability and more interesting times when updating (not to mention constant connectivity between agents and the indexer):

``# systemctl stop wazuh-indexer`` 

``# apt-get install wazuh-indexer`` (running this asks you whether you want to keep your own configuration file or install a default package maintainer one for multiple things; I don't see any reason not to keep my own config files.)

Next:

```
# systemctl daemon-reload
# systemctl enable wazuh-indexer
# systemctl start wazuh-indexer
```

## Post Wazuh Indexer upgrade

Checking if the newly upgraded indexer node is in the cluster (of one, so not really a cluster):

``curl -k -u <USERNAME>:<PASSWORD> https://<WAZUH_INDEXER_IP_ADDRESS>:9200/_cat/nodes?v`` 

This prompts me with a ``connection refused`` . Oof. We'll see if this is only for a larger setup since ``wazuh-indexer`` is running without any errors.

Re-enabling shard allocation:

```
# curl -X PUT "https://<WAZUH_INDEXER_IP_ADDRESS>:9200/_cluster/settings" \
-u <USERNAME>:<PASSWORD> -k -H "Content-Type: application/json" -d '
{
   "persistent": {
      "cluster.routing.allocation.enable": "all"
   }
}
'
```

Initially had an error with this but turns out ``localhost`` will not work even if it literally is the localhost where the service is running, I remember this being an issue before. You need to state the actual IP address to get the feedback:

```
{"acknowledged":true,"persistent":{"cluster":{"routing":{"allocation":{"enable":"all"}}}},"transient":{}}
```

The documentation once again asks you to check on the cluster shard allocation, which I now realise didn't work last time because I had localhost instead of the IP address:

```
ip            heap.percent ram.percent cpu load_1m load_5m load_15m node.role node.roles                               cluster_manager name
192.168.77.12           48          92   3    0.05    0.11     0.09 dimr      data,ingest,master,remote_cluster_client *               node-1

```

Perfection. I can now start the `wazuh-manager` again.

Here we should note that this upgrade process thus far would not upgrade any opensearch plugins, and we can check if there are any outdated ones with ``/usr/share/wazuh-indexer/bin/opensearch-plugin list``; if there are any, the outdated ones should state "outdated" which for me they don't, which is surprising since I installed this quite a while ago, last May.

That's it for the Wazuh indexer.

## Upgrading Wazuh server

Now is the time to apparently just straight up run ``apt-get install wazuh-manager``. Since I am upgrading from a 4.7.x version, I have some additional steps I need to take to configure vulnreability detection it seems; I need to delete the old ``<vulnerability-detector>`` block from ``/var/ossec/etc/ossec.conf`` and replace it with the following (after taking a backup of the file just in case):

```
<vulnerability-detection>
   <enabled>yes</enabled>
   <index-status>yes</index-status>
   <feed-update-interval>60m</feed-update-interval>
</vulnerability-detection>
```

The documentation does not say anything about anything below the vulnerability-detector block which includes OS specific listings, so I assume those are not touched. 

Next, I need to ensure the <indexer> block contains the details of my Wazuh indexer host, specifically the IP address, which by default is just set to ``0.0.0.0`` and it needs to be changed to the actual IP address of the indexer node. 

**Note for later**: apparently in a Wazuh indexer cluster, you have to add a <host> entry in the Wazuh manager `/var/ossec/etc/ossec.conf` for each indexer node in the cluster (example of using two nodes, first one on the list is prioritized for reporting and then it would go down the list):

```
<hosts>
   <host>https://10.0.0.1:9200</host>
   <host>https://10.0.0.2:9200</host>
</hosts>
```

Now I need to store Wazuh indexer credentials to a Wazuh manager keystore:

```
# echo '<INDEXER_USERNAME>' | /var/ossec/bin/wazuh-keystore -f indexer -k username
# echo '<INDEXER_PASSWORD>' | /var/ossec/bin/wazuh-keystore -f indexer -k password
```

## Upgrading Filebeat

Now I have to update the Wazuh Filebeat module and alerts template to work with the latest Wazuh indexer version:

``` 
# curl -s https://packages.wazuh.com/4.x/filebeat/wazuh-filebeat-0.4.tar.gz | sudo tar -xvz -C /usr/share/filebeat/module
```

```
# curl -so /etc/filebeat/wazuh-template.json https://raw.githubusercontent.com/wazuh/wazuh/v4.11.0/extensions/elasticsearch/7.x/wazuh-template.json
chmod go+r /etc/filebeat/wazuh-template.json
```

```
# systemctl daemon-reload
# systemctl enable filebeat
# systemctl start filebeat
```

```
# filebeat setup --pipelines
# filebeat setup --index-management -E output.logstash.enabled=false
```

That is it for Filebeat for me since I'm coming from version 4.7.x.

## Upgrading Wazuh dashboard

I need to backup the old `opensearch_dashboards.yml` file:

`cp /etc/wazuh-dashboard/opensearch_dashboards.yml /etc/wazuh-dashboard/opensearch_dashboards.yml.old` 

At this point I ran out of disk space... In my hypervisor you cannot expand the disk space of a VM if it has snapshots, good thing I have a full backup of this VM. After deleting the snapshot and expanding the disk space on the hypervisor side:

`echo 1>/sys/block/sda/device/rescan` (to get access to the extra space in the next step):

`cfdisk /dev/sda`

```
                                       Disk: /dev/sda
                          Size: 40 GiB, 42949672960 bytes, 83886080 sectors
                        Label: gpt, identifier: 48D75F38-DC77-41E2-BBF0-AE4AB5EC2DC7

    Device                                 Start                  End              Sectors              Size Type
>>  /dev/sda1                               2048              2203647              2201600                1G EFI System
    /dev/sda2                            2203648              6397951              4194304                2G Linux filesystem
    /dev/sda3                            6397952             67106815             60708864             28.9G Linux filesystem
    Free space                          67106816             83886046             16779231                8G

```

In the above menu, you merely have to choose the device to expand, select resize which will automatically prompt to use the newly acquired space, press enter, choose write, type yes and then quit out of the program. This alone did not give me the access to the expanded space, so I had to also run: 

``# lvextend -l +100%FREE /dev/mapper/ubuntu--vg-ubuntu--lv`` 

Still not enough:

`# resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv`

And there, disk extended.

Continuing with the upgrade:

`# apt-get install wazuh-dashboard`

This time when I'm prompted about keeping or updating the .yml file, the documentation tells me to replace the existing one with the updated one (package maintainer's version.) 

Now I have to reapply "any configuration changes" to the .yml file, having to pay special attention to values of `server.ssl.key` and `server.ssl.certificate` , and to make sure they match the files located in `/etc/wazuh-dashboard/certs/`. Wonder why I had to replace it then... Only things that were missing were the correct IP addresses for me. Another thing to check for was to make sure the following line is in the file, which it is for me:

`uiSettings.overrides.defaultRoute: /app/wz-home`

Then we need to run the good ol' systemctl stuff:

```
systemctl daemon-reload
systemctl enable wazuh-dashboard
systemctl start wazuh-dashboard
```

... And at this stage the Wazuh dashboard should be accessible at ``https://<DASHBOARD_IP_ADDRESS>/app/wz-home`` which it isn't for me:

![img](/wazuh/wazuhdashboard.png)

Doing `systemctl status wazuh-dashboard` shows me the following:

```
Feb 23 15:49:21 reina opensearch-dashboards[633212]: {"type":"log","@timestamp":"2025-02-23T15:49:21Z","tags":["error","opensearch","data"],"pid":633212,"message":"[ConnectionError]: connect ECONNREFUSED 127.0.0.1:9200"}
```

I looked again at the ``/etc/wazuh-dashboard/opensearch_dashboards.yml`` file, and noticed commented out lines for `opensearch.username` and `opensearch.password`; I populated the fields, restarted the service and now could once again login to the dashboard. I was already surprised at how easy this was to fix until I see this upon logging in:

![apiconnectionproblem](/wazuh/apiconnectiondown.png)

Of course it won't be easy!

Thankfully there is a handy troubleshooting link there, which asks you to check the service status for `wazuh-manager`:

```
Feb 22 23:58:32 reina env[616941]: 2025/02/22 23:58:32 wazuh-modulesd: WARNING: (1230): Invalid element in the configuration: 'provider'.
```

Looking at the ``/var/ossec/etc/ossec.conf`` file where I had to remove ``<vulnerability-detection`` and replace it with a new one, I realise this is referring to what I was thinking about earlier; the OS specific vulnerabilities which in my version were listed one by one after the old ``<vulnerability-detection`` block.

Let's see if https://documentation.wazuh.com/current/user-manual/capabilities/vulnerability-detection/configuring-scans.html helps here. I suppose it is a bit of a jump from my older 4.7.x version to the newest version at the time of writing of 4.11. At least they made vulnerability detection an automatically enabled feature now, which I completely agree with being on by default.

I can't find anything helpful from the above article, but did notice several new interesting links that I'll have to take a look at in later posts; "[Leveraging LLMs for alert enrichment](https://documentation.wazuh.com/current/proof-of-concept-guide/leveraging-llms-for-alert-enrichment.html)" sure sounds interesting. I also notice that the upgrade guide doesn't mention anything about this particular error after an upgrade. Apparently the vulnerability detection changed quite a bit in the 4.8 version, and searching around I can see many people had this same issue I'm facing right now.

Well I ended up finding a Reddit post that linked to [this YouTube video](https://www.youtube.com/watch?app=desktop&v=_RWWALAAddo) about upgrading to 4.8.0; although they do technically say it in the documentation, they don't directly say that now all you need for the detection of vulnerabilities is the following:

```
<vulnerability-detection>
   <enabled>yes</enabled>
   <index-status>yes</index-status>
   <feed-update-interval>60m</feed-update-interval>
</vulnerability-detection>

<indexer>
   <enabled>yes</enabled>
   <hosts>
      <host>https://0.0.0.0:9200</host>
   </hosts>
   <ssl>
      <certificate_authorities>
         <ca>/etc/filebeat/certs/root-ca.pem</ca>
      </certificate_authorities>
      <certificate>/etc/filebeat/certs/filebeat.pem</certificate>
      <key>/etc/filebeat/certs/filebeat-key.pem</key>
   </ssl>
</indexer>
```

That's it. Yes they state it, however they never explicitly said that those coming from previous versions could just delete any mention of the various different OS vulnerability feeds, which is the reason I was having this error.

Getting rid of all the OS specific lines and restarting the dashboard now makes the API connection show up as online, and after manually pressing "check for updates", the server is able to look for updates. The upgrade is done!

... Yet I see this:

![vulnprob](/wazuh/vulnproblem.png)

So it doesn't completely work still. 

![vulndetection](/wazuh/vulndetection.png)

Right. So on one hand it is enabled by default, but on the other hand you have to go enable the detection in `/var/ossec/etc/internal_options.conf`. I love this software and its documentation, but this is just confusing. I went ahead and changed the 1 into 0 in the `internal_options.conf`, restarted the Wazuh manager and still had the same issue. The [FAQ](https://documentation.wazuh.com/4.11/user-manual/capabilities/vulnerability-detection/FAQ.html#step-2-verify-the-connection) leads me to suspect that the issue is with the certificate, as the curl command suggested for troubleshooting here fails for me. Looking at `/var/ossec/etc/ossec.conf`, a certificate referenced exists nowhere on the system which is very interesting indeed (confirmed with `find / -name`).

## YES!

The issue was indeed with the certificate + key naming in `ossec.conf`. I have no idea whether the certificate and key somehow changed names, or if the `ossec.conf` changed with the update. Whichever it was, it now works, and I can see all the events and vulnerabilities of all my agents in a very slick improved GUI. 

Finally, upgrading agents, if lucky, is as easy as running this on the server containing the ``wazuh-manager``: 

```
# /var/ossec/bin/agent_upgrade -a 010

Upgrading...

Upgraded agents:
        Agent 010 upgraded: Wazuh v4.7.4 -> Wazuh v4.11.0
```

That is all.

As I expected, the upgrade process was not as straightforward as one might imagine, but having done this sort of a thing for many years now, it was certainly doable if a bit irritating at times. I'll certainly be looking more into the new interesting documentation on Wazuh such as the aforementioned LLM integrations for alert enrichment, sounds very interesting indeed. For now however, thank you for reading, and stay tuned for more content soonTM!

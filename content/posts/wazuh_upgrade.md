---
title: "Upgrading Wazuh components"
slug: "upgrading_wazuh_components"
tags: [azure,sentinel,terraform]
date: 2025-02-23T00:50:00+02:00
draft: true
---

I thought upgrading Wazuh along with its several components would be as simple as running ``sudo apt update && sudo apt upgrade -y`, but last time when I attempted this a few months ago, the setup stopped working, so I'm documenting the steps required to update the setup in a [proper manner](https://documentation.wazuh.com/current/upgrade-guide/upgrading-central-components.html) to the tunes of a lockdown set from Disciple records; if you like EDM, I recommend [giving it a listen](https://www.youtube.com/live/u4uddXsaXfg?si=CyP3jEbFJPxtZYUA&t=1136) (timestamp intended: if you don't like the first set's harder style, DalCo is great.)

First I obviously will run `apt update`  on both the Wazuh Indexer and Wazuh Server servers, as I chose to install the components in Ubuntu. I then apparently need to install the ``gnupg apt-transport-https`` before continuing, and install the Wazuh GPG key with:

```  
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | gpg --no-default-keyring --keyring gnupg-ring:/usr/share/keyrings/wazuh.gpg --import && chmod 644 /usr/share/keyrings/wazuh.gpg
```

Next, I need to make sure the Wazuh repository is available on all the relevant servers:

```
echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" | tee -a /etc/apt/sources.list.d/wazuh.list && apt update
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
curl -X PUT "https://<WAZUH_INDEXER_IP_ADDRESS>:9200/_cluster/settings" \
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
curl -X POST "https://<WAZUH_INDEXER_IP_ADDRESS>:9200/_flush" -u <USERNAME>:<PASSWORD> -k
```

Output:

```
{"_shards":{"total":889,"successful":889,"failed":0}}
```

Perfect.

In my sort of setup where I have just one Wazuh manager node, I now need to run ``systemctl stop wazuh-manager`` .

These following steps need to be ran on every Wazuh indexer node, which for me is just one; I think I'll have to install at least one more of these to have some high-availability and more interesting times when updating (not to mention constant connectivity between agents and the indexer):

``systemctl stop wazuh-indexer`` 

``apt-get install wazuh-indexer`` (running this asks you whether you want to keep your own configuration file or install a default package maintainer one for multiple things; I don't see any reason not to keep my own config files.)

Next:

```
systemctl daemon-reload
systemctl enable wazuh-indexer
systemctl start wazuh-indexer
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

Now is the time to apparently just straight up run ``apt-get install wazuh-manager``. 


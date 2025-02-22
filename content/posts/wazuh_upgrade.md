---
title: "Upgrading Wazuh components"
slug: "upgrading_wazuh_components"
tags: [azure,sentinel,terraform]
date: 2025-02-23T00:50:00+02:00
draft: true
---

I thought upgrading Wazuh would be as simple as running ``sudo apt update && sudo apt upgrade -y`, but last time when I attempted this a few months ago, the setup stopped working, so I'm documenting the steps required to update the setup in a [proper manner](https://documentation.wazuh.com/current/upgrade-guide/upgrading-central-components.html). 

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

These following steps need to be ran on every Wazuh indexer node, which for me is just one; I think I'll have to install at least one more of these to have some high-availability and more interesting times when updating (not to mention constant connectivity between agents and the indexer.)


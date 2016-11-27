+++
keywords = [
  'Security',
  'Cloud',
  'Infrastructure',
  'Azure',
  'Red Hat',
  'RHEL',
  'AWS'
]
categories = [
  'Cloud',
  'Security',
  'Azure'
]
title = "Azure bug bounty Root to storage account administrator"
description = "Details how its possible to become an azure Storage account administrator from having root access on a virtual machine"
date = "2016-11-27T02:47:11Z"

+++

In my previous blog post [Azure bug bounty Pwning Red Hat Enterprise Linux](http://ianduffy.ie/blog/2016/11/26/azure-bug-bounty-pwning-red-hat-enterprise-linux/) I detailed how it was possible to get administrative access to the Red Hat Update Infrastructure consumed by [Red Hat Enterprise Linux](https://www.redhat.com/en/technologies/linux-platforms/enterprise-linux) virtual machines booted from the [Microsoft Azure](https://azure.microsoft.com) Marketplace image. In theory, if exploited one could have gained root access to all virtual machines consuming the repositories by releasing an updated version of a common package and waiting for virtual machines to execute `yum update`.

As an attacker, this would have granted access to every piece of data on the compromised virtual machines. Sadly, the attack vector is actually much more widespread than this. Given some poor implementation within the mandatory [Microsoft Azure](https://azure.microsoft.com) Linux Agent (WaLinuxAgent) one is able to obtain the administrator API keys to the storage account used by the virtual machine for debug log shipping purposes, at the time of research this storage account defaulted to one shared by multiple virtual machines.

At the time of research, the [Red Hat Enterprise Linux](https://www.redhat.com/en/technologies/linux-platforms/enterprise-linux) image available on the [Microsoft Azure](https://azure.microsoft.com) Marketplace came with WaLinuxAgent 2.0.16. When a virtual machine was created with the "Linux diagnostic extension" enabled the API key for access to the specified storage account was written to `/var/lib/waagent/Microsoft.OSTCExtensions.LinuxDiagnostic-2.3.9007/xmlCfg.xml`.

Once acquired one can simply use the [Azure Xplat-CLI](https://github.com/Azure/azure-xplat-cli) to interact the storage account:

```
export AZURE_STORAGE_ACCOUNT="storage_account_name_as_per_xmlcfg"
export AZURE_STORAGE_ACCESS_KEY="storage_account_access_key_as_per_xmlcfg"
azure storage container list # acquire some container name
azure storage blob list # provide the container name
# Copy, download, upload, delete any blobs available across any containers you can access.
```

If the storage account was used by multiple virtual machines there is potential to download their virtual hard disks.


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
title = "Azure bug bounty Pwning Red Hat Enterprise Linux"
date = "2016-11-26T10:29:32Z"
description = "Acquired administrator level access to all of the [Microsoft Azure](https://azure.microsoft.com) managed [Red Hat Update Infrastructure](https://access.redhat.com/documentation/en/red-hat-update-infrastructure/3.0.beta.1/paged/system-administrator-guide/chapter-1-about-red-hat-update-infrastructure) that supplies all the packages for all [Red Hat Enterprise Linux](https://www.redhat.com/en/technologies/linux-platforms/enterprise-linux) instances booted from the Azure marketplace."

+++

*TL;DR Acquired administrator level access to all of the [Microsoft Azure](https://azure.microsoft.com) managed [Red Hat Update Infrastructure](https://access.redhat.com/documentation/en/red-hat-update-infrastructure/3.0.beta.1/paged/system-administrator-guide/chapter-1-about-red-hat-update-infrastructure) that supplies all the packages for all [Red Hat Enterprise Linux](https://www.redhat.com/en/technologies/linux-platforms/enterprise-linux) instances booted from the Azure marketplace.*

I was tasked with creating a machine image of [Red Hat Enterprise Linux](https://www.redhat.com/en/technologies/linux-platforms/enterprise-linux) that was compliant to the [Security Technical Implementation guide defined by the Department of Defense](https://www.stigviewer.com/stig/red_hat_enterprise_linux_6/).

This machine image was to be used for both [Amazon Web Services](https://aws.amazon.com/) and [Microsoft Azure](https://azure.microsoft.com). Both of which offer marketplace images which had a metered billing pricing model[1][2]. Ideally, I wanted my custom image to be billed under the same mechanism, as such the virtual machines would be able to consume software updates from a local [Red Hat Enterprise Linux](https://www.redhat.com/en/technologies/linux-platforms/enterprise-linux) repository owned and managed by the cloud provider.

Both [Amazon Web Services](https://aws.amazon.com/) and [Microsoft Azure](https://azure.microsoft.com) utilise a deployment of [Red Hat Update Infrastructure](https://access.redhat.com/documentation/en/red-hat-update-infrastructure/3.0.beta.1/paged/system-administrator-guide/chapter-1-about-red-hat-update-infrastructure) for supplying this functionality.

This setup requires two main parts:

### Red Hat Update Appliance

There is only one Red Hat Update Appliance per [Red Hat Update Infrastructure](https://access.redhat.com/documentation/en/red-hat-update-infrastructure/3.0.beta.1/paged/system-administrator-guide/chapter-1-about-red-hat-update-infrastructure) installation, however, both [Amazon Web Services](https://aws.amazon.com/) and [Microsoft Azure](https://azure.microsoft.com) create one per region.

The Red Hat Update Appliance is responsible for:

- Downloading new packages from the Red Hat CDN. It is the only component that requires a connection back to Red Hat.
- Copying new packages to each content delivery server
- Supplying a [REST API and command line interface for management of software repositories](http://pulpproject.org/)

**The Red Hat Update Appliance does not need to be exposed to the repository clients.**

### Content Delivery server

The content delivery server(s) provide the yum repositories that clients connect to for updated packages.

## Achieving metered billing

Both [Amazon Web Services](https://aws.amazon.com/) and [Microsoft Azure](https://azure.microsoft.com) use SSL certifications for authentication against the repositories.

However, these are the same SSL certificates for every instance.

 On [Amazon Web Services](https://aws.amazon.com/) having the SSL certificates is not enough, you must have booted your instance from an AMI that had an associated billing code. It is this billing code that ensures you pay the extra premium for running [Red Hat Enterprise Linux](https://www.redhat.com/en/technologies/linux-platforms/enterprise-linux).

On Azure it remains undefined how they manage to track billing. At the time of research, it was possible to copy the SSL certificates from one instance to another and successfully authenticate. Additionally, if you duplicated a [Red Hat Enterprise Linux](https://www.redhat.com/en/technologies/linux-platforms/enterprise-linux) virtual hard disk and created a new instance from it all billing association seemed to be lost but repository access was still available.

## Where Azure Failed

On Azure to setup repository connectivity, they provide an RPM with the necessary configuration. In the older version of their agent, it is responsible for this task [3].  The installation script it references comes from the following [archive](http://rhuirpm.blob.core.windows.net/script/rhui.tar.gz). If you expand this archive you will find the client configuration for each region.

By browsing the metadata of the RPMs we can discover some interesting information:

```bash
$ rpm -qip RHEL6-2.0-1.noarch.rpm
Name        : RHEL6                        Relocations: (not relocatable)
Version     : 2.0                               Vendor: (none)
Release     : 1                             Build Date: Sun 14 Feb 2016 06:40:54 AM UTC
Install Date: (not installed)               Build Host: westeurope-rhua.cloudapp.net
Group       : Applications/Internet         Source RPM: RHEL6-2.0-1.src.rpm
Size        : 20833                            License: GPLv2
Signature   : (none)
URL         : http://redhat.com
Summary     : Custom configuration for a cloud client instance
Description :
Configurations for a client to connect to the RHUI infrastructure
```

As you can see, the build host enables us to discover all of the Red Hat Update Appliances:

```bash
$ host westeurope-rhua.cloudapp.net
westeurope-rhua.cloudapp.net has address 104.40.209.83

$ host eastus2-rhua.cloudapp.net
eastus2-rhua.cloudapp.net has address 13.68.20.161

$ host southcentralus-rhua.cloudapp.net
southcentralus-rhua.cloudapp.net has address 23.101.178.51

$ host southeastasia-rhua.cloudapp.net
southeastasia-rhua.cloudapp.net has address 137.116.129.134
```

At the time of research, all of servers were exposing their REST APIs over HTTPs.

The URL to the archive containing these RPMs was discovered a package labeled PrepareRHUI on available on any [Red Hat Enterprise Linux](https://www.redhat.com/en/technologies/linux-platforms/enterprise-linux) Box running on [Microsoft Azure](https://azure.microsoft.com).

```bash
$ yumdownloader PrepareRHUI
$ rpm -qip PrepareRHUI-1.0.0-1.noarch.rpm
Name        : PrepareRHUI                  Relocations: (not relocatable)
Version     : 1.0.0                             Vendor: Microsoft Corporation
Release     : 1                             Build Date: Mon 16 Nov 2015 06:13:21 AM UTC
Install Date: (not installed)               Build Host: rhui-monitor.cloudapp.net
Group       : Unspecified                   Source RPM: PrepareRHUI-1.0.0-1.src.rpm
Size        : 770                              License: GPL
Signature   : (none)
Packager    : Microsoft Corporation <xiazhang@microsoft.com>
Summary     : Prepare RHUI installation for Redhat client
Description :
PrepareRHUI is used to prepare RHUI installation for before making a Redhat image.
```

The build host is interesting `rhui-monitor.cloudapp.net`, at the time of research running a port scan revealed an application running on port 8080.

![Microsoft Azure RHUI Monitoring tool](/monitor.png)

Despite the application requiring username and password based authentication, It was possible to execute a run of their "backend log collector" on a specified content delivery server. When the collector service completed the application supplied URLs to archives which contain multiple logs and configuration files from the servers.

Included within these archives was an SSL certificate that would grant full administrative access to the Red Hat Update Appliances [4].

![Pulp admin keys for Microsoft Azure's RHUI](/directory.png)

At the time of research all [Red Hat Enterprise Linux](https://www.redhat.com/en/technologies/linux-platforms/enterprise-linux) virtual machines booted from the Azure Marketplace image had the following additional repository configured:

```
[rhui-PA]
name=Packages for Azure
mirrorlist=https://eastus2-cds1.cloudapp.net/pulp/mirror/PA
enabled=1
gpgcheck=0
sslverify=1
sslcacert=/etc/pki/rhui/ca.crt
```

Given no gpgcheck is enabled, with full administrative access to the [Red Hat Enterprise Linux](https://www.redhat.com/en/technologies/linux-platforms/enterprise-linux) Appliance REST API one could have uploaded packages that would be acquired by client virtual machines on their next yum update.

The issue was reported in accordance to the [Microsoft Online Services Bug Bounty terms](https://technet.microsoft.com/en-us/library/dn800983.aspx). Microsoft agreed it was a vulnerability in their systems. Immediate action was taken to prevent public access to `rhui-monitor.cloudapp.net`. Additionally, they eventually prevented public access to the Red Hat Update Appliances and they claim to of rotated all secrets.

[1] https://azure.microsoft.com/en-in/pricing/details/virtual-machines/red-hat/

[2] https://aws.amazon.com/partners/redhat/

[3] https://github.com/Azure/azure-linux-extensions/blob/master/Common/WALinuxAgent-2.0.16/waagent#L2891

[4] https://fedorahosted.org/pulp/wiki/Certificates

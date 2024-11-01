---
title: Setting up a Linux Service Fabric cluster in 2024
slug: linux-service-fabric-cluster
date: 2024-09-28 20:44:00+0200
image: sf-logo.png
links:
  - title: Overview of Service Fabric
    description: Microsoft Learn
    website: https://learn.microsoft.com/azure/service-fabric/service-fabric-overview
    image: https://upload.wikimedia.org/wikipedia/commons/thumb/4/44/Microsoft_logo.svg/240px-Microsoft_logo.svg.png
tags:
    - azure
    - service fabric
    - linux
    - java
categories:
    - Service Fabric
weight: 1
---

**tl;dr**: follow the [Manual installation](https://learn.microsoft.com/en-us/azure/service-fabric/service-fabric-get-started-linux?tabs=sdksetupubuntu%2Clocalclusteroneboxcontainer#manual-installation) steps and ensure that you have `en_US.UTF-8` locale installed (`dpkg-reconfigure locales`)

---

Since I working briefly at a Dutch-Brazilian start-up that ran Service Fabric in production, I have always been curious about it. My first job thereafter was as a platform engineer, specifically focussed on Azure Kubernetes Service; Service Fabric and Kubernetes are often seen as similar, or at lest somewhat comparable.

I have tried a few times since leaving said start-up to setup a Service Fabric cluster myself. Using the managed Azure service works as expected, but is obviously not free. Creating a self-hosted Windows cluster is pretty straightforward; there's an installer and a special tool that runs in the tray which lets you create or destroy a local 1-node or 5-node cluster with just a click

But creating a self-hosted Linux cluster proved to be more challenging.

## The Microsoft documentation

My main issue here is that the documentation is not up to date.

There is of course a page on Microsoft Learn which instructs you in how to setup a Service Fabric cluster on Linux, and if you look at the [Script installation](https://learn.microsoft.com/en-us/azure/service-fabric/service-fabric-get-started-linux?tabs=sdksetupubuntu%2Clocalclusteroneboxcontainer#script-installation) section, you will find a [link to a script hosted on GitHub](https://raw.githubusercontent.com/Azure/service-fabric-scripts-and-templates/master/scripts/SetupServiceFabric/SetupServiceFabric.sh).

Said script contains the following code:

```
Distribution=`lsb_release -cs`
if [%[ "xenial" != "$Distribution" && "bionic" != "$Distribution" ]]; then
    echo "Service Fabric is not supported on $Distribution"
    exit -1
fi
```

This code causes the script to fail with an error if the version of Ubuntu you are running (did I mention you must only use Ubuntu?) is neither _Xenial Xerus_ (16.04) nor _Bionic Beaver_ (18.04). Whilst both being Long-Term Support releases, at the time of writing, both versions are into their final lifecycle phase i.e. _extended security maintenance_ i.e. critical patches only.

This is not what you want for a _new_ production system. Funnily enough, it's also not what Microsoft deliver with their Azure Service Fabric product! Anyway, we know from the Service Fabric release notes that later versions of Ubuntu are supported; versions which are still in the standard support phase. For example, the [release notes for Service Fabric 10.1 CU 4](https://github.com/microsoft/service-fabric/blob/master/release_notes/Service_Fabric_ReleaseNotes_101CU4.md) show that the 'Service Fabric Runtime' package supports only 'Ubuntu 20' and 'Windows'. So, we must surely be able to use Ubuntu 20!

Well, we can. And it's as simple as following the steps listed in [Manual installation](https://learn.microsoft.com/en-us/azure/service-fabric/service-fabric-get-started-linux?tabs=sdksetupubuntu%2Clocalclusteroneboxcontainer#manual-installation) instead. This worked for me with an Ubuntu 20.04 VM.

## Locale issue

Installation instructions were not the only option. It appears that your OS _must_ be configured with the correct locale i.e. `en_US.UTF-8`.

In my case, `servicefabric.service` would not start

```bash
root@ball:/home/ross# systemctl status servicefabric
● servicefabric.service - ServiceFabric Daemon
	Loaded: loaded (/etc/systemd/system/servicefabric.service; enabled; vendor preset: enabled)
	Active: active (running) since Thu 2024-09-26 11:34:45 UTC; 6s ago
	Main PID: 3504 (starthost.sh)
	Tasks: 112 (limit: 3410)
	Memory: 237.1M
	CGroup: /system.slice/servicefabric.service
			├─3504 /bin/bash /opt/microsoft/servicefabric/bin/starthost.sh
			├─3616 /opt/microsoft/servicefabric/bin/FabricHost -c -skipfabricsetup
			├─3669 /usr/bin/dockerd -H unix:///var/run/docker.sock --pidfile /home/sfuser/sfdevcluster/data/_sf_docker_pid/133718240874061580_sfdocker.pid
			├─3697 dotnet FabricCAS.dll
			└─3698 dotnet FabricDCA.dll

Sep 26 11:34:47 ball starthost.sh[3757]: terminating with uncaught exception of type std::runtime_error: collate_byname<char>::collate_byname failed to construct for en_US.UTF-8
Sep 26 11:34:47 ball starthost.sh[3774]: terminating with uncaught exception of type std::runtime_error: collate_byname<char>::collate_byname failed to construct for en_US.UTF-8
Sep 26 11:34:47 ball starthost.sh[3808]: terminating with uncaught exception of type std::runtime_error: collate_byname<char>::collate_byname failed to construct for en_US.UTF-8
Sep 26 11:34:47 ball starthost.sh[3807]: terminating with uncaught exception of type std::runtime_error: collate_byname<char>::collate_byname failed to construct for en_US.UTF-8
Sep 26 11:34:48 ball starthost.sh[3669]: time="2024-09-26T11:34:48.200147262Z" level=info msg="Default bridge (docker0) is assigned with an IP address 172.17.0.0/16. Daemon option --bip can be used to set a p>
Sep 26 11:34:48 ball starthost.sh[3669]: time="2024-09-26T11:34:48.587288757Z" level=info msg="Loading containers: done."
Sep 26 11:34:48 ball starthost.sh[3669]: time="2024-09-26T11:34:48.636215542Z" level=warning msg="WARNING: No swap limit support"
Sep 26 11:34:48 ball starthost.sh[3669]: time="2024-09-26T11:34:48.636374541Z" level=info msg="Docker daemon" commit=662f78c0b1bb5114172427cfcb40491d73159be2 containerd-snapshotter=false storage-driver=overla>
Sep 26 11:34:48 ball starthost.sh[3669]: time="2024-09-26T11:34:48.636459341Z" level=info msg="Daemon has completed initialization"
Sep 26 11:34:48 ball starthost.sh[3669]: time="2024-09-26T11:34:48.732063320Z" level=info msg="API listen on /var/run/docker.sock"
```

The solution was hinted at by [this Autodesk community](https://forums.autodesk.com/t5/eagle-forum/eagle-9-2-error-when-running-on-debian-buster-linux-x64/m-p/8357440/highlight/true#M17666) post (similar [post](https://gitlab.com/pdfgrep/pdfgrep/-/issues/32) for more context).

Basically I had to reconfigure my OS's locales to include `en_US.UTF-8`

This makes at least some sense for me; since I always choose "UK Macintosh" or the equivalently named keyboard layout, the locale is then also set for the UK i.e. `en_GB.UTF-8`.

If you also need to do this, just run `dpkg-reconfigure locales`, selecting the `en_US.UTF-8` locale. I also set is as the default locale, just to be sure, but I don't think that was required; it is probably enough that it's just available locally.

After that, it starts up just fine

```bash
root@ball:/home/ross# systemctl status servicefabric
● servicefabric.service - ServiceFabric Daemon
	Loaded: loaded (/etc/systemd/system/servicefabric.service; enabled; vendor preset: enabled)
	Active: active (running) since Thu 2024-09-26 11:39:20 UTC; 6min ago
	Main PID: 5414 (starthost.sh)
	Tasks: 543 (limit: 3410)
	Memory: 4.9G
	CGroup: /system.slice/servicefabric.service
			├─ 5414 /bin/bash /opt/microsoft/servicefabric/bin/starthost.sh
			├─ 5525 /opt/microsoft/servicefabric/bin/FabricHost -c -skipfabricsetup
			├─ 5578 /usr/bin/dockerd -H unix:///var/run/docker.sock --pidfile /home/sfuser/sfdevcluster/data/_sf_docker_pid/133718243624413540_sfdocker.pid
			├─ 5622 dotnet FabricCAS.dll
			├─ 5623 dotnet FabricDCA.dll
			├─ 5653 /opt/microsoft/servicefabric/bin/Fabric/Fabric.Code/Fabric
			├─ 5673 /opt/microsoft/servicefabric/bin/Fabric/Fabric.Code/Fabric
			├─ 5697 /opt/microsoft/servicefabric/bin/Fabric/Fabric.Code/Fabric
			├─ 5728 /opt/microsoft/servicefabric/bin/Fabric/Fabric.Code/Fabric
			├─ 5731 /opt/microsoft/servicefabric/bin/Fabric/Fabric.Code/Fabric
			├─ 6354 /opt/microsoft/servicefabric/bin/Fabric/Fabric.Code/FabricGateway.exe localhost:35653
			├─ 7198 /opt/microsoft/servicefabric/bin/Fabric/Fabric.Code/FabricGateway.exe localhost:41695
			├─ 7233 /opt/microsoft/servicefabric/bin/Fabric/Fabric.Code/FabricGateway.exe localhost:38963
			├─ 7294 /opt/microsoft/servicefabric/bin/Fabric/Fabric.Code/FabricGateway.exe localhost:45225
			├─ 7355 /opt/microsoft/servicefabric/bin/Fabric/Fabric.Code/FabricGateway.exe localhost:43547
			├─22137 /home/sfuser/sfdevcluster/data/_App/_Node_3/__FabricSystem_App4294967295/FileStoreService.Code.Current/FileStoreService.exe
			├─23576 /home/sfuser/sfdevcluster/data/_App/_Node_0/__FabricSystem_App4294967295/FileStoreService.Code.Current/FileStoreService.exe
			└─24888 /home/sfuser/sfdevcluster/data/_App/_Node_2/__FabricSystem_App4294967295/FileStoreService.Code.Current/FileStoreService.exe

Sep 26 11:39:27 ball starthost.sh[7294]: Start or Close server operation succeeded!
Sep 26 11:39:27 ball starthost.sh[7294]: Fabric Gateway started.
Sep 26 11:39:27 ball starthost.sh[7355]: Starting Fabric Gateway.
Sep 26 11:39:27 ball starthost.sh[7355]: Start or Close server operation succeeded!
Sep 26 11:39:27 ball starthost.sh[7355]: Fabric Gateway started.
Sep 26 11:39:27 ball starthost.sh[7198]: Start or Close server operation succeeded!
Sep 26 11:39:27 ball starthost.sh[7198]: Fabric Gateway started.
Sep 26 11:40:01 ball starthost.sh[5697]: Failover Manager is ready.
Sep 26 11:40:36 ball usermod[22093]: add 'root' to group 'ServiceFabricAllowedUsers'
Sep 26 11:40:36 ball usermod[22093]: add 'root' to shadow group 'ServiceFabricAllowedUsers'
```

## Conclusion

So there you go; two gotchas to bear in mind when creating a Linux Service Fabric cluster.

Happy hacking!

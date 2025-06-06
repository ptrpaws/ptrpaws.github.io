---
title: Meta Quest Firmware Archive & How it works
layout: post
description: A detailed guide on the Meta Quest Firmware Archive and how it works.
tags: vr virtual-reality oculus-quest meta firmware cocaine.trade website
---

# Introducing the Quest Firmware Archive

I'm happy to announce the launch of a new website, [cocaine.trade](https://cocaine.trade/), which hosts an extensive collection of update .zip files for Meta VR headsets. As of July 30th, the archive includes firmware versions for various Meta VR headsets. Specifically, there are 53 Quest 1 firmware versions ranging from v3 to v50, 86 Quest 2 firmware versions from v21 to v56, and 15 Quest Pro firmware versions from v50 to v56. This archive aims to provide Meta VR headset owners with easy access to firmware updates and historical versions.

![Screenshot of the Website](/assets/images/posts/archive_screenshot.png)
*Image: Screenshot of the Website*

## How to Manually Sideload a Quest .zip Update from the Archive

Before attempting to manually update your Quest VR headset, it's essential to ensure that the update you are trying to install is either a newer version or the same version as your current firmware. Otherwise, the update will fail. Additionally, make sure that developer mode is enabled on your Quest to perform sideloading.

A complete tutorial from me will follow, but until then you can use this [reddit tutorial on how to force an update](https://www.reddit.com/r/oculusquest/wiki/guides/manualupdate/)

## How Firmware Archiving Works

Now onto something fun: Here is how we obtain the .zip firmware updates in the first place in order to put them on the archive.

Archiving the firmware for Meta VR headsets can be a challenging task due to various factors, including Quests auto updating and the existence of partial updates, which are pretty much useless for analysis. Most .zip files on the web originate in methods like running `adb logcat | grep "initial url:"` to find URLs ending in .zip, or using `adb bugreport` during an update to obtain the URL from the generated bug report zip.

### Obtaining a DAT

For Quest 1 and Quest 2 devices, there is an easier way to grab firmware update files, thanks to an exploit discovered indipendantly by myself and Samulia. This method requires your Quest to be on a version lower than v38, as the exploit was patched in subsequent versions. The trick involves setting a proxy via adb and then corrupting the SSL to capture the device access token (DAT) from `adb logcat` output on the update settings page. Before v38, the DAT was stored in URL parameters and thus was logged by okhttp in case of an error. On newer versions the DAT is stored in the headers which aren't logged by okhttp.

Another way to acquire a DAT is by using a rooted Quest (make sure it isn't a prototype, since they most likely don't have a device identity injected), which allows you to generate a new DAT or retrieve the one generated by the OSUpdater from the file system. It's important to note that DATs are only valid for approximately 24 hours, so use them quickly.

For Quest Pro devices, since we lack a reliable exploit to gain a DAT, Samulia developed a workaround to trick the Quests into always requesting full firmware instead of partial firmware. However, to avoid this from being patched, I have decided not to disclose the details here.

### Using the DAT to download the firmware

To download Quest firmware from Meta directly, we can use one of the available API endpoints. Below is an example of using curl to download the firmware from the `mobile_release_updates` graphql endpoint, but there are other outdated ones. Ensure you replace {INSERT YOUR DAT HERE} with your DAT in the curl request and update the device type (e.g., `vr_monterey` for Quest 1, `hollywood` for Quest 2, and `seacliff` for Quest Pro). You can also try to replace `user` with `userdebug`. These builds are usually rooted and meant for Meta employees, but we have been able to pull them from this api before on some occasions.

```bash
curl --location --globoff 'https://graph.oculus.com/mobile_release_updates?access_token=OC%7C3733290306686872%7C&device_managed_mode=0&channel_app_id=399374017083309&fields=update_interval%2Cota.device_type(ota.hollywood.user).device_serial(0).sdk_version(0).version(50600670029600150).security_patch_time(2021-04-05){download_uri%2Ctarget_version%2Cbase_version%2Cinstall_options%2Cfile_checksum%2Crelease_channel_id%2Crelease_channel_name}&device_access_token={INSERT YOUR DAT HERE}'
```

Please note that the DAT determines which release channel you get. If the DAT comes from a Quest with PTC (Public Test Channel) enabled, you will only get PTC firmware. Without PTC, you will most likely only receive release firmware.

## Conclusion

The Meta Quest Firmware Archive at [cocaine.trade](https://cocaine.trade/) is a valuable resource for looking up information about various firmware versions conveniently, be that to analyse them, update to a specific version, before updates are available or even downgrade in the future if unlocked Quests become a thing.

Firmware archiving is an ongoing process, and I will continue to update the archive with new firmware versions as they are released and if you stumble across missing .zip files feel free to shoot me a dm on Twitter @ptrpaws

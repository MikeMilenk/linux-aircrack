# linux-aircrack
This guide shows how to crack a **WPA2 Wi-Fi password** using an offline dictionary attack with the **Aircrack-ng** suite and a wireless adapter that supports monitor mode. I used the **Alfa AWUS036ACM**.

I also created a separate, isolated Wi-Fi network called **Papa Tango** (military-style wording for “Penetration Testing”) specifically for testing.

![Separate test network](https://github.com/MikeMilenk/linux-aircrack/blob/0bcdf262ddbc4948a7fd458fa6132c4c9d4dd5f1/images/0.0%20-%20Create%20testing%20network.png)

While testing both frequency bands, I ran into some extra challenges with 5GHz. So I decided to split the guide into 2 parts: **2.4 GHz** and **5 GHz**.

> Do not use them against networks you do not control!

---

## Table of Contents

- [0. Adapter Setup](#0-adapter-setup)
- [1. Enable Monitor Mode](#1-enable-monitor-mode)
- [2. Find the Target Network](#2-find-the-target-network)
- [3. Capture 2.4 GHz Traffic to a File](#3-capture-24-ghz-traffic-to-a-file)
- [4. Force a Handshake with Deauthentication](#4-force-a-handshake-with-deauthentication)
- [5. Dictionary Attack with Aircrack-ng](#5-dictionary-attack-with-aircrack-ng)
- [6. Capture 5GHz Traffic to a File](#6-capture-5ghz-traffic-to-a-file)
   - [6.1 Initial Issue](#61-initial-issue)
   - [6.2 Standard 5GHz Traffic Capture](#62-standard-5ghz-traffic-capture)
   - [6.3 Why Deauthentication Wasn’t Working](#63-why-deauthentication-wasnt-working)
   - [6.4 Country Restrictions](#64-country-restrictions)
   - [6.5 Restricted Channels on the 2.4GHz Network](#65-restricted-channels-on-the-24ghz-network)
 - [7. Important Notes](#7-important-notes)
 - [8. What About WPA3?](#8-what-about-wpa3)


---

## 0. Adapter Setup
Shut down your VM and plug in your new USB Wi-Fi adapter. To make it available inside the VM, you need to identify it on the host (Proxmox, in my case) and pass it through to the VM. To identify adapter, run:

```bash
lsusb
```
![Identify adapter on host](https://github.com/MikeMilenk/linux-aircrack/blob/0bcdf262ddbc4948a7fd458fa6132c4c9d4dd5f1/images/0.1%20-%20Locate%20Adapter%20on%20host.png)

I pass it to the VM directly through the Proxmox GUI:
- **VM → Hardware → Add → USB Device → Use USB Vendor/Device ID → *Select your adapter* → Add.**

![Add usb to a VM 1](https://github.com/MikeMilenk/linux-aircrack/blob/0bcdf262ddbc4948a7fd458fa6132c4c9d4dd5f1/images/0.2.1%20-%20Add%20usb%20to%20a%20VM.png)

![Add usb to a VM 2](https://github.com/MikeMilenk/linux-aircrack/blob/0bcdf262ddbc4948a7fd458fa6132c4c9d4dd5f1/images/0.2.2%20-%20Add%20usb%20to%20a%20VM.png)

The adapter should now appear in the VM's USB device list and be accessible inside the guest OS (Kali, in my case).

![New usb added](https://github.com/MikeMilenk/linux-aircrack/blob/0bcdf262ddbc4948a7fd458fa6132c4c9d4dd5f1/images/0.3%20-%20New%20usb%20added.png)

> My **Alfa AWUS036ACM** uses the MediaTek MT7612U chipset. On current Kali the **mt76x2u** driver is already in the kernel. so no external compilation needed.
> If you're using a different OS, visit the official [Alfa website](https://docs.alfa.com.tw/Product/AWUS036ACM/) to download the required drivers.

Start VM and check if the new adapter is detected:
```bash
lsusb
```
![Identify adapter on VM](https://github.com/MikeMilenk/linux-aircrack/blob/0bcdf262ddbc4948a7fd458fa6132c4c9d4dd5f1/images/0.4%20-%20Locate%20Adapter%20on%20VM.png)

If you're using a different Linux distribution, such as Ubuntu, check whether the required Wi-Fi driver module is loaded.

```bash
lsmod | grep mt76
```

![Check driver module on VM](https://github.com/MikeMilenk/linux-aircrack/blob/0bcdf262ddbc4948a7fd458fa6132c4c9d4dd5f1/images/0.5%20-%20Check%20Module%20on%20VM.png)

And load them if needed:
```bash
sudo modprobe mt76x2u
```

---

## 1. Enable Monitor mode

Check current mode:

```bash
iwconfig
```
You should now see your wireless interface in **Managed Mode**. We need to switch it to **Monitor Mode**.

![Check adapter mode](https://github.com/MikeMilenk/linux-aircrack/blob/0bcdf262ddbc4948a7fd458fa6132c4c9d4dd5f1/images/1.1%20-%20Check%20mode.png)

Before proceeding, make sure you have the necessary tools installed. I use **Aircrack-ng**, a suite of tools for Wi-Fi security assessment and auditing. It allows you to monitor wireless networks, capture Wi-Fi traffic, test wireless security, and perform password attacks against captured WPA/WPA2 handshakes. It's commonly used in Kali Linux for authorized Wi-Fi security testing and pentesting.

**Aircrack-ng** is usually pre-installed in Kali. To check if it's installed and to see its version, run:

```bash
apt show aircrack-ng
```

![Check Aircrack version](https://github.com/MikeMilenk/linux-aircrack/blob/d246afce55616abf2c08967f48d7e1a2fdefa37f/images/1.2%20-%20Check%20Aircrack%20version.png)

If it's not installed, run the following commands:

```bash
sudo apt update
sudo apt install aircrack-ng
```

Now lets switch it to **Monitor Mode**:

```bash
sudo airmon-ng check kill
```
This command checks for and terminates processes that may interfere with Monitor Mode. This helps prevent conflicts when switching adapter to **Monitor Mode**.

```bash
sudo airmon-ng start wlan0
```

![Start monitor mode](https://github.com/MikeMilenk/linux-aircrack/blob/d246afce55616abf2c08967f48d7e1a2fdefa37f/images/1.3%20-%20Start%20monitor%20mode.png)

The interface name has changed from **wlan0** to **wlan0mon**. Use **wlan0mon** for subsequent commands that require Monitor Mode.

Verify:
```bash
iwconfig
```

![Verify mode](https://github.com/MikeMilenk/linux-aircrack/blob/d246afce55616abf2c08967f48d7e1a2fdefa37f/images/1.4%20-%20Verify%20mode.png)

The interface should now appear as **wlan0mon** (or similar) in **Monitor mode**.

## 2. Find the target network
```bash
sudo airodump-ng wlan0mon
```

![See all 2.4 GHz networks](https://github.com/MikeMilenk/linux-aircrack/blob/d246afce55616abf2c08967f48d7e1a2fdefa37f/images/2.1%20-%20See%20all%202%2C4%20GHz%20networks.png)

Look for:

- **BSSID** (MAC of the access point)
- **CHANNEL** (Channel 1, in my case)
- **STATION** (connected clients)

> For privacy reasons, I've hidden all detected Wi-Fi networks and connected devices in the screenshot, except for my test network, which remains visible for demonstration purposes.

> **NOTE:** I noticed that `airodump-ng` was scanning only the 2.4 GHz band (CH 1 to 11), while my phone was connected to a 5 GHz network. Here I encountered some additional challenges when testing 5 GHz networks. Therefore, I've decided to split this guide into 2 separate parts: **2.4 GHz** and **5 GHz**. 

To explicitly scan both bands, specify the `--band` parameter (my adapter doesn't support 6 GHz):

```bash
sudo airodump-ng --band abg wlan0mon
```

The `abg` parameter enables scanning of both supported frequency bands:
- `a` — 5 GHz
- `b/g` — 2.4 GHz

![See all band networks](https://github.com/MikeMilenk/linux-aircrack/blob/d246afce55616abf2c08967f48d7e1a2fdefa37f/images/2.2%20-%20See%20all%20band%20networks.png)

Cancel with Ctrl+C when ready.

## 3. Capture 2.4 GHz traffic to a file

Lock airodump to the target network so the screen stays clean and the client list remains visible:
```bash
sudo airodump-ng -c <CHANNEL> --bssid <BSSID> -w capture wlan0mon
```
Example:
```bash
sudo airodump-ng -c 11 --bssid 8A:D3:43:01:D1:DD -w TestingWPA2 wlan0mon
```
![Capture traffic](https://github.com/MikeMilenk/linux-aircrack/blob/c9b3b9556eecb4d9212ef41050a726bd9bd91f08/images/3.1%20-%20Capture%202.4GHz%20%20traffic.png)

Leave this terminal running. It will write files such as *TestingWPA2-01.cap*

## 4. Force a handshake with deauthentication

Open a second terminal and send deauth packets to a connected client (or broadcast):
```bash
sudo aireplay-ng --deauth 0 -a <BSSID> -c <CLIENT_MAC> wlan0mon
```

`--deauth 0` = keep sending until you stop it
`-a` = access point BSSID
`-c` = client MAC (recommended)

![Deauth 2.4GHz](https://github.com/MikeMilenk/linux-aircrack/blob/c9b3b9556eecb4d9212ef41050a726bd9bd91f08/images/3.2%20-%20Deauth%202.4GHz.png)

After several deauth packets, the client will reconnect and generate a 4-way handshake. Stop both tools (Ctrl+C).

## 5. Dictionary attack with aircrack-ng

After stopping the capture process, several files were created in our home directory. These files contain the captured handshake data and related information.

![Capture files](https://github.com/MikeMilenk/linux-aircrack/blob/c9b3b9556eecb4d9212ef41050a726bd9bd91f08/images/4%20-%20Capture%20files.png)

They were saved using the filename prefix specified with the `-w` parameter. We specifically need the `.cap` file, as it contains the captured Wi-Fi packets, including the WPA2 handshake.

```bash
aircrack-ng TestingWPA2-01.cap -w /usr/share/wordlists/rockyou.txt
```
`-w` specifies the wordlist. Without `-w` aircrack-ng will attempt a much slower pure brute-force

![Cracking 2.4 password](https://github.com/MikeMilenk/linux-aircrack/blob/c9b3b9556eecb4d9212ef41050a726bd9bd91f08/images/5.1%20-%20Cracking%202.4%20password.png)

**Aircrack-ng** started the offline password cracking process, and CPU usage began increasing significantly.
If the password exists in the wordlist, aircrack-ng will display:
```text
KEY FOUND! [ the_password ]
```
If the password is not in the list, you will eventually see `KEY NOT FOUND`. In my case, it took about 10 min to find the password using the **rockyou.txt** wordlist.

![Cracked 2.4 password](https://github.com/MikeMilenk/linux-aircrack/blob/c9b3b9556eecb4d9212ef41050a726bd9bd91f08/images/5.2%20-%20Cracked%202.4%20password.png)


> A “strong” looking password (mixed case + numbers + symbols) can still be cracked quickly if it appears in a common wordlist such as `rockyou.txt`.

If you want to revert the adapter from `Monitor mode` back to `Managed mode` (normal Wi-Fi connection mode), just run:
```bash
sudo airmon-ng stop wlan0mon
```

![Disable Monitor Mode](https://github.com/MikeMilenk/linux-aircrack/blob/fcd1c1b5655e9ad7727216a6398375d5b5e4c52e/images/5.3%20-%20Disable%20Monitor%20mode.png)

The interface goes back to `wlan0` instead of `wlan0mon`.


## 6. Capture 5GHz traffic to a file

### 6.1 Initial Issue

Initially, my 5GHz channel was `112`. The problem was that I couldn't deauthenticate the device from the access point at all - it just sent back zero in the response, and on my phone I could see that the connection wasn't dropping (which was supposed to happen).

### 6.2 Standard 5GHz Traffic Capture

After a few days I repeated the test. At this time the channel was `149`, everything worked as expected, same as with the 2.4GHz network.

1) Scanning for the target network using **airodump-ng** and remembering its **channel** with **BSSID**.

   ![Listening all 5GHz network](https://github.com/MikeMilenk/linux-aircrack/blob/389f1a64f7481a5099dd0969d59d040b3bf03307/images/6.1%20-%20Listen%20all%205GHz%20network.png)

2) Starting a targeted capture on that channel/BSSID, then deauthenticating a connected client with **aireplay-ng** to force a fresh WPA handshake.

   ![Capturing 5GHz handshake](https://github.com/MikeMilenk/linux-aircrack/blob/389f1a64f7481a5099dd0969d59d040b3bf03307/images/6.2%20-%20Capture%205GHz%20handshake.png)

3) Running an offline dictionary attack against the captured handshake using **aircrack-ng** with the **rockyou.txt** wordlist.

   ![Finding 5GHz password](https://github.com/MikeMilenk/linux-aircrack/blob/389f1a64f7481a5099dd0969d59d040b3bf03307/images/6.3%20-%20Found%205GHz%20password.png)
 

### 6.3 Why Deauthentication Wasn’t Working

So I decided to figure out whether my adapter could actually inject packets on channel `112` at all, since deauth wasn’t working there. So I ran:

```bash
sudo iw dev wlan0mon set channel 112
sudo aireplay-ng --test wlan0mon
```

This  tests packet injection, separate from just scanning/receiving, which is what showed that channel `112` was blocked for transmission, while channel `149` was injecting as expected.

![Testing injection on different channels](https://github.com/MikeMilenk/linux-aircrack/blob/4f54f23412022baca6c40a36d275d035d763a6ac/images/6.3.1%20-%20testing%20different%20channels.png)

After a bit of googling, I found out that there are **DFS** channels. These exist because part of the 5GHz band overlaps with frequencies reserved for weather and aviation radar — devices are required to detect radar signals and avoid interfering with them before transmitting on these channels. That's why my adapter can see these channels, but can’t inject packets on them. Packet injection on **DFS channels** requires certified radar-detection support in the driver/firmware, which most open-source Linux Wi-Fi drivers, including mine, don’t implement. So they block transmission there to stay compliant with regulations. I decided to check for myself which channels my adapter could and couldn’t inject on:

```bash
iw list
```

![5GHz radar channels](https://github.com/MikeMilenk/linux-aircrack/blob/389f1a64f7481a5099dd0969d59d040b3bf03307/images/6.4%20-%205GHz%20radar%20channels.png)

In the output of `iw list`, under **Band 2 → Frequencies**, you can see which channels are marked with **No IR** and **Radar detection** — these are the channels where packet injection (and therefore deauthentication) isn't possible. During my 1st attempt, the 5GHz network happened to be on channel 112, which was exactly one of these restricted channels.

### 6.4 Country Restrictions

Then I checked which regulatory domain the adapter was using and whether it was unnecessarily restricting which channels I could transmit on.

```bash
iw reg get
```

The country was set to `00-UNSET` — meaning the adapter was running under the most restrictive, generic international configuration, with a lot of channels locked to `PASSIVE-SCAN` only (receive-only, no transmission at all). I set the region to Canada with:

```bash
sudo iw reg set CA
```

![Changing country code](https://github.com/MikeMilenk/linux-aircrack/blob/b430a344f420ccace73b0ad2a63c6b1e0f733774/images/6.5%20-%20Changing%20country%20code.png)

I did this because the default ``00`` region is conservative. It doesn’t know what country’s regulations actually apply, so it locks down transmission on anything it isn’t sure is legal. Setting the correct region (CA) lets the driver apply Canada’s actual regulatory rules instead, which unlocks transmission on channels that are legal in Canada. Switching to the CA region also bumped the allowed TX power on some non-DFS channels, but it didn't unlock any new non-DFS channels that weren't already available.

In this situation, my adapter with *mt76x2u* driver doesn’t support packet injection on DFS channels, but this is possible with other certified adapters. In my case, I just waited, either for the network to switch to a different non-DFS channel on its own, or for the client to naturally reconnect to the network at some point, so I could passively catch the handshake when that happened.

### 6.5 Restricted Channels on the 2.4GHz Network

I ran `iw list` again and checked the **2.4GHz** channels on **Band 1 → Frequencies**, and saw some `No IR` channels there too. Channels `12–14` are marked that way because they fall outside the 1–11 range allowed in North America — they're used in other regions instead (12–13 in Europe, 14 in Japan) - so transmission on them isn't permitted under the CA regulatory domain the adapter is set to.

![2.4GHz radar channels](https://github.com/MikeMilenk/linux-aircrack/blob/c3745aba3867b8450cb5c5b08c48ebb43b98e30b/images/6.6%20-%202.4GHz%20radar%20channels.png)

## 7. Important Notes

1) **Capture quality matters** — if the handshake isn’t captured completely (for example, it gets cut off mid-way due to a weak signal or bad timing during deauth), **aircrack-ng** won’t be able to verify it, even with the correct password in the wordlist. You need the full 4-way handshake.

2) **VPN on client doesn’t block capture** — the handshake happens at Layer 2 (connecting to Wi-Fi), before IP even comes up, let alone a VPN tunnel. VPN only encrypts application traffic after the connection is already established, it has no effect on the handshake itself.

3) **PMF can weaken deauth** — if _Protected Management Frames_ (PMF/802.11w) is enabled on the access point, spoofed deauth frames get ignored by the client, and the attack may simply not work, regardless of channel or adapter. PMF is optional on WPA2 but mandatory on WPA3, so it’s worth checking for either.

4) **Auto-channel and MAC randomization** — routers can switch channels mid-session to avoid interference, which can invalidate an ongoing capture, and modern devices often randomize their MAC address per session, so a client MAC noted earlier may already be stale.

5) **Client in power-saving mode** — some devices respond less reliably to deauth when the screen is off/sleeping, due to Wi-Fi power-saving policies.

## 8. What about WPA3?

**What about WPA3**

**WPA3** doesn't use a 4-way handshake like WPA2, it uses **SAE** (Simultaneous Authentication of Equals, based on the **Dragonfly key exchange**). Each password attempt requires live interaction with the Access Point in real time, so there's no static exchange you can capture once and crack offline afterward. Combined with PMF being mandatory, this makes WPA3 resistant to the deauth packets injection + offline-crack chain that works against **WPA** or **WPA2**.

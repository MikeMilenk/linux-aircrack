# linux-aircrack
This guide shows how to crack a **WPA2 password** using a dictionary attack with the **Aircrack-ng** suite and a wireless adapter that supports monitor mode. I used the **Alfa AWUS036ACM**.

While testing both frequency bands, I ran into some extra challenges with 5 GHz. So I decided to split the guide into 2 parts: **2.4 GHz** and **5 GHz**.

I also created a separate, isolated Wi-Fi network called **Papa Tango** (military-style wording for “Penetration Testing”) specifically for testing.

![Separate test network](https://github.com/MikeMilenk/linux-aircrack/blob/0bcdf262ddbc4948a7fd458fa6132c4c9d4dd5f1/images/0.0%20-%20Create%20testing%20network.png)

> Do not use them against networks you do not control!

---

## Table of Contents

0. [Adapter Setup](#0-adapter-setup)
1. [Enable Monitor mode](#1-enable-monitor-mode)
2. [Find the target network](#2-find-the-target-network)
3. [Capture traffic to a file](#3-capture-traffic-to-a-file)
   - [3.1 Capture 2.4 GHz traffic](31-capture-2.4-ghz-traffic)
5. [Force a handshake with deauthentication](#4-force-a-handshake-with-deauthentication)
6. [Dictionary attack with aircrack-ng](#5-dictionary-attack-with-aircrack-ng)
7. [Quick command reference](#quick-command-reference)
8. [Notes from the lab](#notes-from-the-lab)

---

## 0. Adapter Setup
Shut down your VM and plug in your new USB Wi-Fi adapter. To make it available inside the VM, you need to identify it on the host (Proxmox, in my case) and pass it through to the VM. To identify adapter, run:

```bash
lsusb
```
![Identify adapter on host](https://github.com/MikeMilenk/linux-aircrack/blob/0bcdf262ddbc4948a7fd458fa6132c4c9d4dd5f1/images/0.1%20-%20Locate%20Adapter%20on%20host.png)

I pass it to the VM directly through the Proxmox GUI:
**VM → Hardware → Add → USB Device → Use USB Vendor/Device ID → *Select your adapter* → Add.**
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
The interface name has changed from **wlan0** to **wlan0mon**. Use **wlan0mon** for subsequent commands that require Monitor Mode.

Verify:
```bash
iwconfig
```
The interface should now appear as wlan0mon (or similar) in Monitor mode.

## 2. Find the target network
```bash
sudo airodump-ng wlan0mon
```

Look for:

- **BSSID** (MAC of the access point)
- **CHANNEL** (Channel 1, in my case)
- **STATION** (connected clients)

> For privacy reasons, I have hidden all detected Wi-Fi networks and connected devices in the screenshot, except for my test network, which remains visible for demonstration purposes.

> **NOTE:** I noticed that `airodump-ng` was scanning only the 2.4 GHz band (CH 1 to 11), while my phone was connected to a 5 GHz network. Here I encountered some additional challenges when testing 5 GHz networks. Therefore, I've decided to split this guide into 2 separate parts: **2.4 GHz** and **5 GHz**. 

To explicitly scan both bands, specify the `--band` parameter (my adapter doesn't support 6 GHz):

```bash
sudo airodump-ng --band abg wlan0mon
```

The `abg` parameter enables scanning of both supported frequency bands:
- `a` — 5 GHz
- `b/g` — 2.4 GHz

Cancel with Ctrl+C when ready.

## 3. Capture traffic to a file

### 3.1 Capture 2.4 GHz traffic

Lock airodump to the target network so the screen stays clean and the client list remains visible:
```bash
sudo airodump-ng -c <CHANNEL> --bssid <BSSID> -w capture wlan0mon
```
Example:
```bash
sudo airodump-ng -c 11 --bssid 8A:D3:43:01:D1:DD -w TestingWPA2 wlan0mon
```
Leave this terminal running. It will write files such as *TestingWPA2-01.cap*

## 4. Force a handshake with deauthentication

Open a second terminal and send deauth packets to a connected client (or broadcast):
```bash
sudo aireplay-ng --deauth 0 -a <BSSID> -c <CLIENT_MAC> wlan0mon
```

`--deauth 0` = keep sending until you stop it
`-a` = access point BSSID
`-c` = client MAC (recommended)

After several deauth packets, the client will reconnect and generate a 4-way handshake. Stop both tools (Ctrl+C).

## 5. Dictionary attack with aircrack-ng

After stopping the capture process, several files were created in our home directory. These files contain the captured handshake data and related information. They were saved using the filename prefix specified with the `-w` parameter. We specifically need the `.cap` file, as it contains the captured Wi-Fi packets, including the WPA2 handshake.

```bash
aircrack-ng TestingWPA2-01.cap -w /usr/share/wordlists/rockyou.txt
```
`-w` specifies the wordlist. Without `-w` aircrack-ng will attempt a much slower pure brute-force

**Aircrack-ng** started the offline password cracking process, and CPU usage began increasing significantly.
If the password exists in the wordlist, aircrack-ng will display:
```text
KEY FOUND! [ the_password ]
```
If the password is not in the list, you will eventually see `KEY NOT FOUND`.



> A “strong” looking password (mixed case + numbers + symbols) can still be cracked quickly if it appears in a common wordlist such as `rockyou.txt`.

Capture quality matters: you need a complete 4-way handshake.

VPN on the client does not prevent handshake capture (it starts after Layer 2).

Protected Management Frames (PMF / 802.11w) can make deauth less effective.

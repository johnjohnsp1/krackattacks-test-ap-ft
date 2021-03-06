This script tests if APs are affected by CVE-2017-13082 (KRACK attack). See [the KRACK attack website for details](https://www.krackattacks.com) and also read [the research paper](https://papers.mathyvanhoef.com/ccs2017.pdf).

# CVE-2017-13082: Key Reinstall in FT Handshake (802.11r)

Access Points (APs) might contain a vulnerable implementation of the Fast BSS Transition (FT) handshake. More precisely, a retransmitted or replayed FT Reassociation Request may trick the AP into reinstalling the pairwise key. If the AP does not process retransmitted FT reassociation requests, or if it does not reinstall the pairwise key, it is not vulnerable. If it does reinstall the pairwise key, the effect is similar to the attack against the 4-way handshake, except that the AP instead of the client is now reinstalling a key. More precisely, the AP will subsequently reuse packet numbers when sending frames protected using TKIP, CCMP, or GCMP. This causes nonce reuse, voiding any security these encryption schemes are supposed to provide. Since the packet number is also used as a replay counter for received frames, frames sent *towards* the AP can also be replayed.

In contrast to the 4-way handshake and group key handshake, this is not an attack against the specification. That is, if the state machine as shown in Figure 13-15 of the 802.11-2016 standard is faithfully implemented, the AP will not reinstall the pairwise keys when receiving a retransmitted FT Reassociation Request. However, we found that many APs do process this frame and reinstall the pairwise key.


# Script Usage Instructions

We created scripts to determine whether an implementation is vulnerable to any of our attacks. These scripts were tested on Kali Linux. To install the required dependencies on Kali, execute:

	apt-get update
	apt-get install libnl-3-dev libnl-genl-3-dev pkg-config libssl-dev net-tools git sysfsutils python-scapy python-pycryptodome

Remember to disable Wi-Fi in your network manager before using our scripts. After doing so, execute `sudo rfkill unblock wifi` so our scripts can still use Wi-Fi.

The included Linux script `krack-ft-test.py` can be used to determine if an AP is vulnerable to our attack. The script contains detailed documentation on how to use it:

	./krack-ft-test.py --help

Essentially, it wraps a normal `wpa_supplicant` client, and will keep replaying the FT Reassociation Request (making the AP reinstall the PTK). We tested the script on a Kali Linux distribution using a USB WiFi dongle (a TP-Link WN722N v1).

Remember that this is not an attack script! You require credentials to the network in order to test if an access point is affected by the attack.


# Suggested Solution

If the implementation is vulnerable, the suggested fix is similar to the one of the 4-way handshake. That is, a boolean can be added such that the first FT Reassociation Requests installs the pairwise keys, but any retransmissions will skip key installation. Note that ideally the AP should still send a new FT Reassociation Response, even though it did not reinstall any keys.


# Impact and Exploitation Details

Exploiting this vulnerability does not require a man-in-the-middle position! Instead, an adversary merely needs to capture a Fast BSS Transition handshake and save the FT Reassociation Request. Because this frame does not contain a replay counter, the adversary can replay it at any time (and arbitrarily many times). Each time the vulnerable AP receives the replayed frame, the pairwise key will be reinstalled. This attack is illustrated in Figure 9 of the paper.

An adversary can trigger FT handshakes at will as follows. First, if no other AP of the network is within range of the client, the adversary clones a real AP of this network next to the client using a wormhole attack (i.e. we forward all frames over the internet). The adversary then sends a BSS Transition Management Request to the client. This request commands to the client to roam to another AP. As a result, the client will perform an FT handshake to roam to the other AP.

The included network trace [example-ft.pcapng](example-ft.pcapng) is an example of the attack executed against Linux's hostapd. When using the wireshark filter `wlan.sa == 7e:62:5c:7a:cd:47`, notice that packets 779 to 1127 all use the CCMP IV value 1. This was caused by malicious retransmissions of the FT reassociation request.

# Extra: Ubuntu 16.04

This tool is officially only supported on Kali Linux. Nevertheless, some users have been able to get it running on Ubuntu 16.04. These users remarked that the `python-pycryptodome` package is not present on Ubuntu, but can be installed as follows:

1. Install python-pip package
2. Execute `pip install pycryptodomex`

They further recommended to install this python module under a virtual python environment using virtualenv.


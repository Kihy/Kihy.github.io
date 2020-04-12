---
layout: post
title:  "Investigation of CIC IDS Dataset"
author: "Ke He"
description: "Investigation of why it doesn't match our experimental setup"
---

### Wireshark format

Wireshark actually has different pcap formats: tcpdump, modified tcpdump, nano seconds, red hat, etc. All of the above formats have extension pcap, and saving it in the wrong format could cause different flows being extracted from the data. The above formats were all tried, but the output of our classifier is the same (red hat and suse formats gave no output from CICFlowMeter).

### Flow examination

Upon examining the extracted features, it was found that flow in the CIC dataset that are labelled as FTP patator(or any other attacks) has forward and backward packet number of 1 (most of the time, with occasional 3). For our attack, each flow has around a total of 28 packets in both direction (most of the time). This causes the statistics of each flow being a number of times larger. A flow is a sequence of packets with the same dest IP, source IP, dest port, source port and protocol. The TCP stream of our attack traffic shows the patator program guesses the password 3 times then changes to a new TCP stream. So making the patator program change port after each guess could solve our issue.

The parameters persistent and max-retries has been set to false and 1 respectively in order to try one password at a time. However when examining the TCP conversation in wireshark, the username and password are specified in separate requests:

    220 (vsFTPd 2.3.4)
    USER msfadmin
    331 Please specify the password.
    PASS internet
    530 Login incorrect.

This means each full attempt has to have at least 5 packets in the flow.

My guess is that the attempts in the CIC dataset have been detected and blocked, so that only the request is captured and no following responses asking for password. Or there is something different with wireshark or the feature extractor.

### Own dataset

Guess its time to make our own dataset with patator and benign traffic. RIP

---
layout: post
title:  "Hulk and patator"
author: "Ke He"
description: "Analyzing DoS Hulk and Patator attacks"
---

### Hulk

As mentioned in the last blog post, hulk attack conducted in our virtual environment was not classified correctly by our classifier, it was classified as portscan. To further analyse the reason behind it, histograms were drawn to visualize the distributions of real hulk attack and the ones given in the dataset(As a side note not all connections during a hulk attack is malicious, some of the connections are indeed benign, which means our real attack could include some BENIGN samples but in the grand scheme of things they are minor).

From the histogram it was clear that there are differences between real and sample hulk attack, a significant difference are the length of packets and number of ACK packets. The Hulk attack was supposed to use multiple ports of the attacker machine and bind them to https port of the target, however in our attack it oscillates between 2 ports on the attacker's machine, meaning the webserver has innate mitigations and drops connections which causes our hulk attack to have different attack patterns. Need to investigate hulk attack more.

### FTP Patator

The FTP patator brute forces passwords. A sample password list of top 10 million passwords are downloaded and the ftp patator on kali is used to brute force ftp account of msfadmin. To shorten the time it took, msfadmin is put as 7th password to try and it was left running for a few minutes after the password was found.

The FTP patator data is still unable to be classified, all samples are classified as BENIGN. A key difference between the real and sample data is that the ACK flag count in all samples (all 15 classes) is 0~1 while in real the flag counts range from 0~30. Which again, needs more investigation.

Have a funny feeling the CIC flow meter used to generate 2017 data is outdated, since the first commit on github is around Feb 2018. Maybe try downloading CIC2018 data or just make our own data.

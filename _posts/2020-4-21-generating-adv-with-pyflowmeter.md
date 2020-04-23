---
layout: post
title:  "Generating slowloris part 2"
author: "Ke He"
description: "Creating adversarial examples of slowloris attack with pyflowmeter"
---

## PyFlowMeter

PyFlowMeter is our very own feature extractor, features extracted are based on CICFlowMeter. The possible features are:

| feature name                | description                                       | number of features |
| --------------------------- | ------------------------------------------------- | ------------------ |
| duration                    | total duration of flow                            | 1                  |
| protocol                    | protocol used for the flow                        | 1                  |
| {dst,src}\_port             | destination and source port number                | 2                  |
| {fwd,bwd}\_tot\_{pkt,byte}  | total number of forward/backward packet and bytes | 2 \* 2             |
| {fwd,bwd}\_pkt_size\_{stat} | distribution of fwd/bwd packet size               | 2 \* 6             |
| {fwd,bwd}\_iat\_{stat}      | distribution of fwd/bwd inter arrival time        | 2 \* 6             |
| {fwd,bwd}\_{flags}\_cnt     | number of packets with various flags              | 2 \* 8             |
| total                       | total number of features                          | 48                 |

## difference of extracted features

The PyFlowmeter is based on pyshark, and thus have the right number of flows. Compared to CICFlowMeter, it has made what should be a single flow into two different flows due to packets inter arrival time being too long. The counts of extracted features from pyflowmeter are:

| all | samples |
| --- | ------- |
| 0   | 605     |
| 1   | 150     |

## Adversarial generation

First, an unconstrainted adversarial sample is generated, and similar to results from CICFlowMeter, it wants to change the flag counts, which is not very helpful.

The next batch was set to fix all flag counts, and it shows it wants to change protocol to UDP and backward iat. protocol is one of the feature that defines slowloris, since without TCP there wouldn't be a connection, and bwd iat is not something the attacker can control, it is based on the server. Thus we fix the protocol field and all fields related to bwd.

Now the perturbation file shows something we can change, it shows reducing fwd packet skewness by -3 and increase minimum forward packet length. A negatively skewed distribution of packet lengths mean more packets have larger packet length. Increasing minimum forward packet length also increases packet length.

The modified attack has a few more bytes in the packet length, and the result was able to bypass the detection engine, however it was purely due to the adversarial attacks having different duration as the ones we are training, thus it will have different number of flag counts, etc. To make things constant, another sample of slowloris is generated, and the duration of the attack is fixed to around 135 seconds.

Interestingly the 135 second slowloris has made the classifier less accurate compared to 123 second duration. it now has an accuracy of ~0.9831 on the training set. The JSMA algorithm also gave a different set of features to change: forward packet size skewness by -3 and forward iat skewness by 19. skewness and kurtosis does not represent a distribution well when there is small number of points, thus skewness and kurtosis is also added to fixed.

So the new one shows forward packet size min by 37  and forward iat mean by -7. When inspecting the packets in wireshark, the minimum packet size in the forward direction was a TCP three way handshake with no payload, thus we cannot modify that field. Thus another constraint fixing min was placed, and the perturbations are increase mean forward packet size by ~190 and decrease fwd iat mean by 8 seconds.

Now here is a slight problem: although decreasing fwd iat is possible, it will result in more packets being sent, and thus more flag counts. The take away message from this is that due to the nature of DoS attacks, or maybe even network attacks in general, there are many indirectly interconnected features, and for a JSMA attack it is rather difficult to fix a particular feature while have free reign of other features. A more advanced way of generating adversarial samples is to capture the interconnectivity between features and use that during generation.

Even if the iat is fixed, the result of perturbation shows to increase mean fwd packet length by 230 and nothing else. This seems easy to do, but it does not take into account that increasing fwd packet length by 230 will also increase the max fwd packet size.

## Findings

Due to the nature of DoS (may be able to generalize to all network attacks), the flow features are interconnected and thus it is highly unlikely that we can change a single feature and make other features remain the same. Under such circumstances, JSMA is not very useful, and a better way of generating adversarial samples is needed.

So in order to make adversarial perturbations more realistic with heavily interconnected features, I have thought of two possible ways:
- Use variational autoencoders to create a good compression of the input data, hopefully with the right hyperparameters, it will naturally form clusters based on the labels. With such clusters, we can generate adversarial samples in two possible ways:
  - we can pick a point that is on the boundary of the between malicous and benign clusters to generate a sample and then classify the sample. This loses the ability to place constraints on the sample, and might end up with attacks that exhibits different signatures.
  - alternatively we can start with a malicious attack and move the sample towards the benign cluster. This way, we can control the direction, and have a soft limit on which features can change by what value.
- Use a GAN, but instead of generating from a random seed, we can generated adversarial samples based on a benign input, and somehow combine the feedback from the discriminator to make it more realistic. The discriminator would be discriminating between real and fake samples.

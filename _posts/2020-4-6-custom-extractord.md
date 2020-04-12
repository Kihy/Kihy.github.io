---
layout: post
title:  "Custom Feature Extractor"
author: "Ke He"
description: "Thoughts and processes of writing our own custom network feature extractor"
---

## Flow Based vs Packet Based

The CIC-IDS features are flow based, the advantages of this is that it reduces the number of samples, and are able to compute statistics of network flows. The classifier, therefore will have to classify the samples as flows. This would work for most network traffic, however it occurred to me while working on the slowloris attack, the attack purposely keeps a connection as long as possible, meaning the flow would go on for a long period of time. This means the classifier would not be able to classify flow immediately as it has not finished, and in a way it naturally bypasses flow based IDS. The CICFLowMeter's real time capturing interface was not able to pick up any flow for slowloris attack.

Packet based features IDS solves the above problem, at the cost of less statistical information about entire flow. The nature of packet based IDS means time-sequences of packets are important (unlike flow based, where flows can be thought of as independent). This makes classifier harder to implement. Generally a RNN have two main components, embedding of input and recurrent units. For our problem, ideally the classifier would read each packet, and give a classification of whether it is malicious or benign after each packet. This is similar to named entity reconginition in NLP, which is identifying the type of words in a sentence(e.g. nouns and verbs). The state of the art approach is to use bidirectional LSTM-CRF model (obviously in real-time we do not have any information about packets afterwards, unless you sacrifice some time and have post packet buffer). The embedding layer for NLP is usually word embeddings. Since we are working with packet data, a more suitable apporach would be to use extracted packet fields. If number of fields becomes too large, a siamese/triplet encoding model can be trained to embed packet data.

Another detail to note is that parallel connections can mess up packet index. Thus when using the classifier for classification, it should either have a classifier instance for each flow (and clean up after flow is finished) or manually save/load/delete current memory to match stream index.

## CIC IDS dataset

As mentioned multiple times in the blog posts before, the flow features exhibits weird behavior that can be caused either by the flow feature extractor or the network traffic data. A closer look at the raw dataset pcap files shows the following:
1. The flows provided with the dataset is wrong. It contains way too many single packet flows. This can be verified by using the CICFlowmeter on the raw pcap files.
2. The extracted flow from the flow meter sometimes have 2 flows instead of 1 (maybe intended because of Bidirectional flows?). This can be verified by examining the conversations in wireshark(which is essentially a flow).

So our own flow extractor is probably needed.

## Features of our own Extractor

features of our feature extractor would be based mainly off CICFlowMeter's features, so it is worth looking at the features they extract. Note CICFlowMeter uses min,max,mean,std as a way to describe the distribution.

| flow meter features                           | comments                                                                         |
| --------------------------------------------- | -------------------------------------------------------------------------------- |
| flow duration                                 | useful to detect unusually long flow durations                                   |
| total fwd,bwd packets and its size            | gives a overview of packet data                                                  |
| distribution of packet sizes in fwd and bwd   | captures the distribution of packet sizes                                        |
| flow byte rate, flow packet rate              | unneccesary as it can be derived                                                 |
| distribution of flow,fwd,bwd iat              | inter arrival time can be useful for slowloris                                   |
| number of flags for both direction            | can be useful for portscan                                                       |
| header length                                 | unneccesary, since the only different is the options fields which is rarely used |
| number of fwd,bwd packets per second          | unneccesary since it can be derived                                              |
| packet length distribution in both directions | provides general information                                                     |
| flag counts                                   | useful for portscan                                                              |
| down up ratio                                 | derived                                                                          |
| bulk rate                                     | can be captured with packet length                                               |
| subflow                                       | no idea what that is                                                             |
| initial windows packet size                   | meh                                                                              |
| active and idle time distribution             | isn't that basically iat?                                                        |

The CICFlowMeter's measure of a distribution assumes that the underlying distribution is somewhat nice and normal, which is often not the case in network traffic as it can be multimodal with many outliers bumping min and max values. Thus in order to efficiently capture the distribution, a novel online algorithm, tdigest, is able to efficiently calculate an approximation of percentiles of data. Using the percentiles, we are able to capture distributions of any shape. For our extractor we'll choose to capture the 1,25,50,75,99 percentiles. The 1% at both ends eliminates outliers for min-max scaling. The features we are going to extract are:

| feature name               | description                                       | number of features |
| -------------------------- | ------------------------------------------------- | ------------------ |
| duration                   | total duration of flow                            | 1                  |
| protocol                   | protocol used for the flow                        | 1                  |
| {dst,src}\_port            | destination and source port number                | 2                  |
| {fwd,bwd}\_tot\_{pkt,byte} | total number of forward/backward packet and bytes | 2 \* 2             |
| {fwd,bwd}_pkt_size_{n}     | distribution of fwd/bwd packet size               | 2 \* 5             |
| {fwd,bwd}_iat_{n}          | distribution of fwd/bwd inter arrival time        | 2 \* 5             |
| {fwd,bwd}\_{flags}\_cnt    | number of packets with various flags              | 2 \* 8             |

Where n={1,25,50,75,99} and flags={FIN,SYN,RST,PUSH,ACK,URG,CWE,ECE}, and forward packet is defined by source to destination, backward is destination to source.
A total of 1+1+2+4+10+10+16=44 features

When implementing tdigest, it was realised each flow does not have enough packets for it to be worth using tdigest, and there are some problems with getting the right percentile. Thus it was switched to use IncrementalStatistics to calculate mean, std, skewness and kurtosis with min and max value.

The upated features are:

| feature name               | description                                       | number of features |
| -------------------------- | ------------------------------------------------- | ------------------ |
| duration                   | total duration of flow                            | 1                  |
| protocol                   | protocol used for the flow                        | 1                  |
| {dst,src}\_port            | destination and source port number                | 2                  |
| {fwd,bwd}\_tot\_{pkt,byte} | total number of forward/backward packet and bytes | 2 \* 2             |
| {fwd,bwd}_pkt_size_{stat}  | distribution of fwd/bwd packet size               | 2 \* 6             |
| {fwd,bwd}_iat_{stat}       | distribution of fwd/bwd inter arrival time        | 2 \* 6             |
| {fwd,bwd}\_{flags}\_cnt    | number of packets with various flags              | 2 \* 8             |
| total                      | total number of features                          | 46                 |

Where stat={mean,std,skew,kurtosis,min,max} and everything else is the same.

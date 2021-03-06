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
1\. The flows provided with the dataset is wrong. It contains way too many single packet flows. This can be verified by using the CICFlowmeter on the raw pcap files.
2\. The extracted flow from the flow meter sometimes have 2 flows instead of 1 (maybe intended because of Bidirectional flows?). This can be verified by examining the conversations in wireshark(which is essentially a flow).

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

Where stat={mean,std,skew,kurtosis,min,max} and everything else is the same.

## Extractor logic

The feature extractor uses observer pattern to be more robust. the observable is a StreamingInterface, where packet data is generated. Currently only offline interface that reads from a pcap file is developed. In the future it is possible to create a real-time interface reading directly from NIC.

Once a packet is read/received, the StreamingInterface notifies all observers. The observers in this case is the flowmeter. Currently only TCP flowmeter is created, and it has a dictionary of flows that contain statistics of the flow. To reduce memory, if a flow did not have a packet 600 seconds since the last packet, it will be considered as finished, and thus saved to file and removed from our flow dictionary.

## implementation Details

-   The offline version of the flow meter relies on tshark's internal stream indexes to determine flows. For real time interface, probably need to create 5 tuples as index.
-   The flow meter currently only checks TCP and UDP packets.
-   The packet size attribute is the size of the entire packet, rather than size of tcp payload. This is done so that we can compare with wireshark conversations.
-   The ordering of flows generated is by its finishing time, or if multiple flows have finished it is by start time.
-   In order to speed up the process, only_summaries is set to True in FileCapture(), this significantly increases the packet generation time, however, the default fields for only_summaries does not include stream index and other fields, thus we have to do the following:
    -   the summaries are specified by psml file, which is linked to wireshark's gui column interface. Whatever attribute is displayed in wireshark's column interface is displayed by only_summaries.
    -   The easiest way is to open wireshark's gui interface and get all the important fields on the columns.
    -   Then go to help -> about wireshark -> folders -> preferences and search gui.column.format and copy the string.
    -   In FileCapture, specify custom_parameters to {'-o', 'gui.column.format:{string}'}
    -   Note that some fields common but have to be specified seperately for tcp and udp.
-   When processing DoS datasets the timeout should set to a low number, otherwise the number of flows stored will be large and the speed of processing reduces significantly. In real time interfaces it is unneccesary.
-   For nmap scan the flags field may be reserved set to 100, in this case we ignore it.

## Speed Boost
When processing large files, the naive approach is extremely slow, since it has to process all flows for each packet arrival. To speed things up, there is a field that reduces the frequency of checking (-i).

To further boost the speed, a Ordered dict is used instead of original dict. Once a new flow is generated, it is added at the end, once a flow is updated it is moved to the end (this operation is O(1), similar to linked list). This means we only have to check the first few flows at the head of the dictionary, as they will have not being updated in the longest time. The number of flows to check is specified with -l.  

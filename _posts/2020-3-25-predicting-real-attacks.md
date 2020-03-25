---
layout: post
title:  "Prediction on real attacks"
author: "Ke He"
description: "Predicting data gathered from real attacks carried out in virtualbox"
---

### Feature Conversion
The extracted feature from CICFlowMeter has different column names and number of columns compared to the trained file. Thus we have to identify a mapping between the columns from CICFlowMeter and the training set provided. While doing so, it was dicovered that our training set has two Fwd Header Length field containing exactly the same information. Thus one of the column is removed.

### Prediction
Two most common attacks were conducted: PortScan and Dos Hulk in virtualbox. The traffic was captured with wireshark and features extracted with CICFlowMeter. The predictions show port scan being successfully detected, but hulk attack is classified as portscan as well. This could be caused by hulk attack not being carried out correctly, as running the hulk script did not make the ubuntu webserver unresponsive. Need to look into HULK attack more.

---
layout: post
title:  "Kitsune"
author: "Ke He"
description: "Creating adversarial examples for Kitsune detection system"
---

## Kitsune

Kitsune is an anomoly detection based IDS using an ensemble of autoencoders. It first extracts features from traffic, then it clusters the features into various groups based on the correlation between each feature. Then each group of features is fed into an autoencoder and respective RMSE value calculated. The RMSE values for each group is then fed into another autoencoder and the final RMSE value is generated. This value is the anomaly value of the network, if it gets above a certain threshold it signals the presence of an attack.

## Kitsune Feature Extractor

The kitsune paper states that for each stream there will be 115 features extracted (23 \* 5 different damping factors). However the 115 features are mostly connection specific, meaning in the database there will be 1 set of 115 feature per socket. Different sets of features are stored in a hash table, which for IoT networks, the network Kitsune is intended for, there will likely not be too much sockets and ip addresses. However for enterprise networks the memory required for storing hash table grows exponentially (each extra host can connection to all other ones, resulting in O(n!) space complexity) and therefore is not suitable.

## Adversarial examples for autoencoders

Kitsune uses autoencoders in an interesting way. Normally autoencoders are used for dimensionality reduction of input or generation of similar examples. Here the authors use autoencoders's RMSE value as a basis for a threshold. This would work if one can prove that autoencoders trained to see samples in a particular datarange can only reform samples in the same range and nothing else. To test this theory, a simple autoencoder is set up with 4 inputs and 3 hidden states. The 4 inputs during training have the following ranges:

```python
a=np.random.uniform(0,0.25,size=sample_size)
b=np.random.uniform(0.25,0.5,size=sample_size)
c=np.random.uniform(0.5,0.75,size=sample_size)
d=np.random.uniform(0.75,1,size=sample_size)
```

The autoencoder is trained with 1 million samples and 10 epochs. the average MSE is 0.0052. Note that MSE is used in practice since sqrt is a monotonic function, so ignoring it simplifies computation.

A test case of 10000 samples is fed through the network and MSE values recorded:
max 0.014227789
min 5.6585784e-05
avg 0.0051914714

So a threshold for anomaly would be set somewhere around 0.014

Then a random sample with attribute a not in its usual range is generated:
[0.4845569,0.39394628,0.64963341,0.83376447]
and it does indeed give a MSE of 0.031246021, which is above the threshold. A method similar to FGSM can be used to create adversarial samples. Using tf.gradient tape we can get the gradient of input wrt to the output:  0.17988138  0.00892995  0.0124712  -0.02067012. With this information we know increasing 1,2,3 attribute will increase the MSE and increase 4 will decrease MSE. Now there are a couple of ways to generate the adversarial sample:

1.  the FGSM way would be getting the sign of the gradient and perturb input accordingly (-positive sign and + negative sign)
2.  Or you could find the maximum abs(gradient) and manipulate the input according to the sign. Probably generates least amount of perturbation
3.  A more complex way is to see if there are positive and negative gradient, and find a balance between adding and subtracting features so the adversarial sample is vastly different.

Using method 2 an adversarial samples is found:
0.3545569  0.39394628 0.64963341 0.83376447
with MSE 0.013860257

A possible downside of using method 2 is that it may lead to adversarial sample back into normal range. If we start with 0,0,0,0 the generation yields 0.01 0.26 0.5  0.76, which is approximately the minimum value of our normal range, similarly if we start with 1,1,1,1 we end up with the maximum of our feature range: 0.24 0.5  0.74 0.99. On the other hand this method could be useful for finding the ranges which the autoencoder is expecting.

Reducing the threshold value would cause our sample to reach 0.13 0.38 0.62 0.82, which is the average of our uniform range. Using random points as starting point, and using various thresholds, maybe we can visualize the expected distribution of the autoencoder's input.

The feature ranges do no overlap for our example, so another example where feature ranges do overlap shows similar results.

## Applying to kitsune

Kitsune consists of a ensemble of autoencoders and a single autoencoder with RMSE of ensemble autoencoders. Our algorithm can probably be ran on the entire network, but it would be difficult to generate valid adversarial examples since many attributes have to be changed. So instead it would be wiser to consider one autoencoder that takes a subset of feature input. depending on the size of the subset we can easily generate adversarial samples.

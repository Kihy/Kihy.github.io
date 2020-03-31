---
layout: post
title:  "Adversarial slowloris"
author: "Ke He"
description: "Creating Adversarial slowloris samples with out network"
---

### Adversarial Generation
To generate adversarial samples we have chosen to use JSMA method. JSMA relies on saliency maps to perturb the feature that contributes most towards a particular class. The original implementation of saliency maps only increase or decrease features. Our implementation is a modification of the original implementation that compares the saliency map of increase and decrease feature and picks the maximum value out of the two. It does come with some consequences: there will be a particular sample with particular theta value that will oscillate between increasing and decreasing a set of features, causing the sample to never reach the end.

### Constraints
It is important to have constraints in our adversarial generation. In our framework we consider 2 different types of constraints: prior and posterior constraints.

#### Prior constraints
Prior constraints are constraints that are placed during adversarial generation (e.g. forcing particular column to not change). These are constraints that describe the key characteristics of a class that makes the adversarial attack, and perturbing such feature makes the adversarial sample impossible to be produced (e.g. changing protol for most DoS attacks), which is not useful at all. In image classification problems, these constraints can be thought of as clipping pixel values to be between 0~255(or 0~1 depending on scale).

#### Posterior constraints
Posterior constraints are placed after an adversarial sample has been generated. It describes the ability of the attacker to modify the sample. As an example, if the JSMA algorithm shows altering payload length can bypass the classifier, but the attacker is a script-kiddy and does not know how to do so, the adversarial sample is unlikely to be useful. Another examples is different frameworks have different levels of customization of the same attack, so depending on the level of customization the posterior contraints will differ.

Prior constraints are fixed for each type of attack, and can be reused, while posterior constraints depend on each individual/framework. The overall constraint is made as an intersection of the two constraints.
![Constraints](assets/images/charts/constraints.png)

---
layout: post
title:  "Preprocessing CIC IDS"
author: "Ke He"
description: "Problems and solutions I encountered while processing CIC-IDS2017 dataset"
---

# CIC-IDS2017 dataset
The NSl KDD dataset is rather small compared to CIC-IDS2017 dataset, which means storing all data in memory which would work for NSK KDD would no longer work for CIC-IDS data. In fact it freezes my computer and crashes my terminal, so we have to make a generator like object to read data as we go.

Lucily there tensorflow offers tf.dataset which solves the problem perfectly. However it uses a different csv format. tf.dataset requires csv files to have a header, which NSL KDD data dont, and it automatically splits by feature and label, which NSL KDD also dont. Thus the input_utils.py has been refactored to account for these changes, which made the code more concise.

Another change that have to be made was minmaxscaler does not work anymore since we have tf.data of batches rather than entire dataset. Thus a metadata file was created to calculate min and max of all samples for scaling. For completeness mean and std was calculated as well in case we want to normalize or use tanh estimator for preprocessing. However while calculating mean and std it turns out Flow Bytes/s and Flow Packets/s contains infinity, which means mean and std calculations for these 2 rows are infinity and nan. At this stage we drop the rows that have Nan or Infinity values.

#### comparison before and after removal
  before                 after                  difference
  0     2273097          0     2271320              1777
  4      231073          4      230124              949
  10     158930          10     158804              126
  2      128027          2      128025              2
  3       10293          3       10293              0
  7        7938          7        7935              3
  11       5897          11       5897              0
  6        5796          6        5796              0
  5        5499          5        5499              0
  1        1966          1        1956              10
  12       1507          12       1507              0
  14        652          14        652              0
  9          36          9          36              0
  13         21          13         21              0
  8          11          8          11              0
2830743                      2827876              2867

## tf.dataset
The tensorflow dataset pipeline offers a completely different structure compared to what I was using. Thus the whole input pipeline has to be re-written. The tensorflow dataset pipeline first consumes a csv file using make_csv_dataset. Unlike what we do for NSL KDD, the epochs and batch sizes for training can be specified in the make_csv_dataset rather than model.fit (note if you specify epochs in make_csv_dataset to 10 and in model.fit to 10, then you would be training on 100 epochs of your data). After the csv file has been consumed, it becomes a dictionary with key being column names and values being 1 by batch_size of the column. To scale the data, we have to first pack each column in the batch to a 78 * batch_size array and then apply our custom scaling to it. The scaling is done on its own layer called DenseFeatures. The model has to be created differently as well to accommodate our changes. The input to the dense_features layer is not an array anymore but a dictionary with the key being the packed data keys and values being an Input layer with dimension of the packed data.

During training the program reported error saying we have large negative values in our dataset. Upon examining the fields in the dataset, it is clear that all fields should be positive. Thus we have removed all rows containing negative values, which significantly reduced the number of records:

0     950588
4     163854
10    158740
2      81478
3       7709
7       6439
11      5883
6       4154
5       2327
1       1956
12      1507
14       635
9         32
13        21
8          7

However in training the loss shows as nan from the very beginning of training for some reason.

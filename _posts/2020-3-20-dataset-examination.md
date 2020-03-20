---
layout: post
title:  "Dataset Examination"
author: "Ke He"
description: "Analysis of the CIC-IDS dataset"
---

### CIC-IDS attribute range
While training the network the loss is nan from the very beginning of training. The following steps were conducted to debug:
1. print the y_true and y_pred values in the loss function. This can be achieved by writing a custom loss function that directly calls categorical_crossentropy but has tf.Print(y_true) and tf.Print(y_pred) values in the function. Using this technique it turns out the y_true values are not one hot encoded, and the y_pred values are all nan. The y_true values are fixed by changing the pack function to return one hot encoded labels.
2. Since the y_pred values are nan, it could only mean there is something wrong in the matrix multiplication steps of the neural network. The network was shrinked to a single dense layer in case it was due to multiple layers introducing vanishing/exploding gradients. Then the weights of the initial layer was extracted and multiplied by the output of numeric_layer manually with np.matmul. The output is all nan.
3. Upon searching on internet, np.matmul causes all nan can be caused by one of the matrixes having nan values. It is unlikely that the weight matrix has nan values, using np.argwhere(np.isnan()) we can find the indexs of any nan values. It turns out we do have nan values after numeric_layer.
4. The numeric layer's job was to do min max scaling. In order for that to generate nans it is highly likely that the dataranges for some attributes are 0 (i.e. max=0 and min=0). Indeed this is the cause of our problem. To resolve it np.where is used to replace 0 in data_range with 1 so that the scaled values are still 0.
5. After these changes the loss is normal and we can finally start training.


### batch_size conflicts
Due to the batch size is fixed, it is highly likely out data does not fit perfectly with batch_size. This will report:
BaseCollectiveExecutor::StartAbort Out of range: End of sequence
     [[{{node IteratorGetNext}}]]
To solve this more meta information such as number of train, val test samples are added to metadata file, and adding steps_per_epoch and validation_steps to model.fit.

### cannot save model with functools.partial
If functools.partial is used for passing parameter(i.e. min and max) to the normalizer function, the model cannot be saved due to functools.partial cannot be serialized. This can be solved by using function closures in python:
```python
def a(min,max):
    def b(data):
      ...
      return scaled
    return b
```


### dataset visualization and evaluation
It would be useful to create dataset stats and visualizations, therefore dataframe.hist() and counts of each class is generated. Once that is done we use the evaluation module for NSL KDD to generate heatmaps and classification reports.

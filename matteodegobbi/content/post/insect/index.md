+++
date = '2025-06-18T00:51:54+02:00'
draft = false
title = 'Paper on Insect Species and Genus Classification 🐝🐞🐛'
categories = ["Deep Learning"]
tags = ["python","matlab","pytorch"]
image = "img.png"
+++

In February of this year we published a [paper](https://www.mdpi.com/1999-4893/18/2/105) on undescribed Insect species and genus classification using DNA and image data.

My main contributions, together with Roger, were:
* ReACGAN for image feature extraction
* CNN model with vertical kernels for DNA feature extraction
* Compiling of finetuning dataset for the ReACGAN and CNN, scraping data from BOLDSystemsV3. (Image+DNA)
* Compiling of pretraining dataset for the ReACGAN, merging previous datasets. (Only Image)
* Replicating a previous study on the new dataset, since the dataset of the original paper is not publicly available

The finetuning dataset can be found at: <https://zenodo.org/records/14277812>\
The pretraining dataset can be found at: <https://zenodo.org/records/14577906>\
The code can be found at: <https://github.com/matteodegobbi/InsectClassification>\
Paper: <https://www.mdpi.com/1999-4893/18/2/105>\





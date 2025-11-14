+++
date = '2023-10-16T02:06:54+02:00'
draft = false 
title = 'Dataset Inference attacks for Generative Adversarial Networks 🤖'
image="gan.png"
categories=['Deep Learning']
tags = ["python","tensorflow"]
+++

In my bachelor's thesis I explored the idea of membership inference attacks, where we try to determine whether a given sample was present in the training set of a neural network without having access to the weights of the neural network itself. In particular I targeted Generative Adversarial Networks where the training data can contain sensitive information, especially in the medical setting, in forensics and criminal justice.

The full thesis can be found at: https://thesis.unipd.it/handle/20.500.12608/57083

## Abstract:
Generative Adversarial Networks (GANs) have had great success in the generation of artifical samples from datasets made of sensitive data which can't be disclosed publicly. These GANs, if released to the public, could allow an attacker to leak sensitive information from the GAN's training dataset. We analize a type of attack called Membership Inference Attack (MIA), which consists of determining the membership of a certain sample to the training set of the GAN. We analize the success of both black box and white box Membership Inference Attacks on GANs trained on MNIST and anime faces. We look for a relationship between the precision of the attacks and the several hyperparameters of the GANs such as: amount of images in the training set, number of epochs of training, quality of generated images, number of generated images available to the attacker. We show how an insufficient number of training images or an excessive number of training epochs causes overfitting in the GAN, which is then vulnerable to MIAs. We analize how the Fréchet's Inception Distance (FID) between the set of generated images and the original training set impacts on the success of the MIAs. 


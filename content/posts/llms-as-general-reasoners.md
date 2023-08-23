---
title: "LLMs as general reasoners"
date: 2023-08-21T12:00:00+02:00
# weight: 1
# aliases: ["/first"]
tags: ["llms"]
author: "Rick Lamers"
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: "Measuring in-context learning performance to unlock further progress"
canonicalURL: "https://ricklamers.io/posts/llms-as-general-reasoners"
disableHLJS: true # to disable highlightjs
disableShare: false
disableHLJS: false
hideSummary: false
searchHidden: false
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
ShowRssButtonInSectionTermList: true
UseHugoToc: true
---

In ML it's common to measure generalization performance as the delta between the training, validation and test dataset. Training data is what your model actively optimizes against using some optimization procedure (e.g. SGD or conjugate gradient or quadratic programming). The validation data is used to get an unbiased estimate of the performance of the model on unseen data. Since the process of selection (hyper parameter selection, stopping criteria) invariably overfits the model to the validation data at the very end we get an estimate of model performance through the test dataset that should not be used for the process of training a model.

In the context of Large Language Models today evaluation is particularly important. For model developers (whether foundational or through fine-tuning) to decide which choices (architecture, dataset, training procedure) lead to better model performance. And for application builders which model to use for their use case depending on the trade-offs involved: task performance, inference costs, model freedom (open weights vs proprietary API based models), inference hardware requirements, inference performance (token/s), context window size, etc.

To make decisions you need to focus on the right metric. But what is the right metric? The answer is of course "it depends" but based on my observations of LLMs so far I hypothesize that what really matters for improving task performance moving forward is the ability of the model to perform in-context reasoning.

Here's where I loop back to the introduction where I covered the common ML procedure of measuring performance. The idea of the setup is to measure how well the model has learned to generalize beyond the data it was exposed to during training. The thing with LLMs today is well that they are, large, and that good performance for many use cases requires the model to perform well on unseen data. Performing on unseen data becomes more important because continuously expanding the data the model has seen is impractical (e.g. model management, updating all the inference endpoints to use the new set of weights) and expensive. Unseen data includes news articles or other factoids that were not yet available during the time of training, but also complex new reasoning problems like the unique combination of components that might come together in unique ways never seen before when engineering new (software) systems. Imagine the common scenarios in which GitHub Copilot is deployed: coding software and navigating the challenges the come up in the process (dependency issues, architectural decisions, complex bug interactions, etc.).

The way LLMs are optimized today is on a simple objective of next-word token prediction. It enables the self-supervised paradigm that can tap into exploiting statistical structures across the vast body of pre-existing corpus of any type of writted language that exists (code and prose alike). That objective function has produced models that are capable of astounding feats. However, what we care about and what we're able to optimize against through a procedure are often not the same as usefully instilled by my once academic advisor [Marco Loog](https://www.tudelft.nl/ewi/over-de-faculteit/afdelingen/intelligent-systems/pattern-recognition-bioinformatics/pattern-recognition-bioinformatics/people/marco-loog).

I'd argue that for LLMs to continue to improve we need to introduce measurements that focus on the generalization capabilities of models, do they show the ability to learn _quickly_. Yann LeCun [remarked](https://twitter.com/ylecun/status/1688168161445568512) that one of the big shortcomings of self-supervised learning is the astounding gap between speed of learning between LLMs and humans. Perhaps pre-training as a paradigm can be used to create quick learners. Quickly as measured in few-shot performance on single pass-forward inferences. This will further establish the paradigm that underlies the proliferation of LLM applications which is crucially dependent on the clear division of responsibility of the API line: software that uses LLMs need not be concerned with any of the details involved in creating this general purpose reasoning function.

This is a deep field of scientific inquiry and my intuitive description doesn't do justice to the existing research involved and theoretical concepts that already capture the core of these ideas. But I hope the intuitive description can create alignment in the LLM community about what we really should be measuring. I believe we shouldn't want big memorizing machines but general learners that will be capable to go much beyond the data its reasoning engine was bootstrapped with.

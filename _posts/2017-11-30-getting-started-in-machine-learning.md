---
layout: post
title: Getting started in machine learning
---

It's only relatively recently that I learned that machine learning was becoming such an exciting and important field and that I was late to the game! The first thing that made me tick was when I learned that Google had open sourced [TensorFlow](https://www.tensorflow.org/). It wasn't the first open source ML library, but somehow that's when I realized that the field was ripe for growth and not just for a few niches to sell ads or improve movie recommendations.

I made this post to be able to refer it to friends who, like me, want to enter the field and who don't know where to start. It's only one data point, so I would still recommend doing additional research to find what works best for one's situation and natural inclinations. Some people learn better when they have a concrete project right from the start for example, so they'll learn just what they need to solve that problem. There is a speed VS depth tradeoff, but ultimately, whatever keeps one motivated and excited the best is often a good starting point. 

# Where to Start

## Learning the basics

I wasn't sure where to start, but eventually I learned about [Stanford University Machine Learning course](https://www.coursera.org/learn/machine-learning)  taught by Andrew Ng on Coursera.

I liked it so much that in fact, I recently built on it with [Andrew Ng's Deep Learning](https://www.deeplearning.ai/) specialization. (3 courses completed out of 5 so far).

Andrew has a knack to make the material simple to understand. He's really a gifted teacher and for that alone, I think those classes are a great time investment. It's a bonus that they are either free or very cheap. Some basic knowledge in programming is required. I had to dust off maths knowledge from 15 years ago, but Andrew did an incredible job of quickly ramping me up to speed. That may seem odd, but I think that the fact that the original course uses Matlab may have helped me here. I'll likely never use Matlab again, but it's a great learning environment for maths.

I found the classes to still have a few caveats though:
* Most people don't finish online classes because no one is forcing you to and sometime, watching Netflix is easier after a long day at work. Make sure you have a clear goal that will motivate you through.
* Once you have completed the original Stanford Machine Learning course, the coding exercises in the Deep Learning specialization feel superficial and tedious. You can summarize most of them as "write down equation XYZ in Python between those two lines". I suspect they were short on time when they built the curriculum and eventually, they will circle back and revamp them completely.
* The exams are simple multiple choices questions that you can retake over and over so they don't reliably prove anything. As an employer, I would see these Coursera courses on a resume as a positive sign that a candidate is motivated, but they mean nothing about expertise gained. I wouldn't even bother looking at Coursera grades.
* A more insidious side effect of the easy exams and shallow coding exercises is that it's easy to think that you've learned the material, while in fact there are still holes in your understanding. In those matters, it's still superior to a book though!
* Even after all those courses, once I started to attack real life problems, I realized that my practical experience wasn't very deep. I would really recommend to mix the learning material with some real projects.

## Alternatives

I ordered and started to read the [Deep Learning](http://www.deeplearningbook.org/) book by Ian Goodfellow, Yoshua Bengio and Aaron Courville. It's very complete, but concepts that were almost childish in Andrew's classes were much more opaque in that book, simply because of the much more academic approach and the way the information is structured. It's a better reference once you have spend some time with Andrew Ng.

I also tried my hand at [Python Machine Learning](https://www.packtpub.com/big-data-and-business-intelligence/python-machine-learning) by Sebastian Raschka. It's an interesting book that has the advantage of containing more code samples. I also noticed online that the author is very active in online forums to help others and explain concepts so I find it even more worthy to encourage him by buying his book. At first I didn't enjoy it as much but I'll give it a second try sooner rather than later as it seems full of useful information.

## Practical experience

Luckily, I found it incredible how many high quality resources available to gain practical experience.

There is the [scikit-learn](http://scikit-learn.org) machine learning library that is chock-full of tutorials and examples. Once you have a good grasp at the basics, I found this library very enjoyable to explore traditional machine learning (it doesn't cover deep learning).

I also have to mention [Kaggle](https://www.kaggle.com/). The Iris and Digits datasets are neat to start dabbling in ML, but eventually, you'll want a more exciting project and creating high quality datasets yourselves is a hard endeavor in itself. Kaggle has hundreds of them. It even has competitions with real money prizes. The best part for learning, is that some of these datasets such as ImageNet, are well known and a lot of people share their solutions (called kernels) publicly so you can learn from them.

If you use Python in your data science experiments, I can't recommend [Anaconda](https://www.anaconda.com/download/) enough. Dealing with Python packages dependencies is very unfun and Anaconda installs data science packages out of the box for you and help deal with the dependencies.

I also found [PyCharm](https://www.jetbrains.com/pycharm/) to be an excellent IDE with a good debugger and integrated Jupyter support.

## Cutting Edge

The field of machine learning, and deep learning in particular, are very open about sharing information. One proof of this is the extensive use of [arXiv.org](https://arxiv.org/) by machine learning researchers to publish their results. After the courses I took, I found well written research papers to be relatively digestible even though I didn't read many academic papers in depth before.

For example, a paper about [pneumonia detection with deep learning](https://arxiv.org/pdf/1711.05225v1.pdf) recently made the news. Like many papers, it's freely shared on arXiv.org and you can learn from it and even try to replicate the results.

Hopefully, that's a useful starting point.

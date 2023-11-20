---
title: "A PhD approach to learning new things"
date: 2023-11-XX
---
## Intro
Two great things that PhDs have are their understanding of the scientific method and fast learning skills. Combining these two attributes allows us to pick up new technologies extremely fast. If you add the incredible mathematical background of a theoretical physicist to the mix, Machine Learning Technologies can be learned in a couple of hours.

In this post, I want to recap how I learned to use PyTorch and Neural Networks in a PhD kind of way. This learning consists of 3 steps:
* Get an understanding of the problem/technology at hand: In this case, I watched a YouTube video on Neural Network (from 3Brown1Blue - highly recommend the channel)
* Get straight into simple applications: Apply PyTorch to the simplest problems you can think of and slowly ramp up the difficulty
* Apply the problem at full scope

## Get an understanding of the problem/technology

Before getting our hands dirty, we should learn the theory. Luckily, the mathematics of Neural Networks are fairly simple if you used them almost daily over the last 10 years. Therefore, it was enough to get a conceptual understanding of Neural Networks without the need to spend much time on the mathematics behind it. For conceptual understanding, there is an exceptional YouTube channel called 3Brown1Blue that explains many interesting mathematical concepts in simple terms with beautiful visualisation (Seriously, I wonder how much time is spent on the animation vs writing the script)

Other topics that use less familiar methods will require more time in this step. But as a PhD in theoretical physics Machine Learning concepts usually require very little learning of new mathematical concepts. It is rather the application or scope that they are used in. If new concepts have to be learned, using textbooks, research papers or more math-heavy YouTube videos will be the go-to

## Get straight into simple applications

This is the point where the scientific method comes into play or I should rather say a scientific approach. What I call the scientific approach is the process of establishing fact, or in our case, gaining an understanding through testing and experimentation. When learning new programming skills, we want these experiments to be as controlled as possible. That means we want to build up our experiments as simple as possible so that if something goes wrong, we will be able to pinpoint the problem immediately. 

For our neural network that means the first application will be fitting a line. We will then slowly ramp up the complexity with the end goal of this section being an image detection of hand-written digits. To some people's surprise, we will spend most time building the first application 'fitting a line'.

! Image of a line fit

In this step, we learn about the syntax of PyTorch and we apply what we learned about neural networks. If you want to learn a bit more about these steps, neural networks and PyTorch, you can have a look at another post of mine. For now, we are less interested in the details of neural networks, but in the approach to learning.

Once, we are able to fit the line, we can increase the difficulty. We add some noise to the data to test if the regression also works on more complicated problems. Once that is established, we move on to a slightly more complicated regression - Logistic Regression. For the neural network that means we need to add an activation function. Now, we have all the skills needed to build an application that can classify digits. We have now built the trust in us to build neural networks, so let's build a slightly more complicated architecture for the neural network. 

To classify the hand-written digits, we simply add one more layer to the architecture of our neural network and that is already enough to get good results.

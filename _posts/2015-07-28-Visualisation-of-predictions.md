---
layout: post
title: Visualization of predictions
author: jakob
---

In this post I want to shortly introduce you to the great visualization possibilities of `mlr`.
Within the last months a lot of work has been put into that field.
This post is not a [tutorial](https://mlr-org.github.io/mlr/) but more a demonstration of how little code you have to write with `mlr` to get some nice plots showing the prediction behaviors for different learners.

<!--more-->

First we define a list containing all the [learners](https://mlr-org.github.io/mlr/articles/tutorial/devel/integrated_learners.html) we want to visualize.
Notice that most of the `mlr` methods are able to work with just the string (i.e. `"classif.svm"`) to know what learner you mean.
Nevertheless you can define the learner more precisely with `makeLearner()` and set some parameters such as the `kernel` in this example.

First we define the list of learners we want to visualize.

{% highlight r %}
library(mlr)
learners = list(
  makeLearner("classif.svm", kernel = "linear"),
  makeLearner("classif.svm", kernel = "polynomial"),
  makeLearner("classif.svm", kernel = "radial"),
  "classif.qda",
  "classif.randomForest",
  "classif.knn"
  )
{% endhighlight %}

## Support Vector Machines
Now lets have a look at the different results and lets start with the SVM with a *linear kernel*.


{% highlight r %}
plotLearnerPrediction(learner = learners[[1]], task = iris.task)
{% endhighlight %}

![plot of chunk linear-svm](/figures/2015-07-28-Visualisation-of-predictions/linear-svm-1.svg)

We can see clearly that in fact the decision boundary is indeed linear.
Furthermore the misclassified items are highlighted and a 10-fold cross validation to obtain the mean missclassification error is executed.

For the *polynomial* and the *radial kernel* the decision boundaries already look a bit more sophisticated:

{% highlight r %}
plotLearnerPrediction(learner = learners[[2]], task = iris.task)
{% endhighlight %}

![plot of chunk polynomial-radial-svm](/figures/2015-07-28-Visualisation-of-predictions/polynomial-radial-svm-1.svg)

{% highlight r %}
plotLearnerPrediction(learner = learners[[3]], task = iris.task)
{% endhighlight %}

![plot of chunk polynomial-radial-svm](/figures/2015-07-28-Visualisation-of-predictions/polynomial-radial-svm-2.svg)

Note that the intensity of the colors also indicates the certainty of the prediction and that this example is probably a rare case where the linear kernel performs best. although this is likely only the case because we didn't optimize the parameters for the radial kernel.

## Quadratic Discriminant Analysis

{% highlight r %}
plotLearnerPrediction(learner = learners[[4]], task = iris.task)
{% endhighlight %}

![plot of chunk qda](/figures/2015-07-28-Visualisation-of-predictions/qda-1.svg)

A well known classificator from the basic course of statistics delivers a similar performance as the SVMs.

## Random Forest

{% highlight r %}
plotLearnerPrediction(learner = learners[[5]], task = iris.task)
{% endhighlight %}

![plot of chunk randomforest](/figures/2015-07-28-Visualisation-of-predictions/randomforest-1.svg)

A completely different picture is generated by the random forest.
Here you can see that the whole data set is used to generate the model and as a result it looks like it gives a perfect fit but obviously you wouldn't use the train data to evaluate your model.
And the results of the 10-fold cross validation indicate that the random forest is actually not better then the others.

## Nearest Neighbour

{% highlight r %}
plotLearnerPrediction(learner = learners[[6]], task = iris.task)
{% endhighlight %}

![plot of chunk knn](/figures/2015-07-28-Visualisation-of-predictions/knn-1.svg)

In the default setting knn just look for 'k=1' neighbor and as a result the classifier does not return probabilities but only the class labels.


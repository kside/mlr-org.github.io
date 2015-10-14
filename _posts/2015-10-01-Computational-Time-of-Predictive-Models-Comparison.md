---
title: "Computational Time of Predictive Models - A comparison of mlr and caret"
author: "jakob"
layout: post
draft: true
---
In [a recent blogpost](http://freakonometrics.hypotheses.org/20345) Arthur from freakonometrics had a closer look into the computational times of different classification methods.
Interestingly he found out that that *caret* was way slower than calling the method directly in *R*.
It was pointed out in the comments that caret automatically does some kind of resampling and tuning and this makes it obviously slower.
So we have to keep in mind that for *caret* `train` always means parameter tuning as well - although it is not always clear which parameters and regions are taken into account.
For our comparison we will keep it fair and torn it off but still we got curious if *mlr* generates an computational overhead.

<!--more-->

Let's prepare the big data set accordingly to the original posts in freakonometrics. 
The big one with 710000 observations would need to much memory to store all the models in our little benchmark.
That's why we settle for 7100 and some more repititions.

{% highlight r %}
myocarde = read.table("http://freakonometrics.free.fr/myocarde.csv", head=TRUE, sep=";")
levels(myocarde$PRONO) = c("Death","Survival")
idx = rep(1:nrow(myocarde), each=100)
TPS = matrix(NA,30,10)
myocarde_large = myocarde[idx,]
k = 23
M = data.frame(matrix(rnorm(k*nrow(myocarde_large)), nrow(myocarde_large), k))
names(M) = paste("X", 1:k, sep="")
myocarde_large = cbind(myocarde_large,M)
dim(myocarde_large)
{% endhighlight %}



{% highlight text %}
## [1] 7100   31
{% endhighlight %}

We will benchmark the calls using the small and practical package *microbenchmark*.
So first we define only the calls we want to benchmark.
For fairness we always set the glm model parameter to FALSE so that the model data.frame is not included redundantly.

{% highlight r %}
library(randomForest)
glm.call = function() {glm(PRONO~., data=myocarde_large, family="binomial", model = FALSE)}
rf.call = function() {set.seed(1); randomForest(PRONO~., data=myocarde_large, ntree=500)}
{% endhighlight %}

For *caret* we turn off any special training method to get as close as possible to what the original `glm` does.
Same for the *randomForest*. 
Although we had some difficulties finding out how to train the *randomForest* in the default way. 

{% highlight r %}
library(caret)
glm.caret.call = function() {
  caret::train(
    PRONO~., 
    data = myocarde_large, 
    method="glm", 
    model = FALSE,
    trControl = trainControl(method = "none"))
  }
rf.caret.call = function() {
  set.seed(1)
  caret::train(
    PRONO~.,
    data = myocarde_large,
    method = "rf",
    ntree = 500,
    trControl = trainControl(method = "none"),
    tuneGrid = data.frame(mtry = floor(sqrt(ncol(myocarde_large)-1))))
  }
{% endhighlight %}

Finally we define the call for *mlr*. 

{% highlight r %}
library(mlr)
lrn.glm = makeLearner("classif.binomial")
lrn.rf = makeLearner("classif.randomForest", ntree = 500)
tsk = makeClassifTask(id = "myocarde", data = myocarde_large, target = "PRONO")
glm.mlr.call = function() {train(learner = lrn.glm, task = tsk)}
rf.mlr.call = function() {set.seed(1); train(learner = lrn.rf, task = tsk)}
{% endhighlight %}

Now, using *microbenchmark* we can compare the runtimes of each call.
Notice that the benchmark does two warm up iterations per default wich will not be taken into account for the comparison.

{% highlight r %}
library("microbenchmark")
glm.res = microbenchmark(
  glm = glm.call(), glm.caret = glm.caret.call(), glm.mlr = glm.mlr.call(), times = 20, unit = "s")
rf.res = microbenchmark(
  rf = rf.call(), rf.caret = rf.caret.call(), rf.mlr = rf.mlr.call(), times = 20, unit = "s")
boxplot(glm.res, unit = "s")
{% endhighlight %}

![plot of chunk unnamed-chunk-5](../figures/2015-10-01-Computational-Time-of-Predictive-Models-Comparison/unnamed-chunk-5-1.svg) 

{% highlight r %}
boxplot(rf.res, unit = "s")
{% endhighlight %}

![plot of chunk unnamed-chunk-5](../figures/2015-10-01-Computational-Time-of-Predictive-Models-Comparison/unnamed-chunk-5-2.svg) 

We are happy that *mlr* doesn't bring you too much computational overhead.
Strangely *caret* still is a fair amount slower.
It's not clear to us what happens here but one reason might also be that the `caret.fit` object contains also the complete training data.

Let's have a look if we did get the same model for *glm* with each call? 

{% highlight r %}
fits = list(glm = glm.call(), glm.caret = glm.caret.call(), glm.mlr = glm.mlr.call(),
            rf = rf.call(), rf.caret = rf.caret.call(), rf.mlr = rf.mlr.call())
all.equal(fits$glm.caret$finalModel$coefficients, fits$glm$coefficients)
{% endhighlight %}



{% highlight text %}
## [1] TRUE
{% endhighlight %}



{% highlight r %}
all.equal(fits$glm.mlr$learner.model$coefficients, fits$glm$coefficients)
{% endhighlight %}



{% highlight text %}
## [1] TRUE
{% endhighlight %}



{% highlight r %}
all.equal(fit$rf.caret$finalModel$votes, fits$rf$votes)
{% endhighlight %}



{% highlight text %}
## Error in all.equal(fit$rf.caret$finalModel$votes, fits$rf$votes): object 'fit' not found
{% endhighlight %}



{% highlight r %}
all.equal(fit$rf.mlr$learner.model$votes, fits$rf$votes)
{% endhighlight %}



{% highlight text %}
## Error in all.equal(fit$rf.mlr$learner.model$votes, fits$rf$votes): object 'fit' not found
{% endhighlight %}

We were not able to let caret run exactly as the *randomForest* although all important parameters seem to be the same when we run `Map(all.equal, rf.caret.fit$finalModel, rf.fit)`. 

The object sizes:

{% highlight r %}
sapply(fits, function(x) format(object.size(x), units = "KB"))
{% endhighlight %}



{% highlight text %}
##         glm   glm.caret     glm.mlr          rf    rf.caret      rf.mlr 
## "7066.3 Kb" "9334.9 Kb" "7108.9 Kb"   "5287 Kb" "7640.3 Kb" "5353.1 Kb"
{% endhighlight %}

Maybe a reason for *caret*s inferior runtime is the fact that it stores more information in its model objects.
---
layout: post
title: Meta-sandwich with extra mustard
date: April 26, 2014
---

In an earlier post about sandwich standard errors for multi-variate meta-analysis, I [mentioned]({{site.url}}/Robust-meta-analysis-1/) that Beth Tipton has recently proposed small-sample corrections for the covariance estimators and t-tests, based on the bias-reduced linearization approach of [McCaffrey, Bell, and Botts (2001)](http://www.amstat.org/sections/SRMS/Proceedings/y2001/Proceed/00264.pdf). 
You can find her forthcoming paper on the adjustments [here](http://blogs.cuit.columbia.edu/let2119/files/2013/03/Tipton-RVE-Small-to-share.pdf). 
My understanding is that these small-sample corrections are important because the uncorrected sandwich estimators can lead to under-statement of uncertainty and inflated type I error rates when a given meta-regression coefficient is estimated from only a small or moderately sized sample of independent studies (or clusters of studies). 
Moreover, it can be difficult to determine exactly when you have a large enough sample to trust the uncorrected sandwiches. 

I wanted to try out these small-sample corrected sandwich estimators for a meta-analyses project that I'm working on. Beth and one of her students have written an R package called [robumeta](http://cran.r-project.org/web/packages/robumeta/index.html) that implements the sandwich covariance estimator and small-sample corrections as described in her paper. 
However, for my project I want to use the [metafor package](http://www.metafor-project.org/), which doesn't provide these methods. 
I've therefore created a set of functions that implement the sandwich covariance estimators and small-sample corrections for models estimated using the `rma.mv` function in `metafor`. 
Here is [the complete code](https://gist.github.com/jepusto/11302318). Sorry, there's no further documentation at the moment (beyond the rest of this post).

### Consistency with robumeta

In order to check that the functions are correct, I compared the results generated by `robumeta` with the results from `metafor` plus my functions. Here's one example (I looked at a few others as well). First, the robumeta results:

{% highlight r %}
library(grid)
library(robumeta)
data(hierdat)

robu_hier <- robu(effectsize ~ males + binge,
            data = hierdat, modelweights = "HIER",
            studynum = studyid,
            var.eff.size = var, small = TRUE)
robu_hier
{% endhighlight %}



{% highlight text %}
## RVE: Hierarchical Effects Model with Small-Sample Corrections 
## 
## Model: effectsize ~ males + binge 
## 
## Number of clusters = 15 
## Number of outcomes = 68 (min = 1 , mean = 4.53 , median = 2 , max = 29 )
## Omega.sq = 0.1146972 
## Tau.sq = 0.06797866 
## 
##             Estimate  StdErr t-value  dfs P(|t|>) 95% CI.L 95% CI.U Sig
## 1 intercept  -0.0989 0.32140  -0.308 1.79 0.79045  -1.6511   1.4533    
## 2     males   0.0020 0.00441   0.454 1.88 0.69689  -0.0182   0.0222    
## 3     binge   0.6799 0.12156   5.594 4.18 0.00439   0.3482   1.0117 ***
## ---
## Signif. codes: < .01 *** < .05 ** < .10 *
## ---
## Note: If df < 4, do not trust the results
{% endhighlight %}

To maintain consistency, I first need to calculate the approximate weights used in `robumeta` and then fit the model in `metafor` using these fixed weights. 


{% highlight r %}
devtools::source_gist("https://gist.github.com/jepusto/11302318")

hierdat$var_HTJ <- hierdat$var + robu_hier$mod_info$omega.sq + robu_hier$mod_info$tau.sq

meta_hier <- rma.mv(yi = effectsize ~ males + binge, 
                V = var_HTJ, 
                data = hierdat, method = "FE")
meta_hier$cluster <- hierdat$studyid

RobustResults(meta_hier)
{% endhighlight %}



{% highlight text %}
##             Estimate  Std. Error    t value       df    Pr(>|t|)
## intrcpt -0.098869582 0.321400179 -0.3076214 1.788350 0.790446059
## males    0.002002043 0.004410552  0.4539212 1.879142 0.696887075
## binge    0.679929801 0.121556887  5.5935111 4.182783 0.004385654
{% endhighlight %}

The estimated covariance matrices match:

{% highlight r %}
all.equal(sandwich(meta_hier, meat.=meatBRL), 
          robu_hier$VR.r, 
          check.attributes=FALSE)
{% endhighlight %}



{% highlight text %}
## [1] TRUE
{% endhighlight %}

It can also be verified that the p-values based on the Satterthwaite degrees of freedom agree.

### Use with metafor

Of course, the point of writing functions that work with `rma.mv` objects is not to replicate `robumeta` results, but to take advantage of `metafor`'s flexibility. Rather than estimate the model with `robumeta`, typically one would estimate the variance components in `metafor` and then calculate the sandwich covariance estimates and small-sample corrections. For instance:


{% highlight r %}
meta_REML <- rma.mv(yi = effectsize ~ males + binge, 
                V = var, random = list(~ 1 | esid, ~ 1 | studyid), 
                data = hierdat,
                method = "REML")
meta_REML
{% endhighlight %}



{% highlight text %}
## 
## Multivariate Meta-Analysis Model (k = 68; method: REML)
## 
## Variance Components: 
## 
##             estim    sqrt  nlvls  fixed   factor
## sigma^2.1  0.1566  0.3957     68     no     esid
## sigma^2.2  0.0000  0.0000     15     no  studyid
## 
## Test for Residual Heterogeneity: 
## QE(df = 65) = 297.0172, p-val < .0001
## 
## Test of Moderators (coefficient(s) 2,3): 
## QM(df = 2) = 27.2659, p-val < .0001
## 
## Model Results:
## 
##          estimate      se     zval    pval    ci.lb   ci.ub     
## intrcpt   -0.1118  0.2474  -0.4520  0.6513  -0.5966  0.3730     
## males      0.0022  0.0034   0.6467  0.5178  -0.0044  0.0088     
## binge      0.6744  0.1313   5.1349  <.0001   0.4170  0.9319  ***
## 
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
{% endhighlight %}



{% highlight r %}
RobustResults(meta_REML)
{% endhighlight %}



{% highlight text %}
##             Estimate  Std. Error    t value       df    Pr(>|t|)
## intrcpt -0.111796564 0.318156355 -0.3513888 1.794988 0.762200367
## males    0.002173683 0.004380026  0.4962718 1.882842 0.671549040
## binge    0.674435042 0.121660936  5.5435628 4.167780 0.004585142
{% endhighlight %}

One advantage here is that it's possible to compare the model-based standard errors to the robust ones. In this instance, the two are fairly similar. However, the degrees of freedom estimated in the robust results indicate that the model-based standard errors (based on normal approximations) may be much too narrow.

### Differences between robumeta and my implementation

There are two important differences between the approach implemented in `robumeta` and the approach based on `metafor` and the code that I've provided. The first is that `robumeta` uses moment estimators for the variance components, whereas `metafor` uses restricted- or full maximum likelihood. The estimated between-study heterogeneity (and for the hierarchical effects model, the within-study heterogeneity as well) will therefore differ to some degree. 

The second, and perhaps more crucial, distinction has to do with the choice of weights. Weights are used for two purposes: to estimate the fixed effects and to calculate the small-sample correction. The `robumeta` package uses diagonal weights for both purposes. Using diagonal weights in calculating the fixed effects means that the resulting point estimates will be equivalent to those from a weighted ordinary least squares regression: 


{% highlight r %}
WOLS <- lm(effectsize ~ males + binge, data = hierdat, weights = 1 / var_HTJ)
coef(WOLS)
{% endhighlight %}



{% highlight text %}
##  (Intercept)        males        binge 
## -0.098869582  0.002002043  0.679929801
{% endhighlight %}



{% highlight r %}
all.equal(coef(WOLS), as.numeric(robu_hier$b.r), check.attributes = FALSE)
{% endhighlight %}



{% highlight text %}
## [1] TRUE
{% endhighlight %}

A subtler point is that `robumeta` uses the inverse weights for purposes of calculating the small sample-correction. The small sample correction involves choosing a "working" or "target" covariance matrix towards which to adjust the sandwich estimator. If the working covariance model is correct, then the BRL covariance estimator is exactly unbiased. The working matrix is also used to determine the Satterthwaite degrees of freedom. In `robumeta`, the working covariance matrix is taken to be inverse of the weights, which is also a diagonal matrix. Thus, the BRL correction amounts to assuming independence among all of the effect sizes. This may sound somewhat counter-intuitive, but  some simulation results (reported in Beth's paper, referenced above) suggest that the resulting estimators perform well even when the working independence assumption is not correct.

In contrast to the `robumeta` weights, `metafor` calculates the fixed effects based on a weighting matrix that is exactly inverse variance for given estimates of the variance components. Typically, the weighting matrix will be block-diagonal but may have off-diagonal entries corresponding to effect sizes drawn from the same study. Furthermore, my implementation of BRL uses the estimated covariance matrix derived from the posited random effects structure; in other words, the working covariance structure is taken to be the same as the model specified in the `metafor` call. This seems sensible to me, although I do not have any evidence regarding its performance relative to the alternatives. It is possible that any gains in asymptotic efficiency from using exactly inverse variance weights are outweighed by some sort of instability in small samples. It's also possible that the performance of the different approaches to weighting might depend on which variance component estimators are used (i.e., MOM vs. REML). 

Neither implementation that I've described above is fully general. Following the generalized estimating equation framework, a fully general implementation would allow the user to specify an arbitrary weight matrix in addition to a working covariance structure. The weighting matrix would be used for purposes of estimating the fixed effects. The working covariance model would be estimated (based on MOM or REML or what-not) and then used for purposes of BRL adjustment. Of course, this fully general formulation may well be more complicated than what most analysts would actually need or use (especially for linear mixed models), except perhaps when dealing with complex survey data. 
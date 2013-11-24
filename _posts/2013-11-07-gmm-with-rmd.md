---
layout: post
title: "Nonlinear Gmm with R - Example with a logistic regression"
description: ""
tags: [tutorial]
---
{% include JB/setup %}

<head>
<meta http-equiv="Content-Type" content="text/html; charset=utf-8"/>

<!-- MathJax scripts -->
<script type="text/javascript" src="https://c328740.ssl.cf1.rackcdn.com/mathjax/2.0-latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML">
</script>

</head>


<body>
<p>In this post, I will explain how you can use the R <code>gmm</code> package to estimate an non-linear model, and more specifically a logit model. For my research, I have to estimate euler equations using the Generalized Method of Moments. I contacted Pierre Chaussé, the creator of the <code>gmm</code> library for help, since I was having some difficulties. I am very grateful for his help (without him, I&#39;d still probably be trying to estimate my model!).</p>

<h3>Theoretical background and data set</h3>

<p>I will not dwell in the theory too much, because you can find everything you need <a href="https://en.wikipedia.org/wiki/Generalized_method_of_moments">here</a>. As an example, I&#39;ll use data from Marno Verbeek&#39;s <em>A guide to modern Econometrics</em> on page 197. You can download the data from the book&#39;s companion page <a href="http://www.econ.kuleuven.ac.be/gme/">here</a> under the section <em>Data sets</em> or from the <code>Ecdat</code> package in R. I use the data set from Gretl though, as the dummy variables are numeric (instead of class <code>factor</code>) which makes life easier when writing your own functions. You can get the data set <a href="/assets/files/benefits.R">here</a>. </p>

<h3>Implementation in R</h3>

<p>I don&#39;t estimate the exact same model, but only use a subset of the variables available in the data set. Keep in mind that this post is just for illustration purposes.</p>

<p>First load the <code>gmm</code> package and load the data set:</p>

<pre><code class="r">require(&quot;gmm&quot;)
data &lt;- read.table(&quot;/home/cbrunos/Copy/Documents/Work/github_page/b-rodrigues.github.com/assets/files/benefits.R&quot;, header = T)

attach(data)
</code></pre>

<p>We can then estimate a logit model with the <code>glm()</code> function:</p>

<pre><code class="r">native &lt;- glm(y ~ age + age2 + dkids + dykids + head + male + married + rr +  rr2, family = binomial(link = &quot;logit&quot;), na.action = na.pass)

summary(native)
</code></pre>

<pre><code>## 
## Call:
## glm(formula = y ~ age + age2 + dkids + dykids + head + male + 
##     married + rr + rr2, family = binomial(link = &quot;logit&quot;), na.action = na.pass)
## 
## Deviance Residuals: 
##    Min      1Q  Median      3Q     Max  
## -1.889  -1.379   0.788   0.896   1.237  
## 
## Coefficients:
##             Estimate Std. Error z value Pr(&gt;|z|)   
## (Intercept) -1.00534    0.56330   -1.78   0.0743 . 
## age          0.04909    0.02300    2.13   0.0328 * 
## age2        -0.00308    0.00293   -1.05   0.2924   
## dkids       -0.10922    0.08374   -1.30   0.1921   
## dykids       0.20355    0.09490    2.14   0.0320 * 
## head        -0.21534    0.07941   -2.71   0.0067 **
## male        -0.05988    0.08456   -0.71   0.4788   
## married      0.23354    0.07656    3.05   0.0023 **
## rr           3.48590    1.81789    1.92   0.0552 . 
## rr2         -5.00129    2.27591   -2.20   0.0280 * 
## ---
## Signif. codes:  0 &#39;***&#39; 0.001 &#39;**&#39; 0.01 &#39;*&#39; 0.05 &#39;.&#39; 0.1 &#39; &#39; 1
## 
## (Dispersion parameter for binomial family taken to be 1)
## 
##     Null deviance: 6086.1  on 4876  degrees of freedom
## Residual deviance: 5983.9  on 4867  degrees of freedom
## AIC: 6004
## 
## Number of Fisher Scoring iterations: 4
</code></pre>

<p>Now comes the interesting part: how can you estimate such a non-linear model with the <code>gmm()</code> function from the <code>gmm</code> package? </p>

<p>For every estimation with the Generalized method of moments, you will need valid moment conditions. It turns out that in the case of the logit model, this moment condition is quite simple:</p>

$$ 
E[X' * (Y-\Lambda(X'\theta))] = 0
$$

<p>where \( \Lambda \) is the logistic function. Let&#39;s translate this condition into code. First, we need the logistic function:</p>

<pre><code class="r">logistic &lt;- function(theta, data) {
    return(1/(1 + exp(-data %*% theta)))
}
</code></pre>

<p>and let&#39;s also define a new data frame, to make our life easier with the moment conditions:</p>

<pre><code class="r">dat &lt;- data.matrix(cbind(y, 1, age, age2, dkids, dykids, head, male, married, 
    rr, rr2))
</code></pre>

<p>and now the moment condition itself:</p>

<pre><code class="r">moments &lt;- function(theta, data) {
    y &lt;- as.numeric(data[, 1])
    x &lt;- data.matrix(data[, 2:11])
    m &lt;- x * as.vector((y - logistic(theta, x)))
    return(cbind(m))
}
</code></pre>

<p>To use the <code>gmm()</code> function to estimate our model, we need to specify some initial values to get the maximization routine going. One neat trick is simply to use the coefficients of a linear regression; I found it to work well in a lot of situations:</p>

<pre><code class="r">init &lt;- (lm(y ~ age + age2 + dkids + dykids + head + male + married + rr + rr2))$coefficients
</code></pre>

<p>And finally, we have everything to use <code>gmm()</code>:</p>

<pre><code class="r">my_gmm &lt;- gmm(moments, x = dat, t0 = init, type = &quot;iterative&quot;, crit = 1e-25, wmatrix = &quot;optimal&quot;, method = &quot;Nelder-Mead&quot;, control = list(reltol = 1e-25, maxit = 20000))

summary(my_gmm)
</code></pre>

<pre><code>## 
## Call:
## gmm(g = moments, x = dat, t0 = init, type = &quot;iterative&quot;, wmatrix = &quot;optimal&quot;, 
##     crit = 1e-25, method = &quot;Nelder-Mead&quot;, control = list(reltol = 1e-25, 
##         maxit = 20000))
## 
## 
## Method:  iterative 
## 
## Kernel:  Quadratic Spectral
## 
## Coefficients:
##              Estimate    Std. Error  t value     Pr(&gt;|t|)  
## (Intercept)  -0.9090571   0.5751429  -1.5805761   0.1139750
## age           0.0394254   0.0231964   1.6996369   0.0891992
## age2         -0.0018805   0.0029500  -0.6374640   0.5238227
## dkids        -0.0994031   0.0842057  -1.1804799   0.2378094
## dykids        0.1923245   0.0950495   2.0234150   0.0430304
## head         -0.2067669   0.0801624  -2.5793498   0.0098987
## male         -0.0617586   0.0846334  -0.7297189   0.4655620
## married       0.2358055   0.0764071   3.0861736   0.0020275
## rr            3.7895781   1.8332559   2.0671300   0.0387219
## rr2          -5.2849002   2.2976075  -2.3001753   0.0214383
## 
## J-Test: degrees of freedom is 0 
##                 J-test               P-value            
## Test E(g)=0:    0.00099718345776501  *******            
## 
## #############
## Information related to the numerical optimization
## Convergence code =  10 
## Function eval. =  17767 
## Gradian eval. =  NA
</code></pre>

<p>Please, notice the options <code>crit=1e-25,method=&quot;Nelder-Mead&quot;,control=list(reltol=1e-25,maxit=20000)</code>: these options mean that the Nelder-Mead algorithm is used, and to specify further options to the Nelder-Mead algorithm, the <code>control</code> option is used. This is very important, as Pierre Chaussé explained to me: non-linear optimization is an art, and most of the time the default options won&#39;t cut it and will give you false results. To add insult to injury, the Generalized Method of Moments itself is very capricious and you will also have to play around with different initial values to get good results.  </p>

<p>Here, the results are quite satisfactory, and the J-Test does not reject the moment condition!</p>

<p>Should you notice any error whatsoever, do not hesitate to tell me.</p>

</body>



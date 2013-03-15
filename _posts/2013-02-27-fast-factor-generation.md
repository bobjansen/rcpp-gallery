---
title: Fast factor generation with Rcpp
author: Kevin Ushey
license: GPL (>= 2)
tags: factor sugar
summary: We can make use of Rcpp sugar to implement a faster factor generator.
layout: post
src: 2013-02-27-fast-factor-generation.Rmd
---

Recall that factors are really just integer vectors with 'levels', 
i.e., character labels that get mapped to each integer in the vector. 
How can we take an arbitrary character, integer, numeric, or logical vector 
and coerce it to a factor with Rcpp? It's actually quite easy with Rcpp sugar:


{% highlight cpp %}
#include <Rcpp.h>
using namespace Rcpp;

template <int RTYPE>
IntegerVector fast_factor_template( const Vector<RTYPE>& x ) {
  Vector<RTYPE> levs = sort_unique(x);
  IntegerVector out = match(x, levs);
  out.attr("levels") = as<CharacterVector>(levs);
  out.attr("class") = "factor";
  return out;
}

// [[Rcpp::export]]
SEXP fast_factor( SEXP x ) {
  switch( TYPEOF(x) ) {
    case INTSXP: return fast_factor_template<INTSXP>(x);
    case REALSXP: return fast_factor_template<REALSXP>(x);
    case STRSXP: return fast_factor_template<STRSXP>(x);
  }
  return R_NilValue;
}
{% endhighlight %}


Note a few things:

1. We template over the `RTYPE`; i.e., the internal type that R assigns to its
objects. For this example, we just need to know that the R types (as exposed
in an R session) map to internal C types as 
`integer -> INTSXP`, `numeric -> REALSXP`, and `character -> STRSXP`. 

2. We return an IntegerVector. Remember that factors are just
integer vectors with a `levels` attribute and class `factor`.

3. To generate our factor, we simply need to calculate the sorted unique
values (the levels), and then match our vector back to those levels.

4. Next, we can just set the attributes on the object so that R will interpret
it as a factor, rather than a plain old integer vector, when it's returned.

And a quick test:


{% highlight r %}
library(microbenchmark)
all.equal( factor( 1:10 ), fast_factor( 1:10 ) )
{% endhighlight %}



<pre class="output">
[1] TRUE
</pre>



{% highlight r %}
all.equal( factor( letters ), fast_factor( letters ) )
{% endhighlight %}



<pre class="output">
[1] TRUE
</pre>



{% highlight r %}
lets <- sample( letters, 1E5, replace=TRUE )
microbenchmark( factor(lets), fast_factor(lets) )
{% endhighlight %}



<pre class="output">
Unit: milliseconds
               expr   min    lq median    uq   max
1      factor(lets) 5.315 5.766  5.930 6.069 32.93
2 fast_factor(lets) 1.420 1.458  1.474 1.486 28.85
</pre>


(However, note that this doesn't handle `NA`s -- fixing that is left as an
exercise. Similarily for logical vectors -- it's not quite as simple as just
adding a call to a `LGLSXP` templated call, but it's still not tough -- use
`INTSXP` and set set the levels to FALSE and TRUE.)

We can demonstrate a simple example of where this might be useful with
tapply. `tapply(x, group, FUN)` is really just a wrapper to `lapply( split(x, group), FUN )`,
and `split` relies on coercing 'group' to a factor. Otherwise, `split` calls
`.Internal( split(x, group) )`, and trying to do better than an internal C
function is typically a bit futile. So, now that we've written this,
we can test a couple ways of performing a `tapply`-like function:


{% highlight r %}
x <- rnorm(1E5)
gp <- sample( 1:1000, 1E5, TRUE )
all( tapply(x, gp, mean) == unlist( lapply( split(x, fast_factor(gp)), mean ) ) )
{% endhighlight %}



<pre class="output">
[1] TRUE
</pre>



{% highlight r %}
all( tapply(x, gp, mean) == unlist( lapply( split(x, gp), mean ) ) )
{% endhighlight %}



<pre class="output">
[1] TRUE
</pre>



{% highlight r %}
rbenchmark::benchmark( replications=20, order="relative",
                tapply(x, gp, mean), 
                unlist( lapply( split(x, fast_factor(gp)), mean) ),
                unlist( lapply( split(x, gp), mean ) )
                )[,1:4]
{% endhighlight %}



<pre class="output">
                                             test replications elapsed
2 unlist(lapply(split(x, fast_factor(gp)), mean))           20   0.200
3              unlist(lapply(split(x, gp), mean))           20   0.731
1                             tapply(x, gp, mean)           20   1.444
  relative
2    1.000
3    3.655
1    7.220
</pre>


To be fair, tapply actually returns a 1-dimensional array rather than a vector,
and also can operate on more general arrays. However, we still do see a modest
speedup both for using lapply, and for taking advantage of our fast factor generator.
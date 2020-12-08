---
title: Finding the farthest points in a point cloud
excerpt: "A better algorithm for finding a subset of points that maximize the minimal distance between them."
category: r
tags: [qualpalr, color-science]
header:
  image: assets/images/farthest-points-header.png
---

My R package [qualpalr](https://github.com/jolars/qualpalr) selects qualitative
colors by projecting a bunch of colors (as points) to the three-dimensional
DIN99d color space wherein the distance between any pair colors approximate
their differences in appearance. The package then tries to choose the `n`
colors so that the minimal pairwise distance among them is maximized, that is,
we want the most similar pair of colors to be as dissimilar as possible.

This turns out to be much less trivial that one would suspect, which posts on
[Computational Science](http://scicomp.stackexchange.com/questions/20030/selecting-most-scattered-points-from-a-set-of-points),
[MATLAB Central](https://se.mathworks.com/matlabcentral/answers/42622-how-can-i-choose-a-subset-of-k-points-the-farthest-apart), 
[Stack Overflow](http://stackoverflow.com/questions/27971223/finding-largest-minimum-distance-among-k-objects-in-n-possible-distinct-position), and
and [Computer Science](http://cs.stackexchange.com/questions/22767/choosing-a-subset-to-maximize-the-minimum-distance-between-points)
can attest to.

Up til now, qualpalr solved this problem with a greedy approach. If we, for instance,
want to find `n` points we did the following.

```
M <- Compute a distance matrix of all points in the sample
X <- Select the two most distant points from M
for i = 3:n
    X(i) <- Select point in M that maximize the mindistance to all points in X
```

In R, this code looked like this (in two dimensions):


{% highlight r %}
set.seed(1)
# find n points
n <- 3
mat  <- matrix(runif(100), ncol = 2)

dmat <- as.matrix(stats::dist(mat))
ind <- integer(n)
ind[1:2] <- as.vector(arrayInd(which.max(dmat), .dim = dim(dmat)))

for (i in 3:n) {
  mm <- dmat[ind, -ind, drop = FALSE]
  k <- which.max(mm[(1:ncol(mm) - 1) * nrow(mm) + max.col(t(-mm))])
  ind[i] <- as.numeric(dimnames(mm)[[2]][k])
}

par(mfrow = c(1, 2))
plot(mat, asp = 1, xlab = "", ylab = "")
plot(mat, asp = 1, xlab = "", ylab = "")
points(mat[ind, ], pch = 19)
text(mat[ind, ], adj = c(0, -1.5))
{% endhighlight %}

![Numbers note the order the points were picked in.](/figure/posts/2016-10-30-farthest-points/greedy_approach-1.png)

While this greedy procedure is fast and works well for large values of `n`
it is quite inefficient in the example above. It is plain to see that there are
other subsets of 3 points that would have a larger minimum distance but because
we base our selection on the previous 2 points that were selected to be
maximally distant, the algorithm has to pick a suboptimal third point. The
minimum distance in our example is 0.7641.

The solution I came up with is based on a solution from
Schlomer et al. @schlomer_farthest-point_2011 who devised of an algorithm to
partition a sets of points into subsets whilst maximizing the minimal distance.
They used [delaunay triangulations](https://en.wikipedia.org/wiki/Delaunay_triangulation)
but I decided to simply use the distance matrix instead. The algorithm works as
follows.

```
M <- Compute a distance matrix of all points in the sample
S <- Sample n points randomly from M
repeat
    for i = 1:n
        M    <- Add S(i) back into M
        S(i) <- Find point in M\S with max mindistance to any point in S
until M did not change
```

Iteratively, we put one point from our candidate subset (S) back into the
original se (M) and check all distances between the points in S to those in
M to find the point with the highest minimum distance. Rinse and repeat until
we are only putting back the same points we started the loop with, which
always happens. Let's see how this works on the same data set we used above.



{% highlight r %}
r <- sample.int(nrow(dmat), n)

repeat {
  r_old <- r
  for (i in 1:n) {
    mm <- dmat[r[-i], -r[-i], drop = FALSE]
    k <- which.max(mm[(1:ncol(mm) - 1) * nrow(mm) + max.col(t(-mm))])
    r[i] <- as.numeric(dimnames(mm)[[2]][k])
  }
  if (identical(r_old, r)) break
}

par(mfrow = c(1, 2))
plot(mat, asp = 1, xlab = "", ylab = "")
plot(mat, asp = 1, xlab = "", ylab = "")
points(mat[r, ], pch = 19)
text(mat[r, ], adj = c(0, -1.5))
{% endhighlight %}

![The new algorithm for picking points.](/figure/posts/2016-10-30-farthest-points/new_approach-1.png)

Here, we end up with a minimum distance of 0.862. In
qualpalr, this means that we now achieve slightly more distinct colors.

## Performance

The new algorithm is slightly slower than the old, greedy approach and slightly
more verbose


{% highlight r %}
f_greedy <- function(data, n) {
  dmat <- as.matrix(stats::dist(data))
  ind <- integer(n)
  ind[1:2] <- as.vector(arrayInd(which.max(dmat), .dim = dim(dmat)))
  for (i in 3:n) {
    mm <- dmat[ind, -ind, drop = FALSE]
    k <- which.max(mm[(1:ncol(mm) - 1) * nrow(mm) + max.col(t(-mm))])
    ind[i] <- as.numeric(dimnames(mm)[[2]][k])
  }
  ind
}

f_new <- function(dat, n) {
  dmat <- as.matrix(stats::dist(data))
  r <- sample.int(nrow(dmat), n)
  repeat {
    r_old <- r
    for (i in 1:n) {
      mm <- dmat[r[-i], -r[-i], drop = FALSE]
      k <- which.max(mm[(1:ncol(mm) - 1) * nrow(mm) + max.col(t(-mm))])
      r[i] <- as.numeric(dimnames(mm)[[2]][k])
    }
    if (identical(r_old, r)) return(r)
  }
}
{% endhighlight %}


{% highlight r %}
n <- 5
data <- matrix(runif(900), ncol = 3)
microbenchmark::microbenchmark(f_greedy(data, n), f_new(data, n), times = 1000L)
{% endhighlight %}



{% highlight text %}
## Unit: milliseconds
##               expr   min    lq  mean median    uq    max neval cld
##  f_greedy(data, n) 1.322 1.591 2.361  1.787 2.319  27.62  1000  a 
##     f_new(data, n) 1.600 2.343 3.605  2.724 3.675 120.25  1000   b
{% endhighlight %}

The newest development version of qualpalr now uses this updated algorithm which
has also been generalized and included as a new function in my R 
package [euclidr](https://github.com/jolars/euclidr) called `farthest_points`.

## References
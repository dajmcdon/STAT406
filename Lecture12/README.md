STAT406 - Lecture 12 notes
================
Matias Salibian-Barrera
2019-10-17

#### LICENSE

These notes are released under the “Creative Commons
Attribution-ShareAlike 4.0 International” license. See the
**human-readable version**
[here](https://creativecommons.org/licenses/by-sa/4.0/) and the **real
thing**
[here](https://creativecommons.org/licenses/by-sa/4.0/legalcode).

## Lecture slides

The lecture slides are [here](STAT406-19-lecture-12.pdf).

## Bagging

One strategy to obtain more stable predictors is called **Bootstrap
AGGregatING** (bagging). It can be applied to many predictors (not only
trees), and it generally results in larger improvements in prediction
quality when it is used with predictors that are flexible (low bias),
but highly variable.

The justification and motivation were discussed in class. Intuitively we
are averaging the predictions obtained from an estimate of the “average
prediction” we would have computed had we had access to several (many?)
independent training sets (samples).

There are several (many?) `R` packages implementing bagging for
different predictors, with varying degrees of flexibility (the
implementations) and user-friendliness. However, for pedagogical and
illustrative purposes, in these notes I will **bagg** by hand.

### Bagging by hand

Again, to simplify the discussion and presentation, in order to evaluate
prediction quality I will split the data (`Boston`) into a training and
a test set. We do this now:

``` r
library(rpart)
data(Boston, package='MASS')
set.seed(123456)
n <- nrow(Boston)
ii <- sample(n, floor(n/4))
dat.te <- Boston[ ii, ]
dat.tr <- Boston[ -ii, ]
```

I will now train \(N = 5\) trees and average their predictions. Note
that, in order to illustrate the process more clearly, I will compute
and store the \(n_e \times N\) predictions in a two-dimensional array
(aka a matrix), where \(n_e\) denotes the number of observations in the
test set. This is not the best (i.e. most efficient) way of implementing
**bagging**, but the main purpose here is to understand **exactly what**
we are doing. Also note that an alternative (better in terms of
reusability of the ensemble, but maybe still not the most efficient
option) is to store the \(N\) trees directly. This will also allow for
more elegant and easy to read code, and it is illustrated below, but
first we will use the the former (and hopefully clearer) strategy.

First create an array where we will store all the predictions:

``` r
N <- 5
myps <- array(NA, dim=c(nrow(dat.te), N))
con <- rpart.control(minsplit=3, cp=1e-3, xval=1)
```

The last object (`con`) contains my options to train large (potentially
overfitting) trees. As discussed in class, we now generate `N` bootstrap
samples by generating vectors of randomly sampled indices (with
replacement), the relevant lines of code are:

``` r
  ii <- sample(n.tr, replace=TRUE)
  tmp <- rpart(..., data=dat.tr[ii, ], ...)
```

where we train the trees on the data set `dat.tr[ii, ]`, which is the
boostrap sample. Then, for each of these trees we compute the
corresponding (vector of) predictions on the test set (`predict(tmp,
newdata=dat.te, type='vector')`) and store them. Putting all the pieces
together we get:

``` r
n.tr <- nrow(dat.tr)
set.seed(123)
for(j in 1:N) {
  ii <- sample(n.tr, replace=TRUE)
  tmp <- rpart(medv ~ ., data=dat.tr[ii, ], method='anova', control=con)
  myps[,j] <- predict(tmp, newdata=dat.te, type='vector')
}
```

The bagged predictions are the average of the predictions obtained with
each tree in the ensamble. In other words, for each point in `dat.te` we
need to compute the average of the predictions obtained with the `N`
different trees. Because of the way I stored the results in the matrix
`myps`, the bagged prediction of each point in `dat.te` is the average
of the corresponding row in the matrix `myps`. We can compute all these
(\(n_e\)) averages at once using `rowMeans()` as follows:

``` r
pr.bagg <- rowMeans(myps)
```

Finally, the estimated MSPE of the bagged ensemble of trees obtained
with our specific test set is:

``` r
with(dat.te, mean( (medv - pr.bagg)^2 ) )
```

    ## [1] 14.54751

We can now compare this with the similarly estimated MSPE of the pruned
tree:

``` r
myc <- rpart.control(minsplit=2, cp=1e-5, xval=10)
set.seed(123456)
bos.to <- rpart(medv ~ ., data=dat.tr, method='anova',
                control=myc)
b <- bos.to$cptable[which.min(bos.to$cptable[,"xerror"]),"CP"]
bos.t3 <- prune(bos.to, cp=b)
pr.t3 <- predict(bos.t3, newdata=dat.te, type='vector')
with(dat.te, mean((medv - pr.t3)^2) )
```

    ## [1] 16.59113

Would the quality of the bagged predictions improve if we use a larger
ensemble? For example, what happens if we **bagg** \(N = 10\) trees?

    ## [1] 13.97641

or \(N = 100\) trees?

    ## [1] 12.10982

or \(N = 1000\) trees?

    ## [1] 11.48381

Note that, at least for this test set, increasing the number of bagged
trees seems to improve the MSPE. However, the gain appears to decrease,
so it may not be worth the computational effort to use a larger **bag**
/ ensemble. Furthermore, one may also want to investigate whether this
is an artifact of this specific training / test partition, or if similar
patterns of MSPE are observed for other random training / test splits.
Below we try a different test/training split and repeat the bagging
experiment above:

``` r
set.seed(1234)
n <- nrow(Boston)
ii <- sample(n, floor(n/4))
dat.te <- Boston[ ii, ]
dat.tr <- Boston[ -ii, ]
Ns <- c(5, 10, 100, 1000)
all.results <- matrix(NA, length(Ns), 2) 
colnames(all.results) <- c('N', 'MSPE') 
n.tr <- nrow(dat.tr)
for(hh in 1:length(Ns)) {
  N <- Ns[hh]
  myps <- array(NA, dim=c(nrow(dat.te), N))
  set.seed(123)
  for(j in 1:N) {
    ii <- sample(n.tr, replace=TRUE)
    tmp <- rpart(medv ~ ., data=dat.tr[ii, ], method='anova', control=con)
    myps[,j] <- predict(tmp, newdata=dat.te, type='vector')
  }
  pr.bagg <- rowMeans(myps)
  all.results[hh, ] <- c(N, with(dat.te, mean( (medv - pr.bagg)^2 ) ) )
}
print(all.results)
```

    ##         N     MSPE
    ## [1,]    5 18.27651
    ## [2,]   10 17.50426
    ## [3,]  100 14.52966
    ## [4,] 1000 14.27511

The pattern is in fact similar to the one we observed before: increasing
the size of the ensemble \(N\) helps, but the improvements become
smaller after a certain value of \(N\). You are **strongly encouraged**
to verify this by repeating the above experiment with different
train/test splits. Furthermore, a **very good exercise** is to explore
what happens with the MSPE of the bagged ensembles (for different values
of \(N\)) when the MSPE is estimated using cross-validation (instead of
using a specific test set). Do it\!

#### More efficient, useful and elegant implementation

I will now illustrate a possibly more efficient way to implement
bagging, namely storing the \(N\) trees (rather than their predictions
on a given data set). In this way one can re-use the ensemble (on any
future data set) without having to re-train the elements of the **bag**.
Since the idea is the same, I will just do it for ensemble of
\(N = 100\) trees. To simplify the comparison between this
implementation of bagging and the one used above, we first re-create the
original training / test split

``` r
set.seed(123456)
n <- nrow(Boston)
ii <- sample(n, floor(n/4))
dat.te <- Boston[ ii, ]
dat.tr <- Boston[ -ii, ]
```

Now, let’s create a `list` of 100 (empty) elements, each element of this
list will store a regression tree:

``` r
N <- 100
mybag <- vector('list', N)
```

Now, we train the \(N\) trees as before, but store them in the `list`
(without computing any predictions):

``` r
set.seed(123)
for(j in 1:N) {
  ii <- sample(n.tr, replace=TRUE)
  mybag[[j]] <- rpart(medv ~ ., data=dat.tr[ii, ], method='anova', control=con)
}
```

Given a new data set, in order to obtain the corresponding predictions
for each tree in the ensemble, one could either:

  - loop over the \(N\) trees, averaging the corresponding \(N\) vectors
    of predictions; or
  - use `sapply` (check the help page if you are not familiar with the
    `apply` functions in `R`).

The later option results in code that is much more elegant, efficient
(allowing for future uses of the ensemble), and compact. Of course both
give exactly the same results. Below we illustrate both strategies. If
we use the **first approach (loop)** we obtain the following estimated
MSPE using the test set:

``` r
pr.bagg2 <- rep(0, nrow(dat.te))
for(j in 1:N)
  pr.bagg2 <- pr.bagg2 + predict(mybag[[j]], newdata=dat.te) / N
with(dat.te, mean( (medv - pr.bagg2)^2 ) )
```

    ## [1] 12.10982

(compare it with the results we obtained before). Using the **second
approach (sapply)**:

``` r
pr.bagg3 <- rowMeans(sapply(mybag, predict, newdata=dat.te))
with(dat.te, mean( (medv - pr.bagg3)^2 ) )
```

    ## [1] 12.10982

Both results are of course identical.

### Bagging a regression spline

Bagging does not provide much of an advantage when applied to linear
predictors (can you explain why?) Nevertheless, let us try it on the
`lidar` data, which, as we did before, we randomly split into a training
and test set:

``` r
data(lidar, package='SemiPar')
set.seed(123456)
n <- nrow(lidar)
ii <- sample(n, floor(n/5))
lid.te <- lidar[ ii, ]
lid.tr <- lidar[ -ii, ]
```

Now fit a cubic spline, and estimate the MSPE using the test set:

``` r
library(splines)
a <- lm(logratio ~ bs(x=range, df=10, degree=3), data=lid.tr) 
oo <- order(lid.tr$range)
pr.of <- predict(a, newdata=lid.te)
mean( (lid.te$logratio - pr.of)^2 )
```

    ## [1] 0.007427559

We build an ensemble of 10 fits and estimate the corresponding MSPE
using the test set:

``` r
N <- 10 
myps <- matrix(NA, nrow(lid.te), N)
set.seed(123)
n.tr <- nrow(lid.tr)
for(i in 1:N) {
  ii <- sample(n.tr, replace=TRUE)
  a.b <- lm(logratio ~ bs(x=range, df=10, degree=3), data=lid.tr[ii,]) 
  myps[,i] <- predict(a.b, newdata=lid.te)
}
pr.ba <- rowMeans(myps)# , na.rm=TRUE)
mean( (lid.te$logratio - pr.ba)^2 )
```

    ## [1] 0.007552562

Note that the estimated MSPE is almost the same as the one of the
original single spline. Furthermore, adding more elements to the
ensemble does not seem to improve the estimated MSPEs:

``` r
N <- 100
myps <- matrix(NA, nrow(lid.te), N)
set.seed(123)
n.tr <- nrow(lid.tr)
for(i in 1:N) {
  ii <- sample(n.tr, replace=TRUE)
  a.b <- lm(logratio ~ bs(x=range, df=10, degree=3), data=lid.tr[ii,]) 
  myps[,i] <- predict(a.b, newdata=lid.te)
}
pr.ba <- rowMeans(myps)# , na.rm=TRUE)
mean( (lid.te$logratio - pr.ba)^2 )
```

    ## [1] 0.0075887

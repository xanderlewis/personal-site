---
title: Extracting colours from an image using k-means clustering
date: 2017-10-09 16:10:49
tags:
- colours
- data
- algorithm
---

# The problem
I wanted to write some software that would allow me to extract a set of colours from an image, and do it in a way that seems natural and takes human perception into consideration. A colour scheme can often sum up the 'vibe' of an entire image, and so I thought it would be a useful thing to be able to do.

So... I spent some time thinking of some ways that I could do this. I devised some fairly simple algorithms that would, for example, chop the image regularly into chunks and output the mean colour of each of these parts. Maybe extra layers could be added where the chunks are compared to each other and joined into groups, and maybe each colour could be recursively combined with another until the desired number of colours is reached. I quickly realised, though, that this problem had already been solved in the general case, and in a way that will work quite nicely.


# The solution
[K-means clustering](https://en.wikipedia.org/wiki/K-means_clustering) is a method through which a set of data points can be partitioned into several disjoint subsets where the points in each subset are deemed to be 'close' to each other (according to some metric). A common metric, at least when the points can be geometrically represented, is your bog standard euclidean distance function. The 'k' just refers to the number of subsets desired in the final output. It turns out that this approach is exactly what we need to divide our image into a set of colours.

# Our case
In our case, the 'data points' are colours, and the distance function is some measure of 'how different' two colours are. Our task is to group these colours into a given number of sets, and then calculate the mean colour of each set. Using the mean seems like a fairly sensible choice because you can imagine blurring your eyes whilst looking at the different colour clusters, and seeing a mean colour for each. However, we could instead use any other statistical measure (mode, median, or anything else!) and possibly get a better result.

# The implementation

Let's write this in **JavaScript**! ✨

## The data points
Each data point is a colour and can be represented as a point in an [RGB colour space](https://en.wikipedia.org/wiki/RGB_color_space).

In JavaScript, a single data point could look something like this:

```javascript
    // An array if we want to be general
    let colour = [100,168,92];

    // Or an object if we want to be more explicit
    let colour = {red: 100, green: 168, blue: 92};
```

## The distance function
Since we want to be able to calculate how similar two colours are, we need a function. This is another point where we have many choices, but a straightforward one is just to calculate the euclidean distance using the component values of each colour.

Our distance function could look like this:

```javascript
    // Distance function
    function euclideanDistance(a, b) {
        var sum = 0;
        for (let i = 0; i < a.length; i++) {
            sum += Math.pow(b[i] - a[i], 2);
        }
        return Math.sqrt(sum);
    }
```

Since we haven't specified a fixed number of components in our function, it will work for *n-dimenional* data points (with n components). This is useful in case we want to represent colour in a different way later on.

## The algorithm

The most common algorithm used for k-means clustering is called *Lloyd's algorithm* (although it is often known simply as *the* k-means algorithm). We're going to use that algorithm here.

We're going to create a set of objects called 'centroids', each of which defines a single, unique cluster.

<br>

A centroid has two things associated with it:
- a point within the range of the data set (the centroid's position)
- a set of data points from the data set (the points in the centroid's cluster)

<br>

There are three main steps to the algorithm:

1. **Initialisation**. Choose initial values for the centroids. In this case, we'll just choose a random point for each one.
2. **Assignment**. Assign each data point to the cluster whose mean (the centroid) is the least distance away.
3. **Update**. Set the new mean of each centroid to be the mean of all of data points associated with it (in the centroid's cluster).

The algorithm will perform **initialisation** once, and then perform **assignment** and **update** in order, repeatedly, until the algorithm converges.

The algorithm is said to have 'converged' when nothing changes between assignments. Basically, when the points make their mind up about which cluster they're a part of, we can stop looping.

<br>

### Helper functions

Let's get some helper functions defined: one for calculating the range of a given dataset, and one for generating a random integer within a given range.

```javascript
    // Calculate range of a one-dimensional data set
    function rangeOf(data) {
        return data.reduce(function(total,current) {
            if (current < total.min) { total.min = current; }
            if (current > total.max) { total.max = current; }
            return total;
        }, {min: data[0], max: data[0]});
    }

    // Calculate range of an n-dimensional data set
    function rangesOf(data) {
        var ranges = [];
        for (let i = 0; i < data[0].length; i++) {
            ranges.push(rangeOf(data.map(x => x[i])));
        }
        return ranges;
    }

    // Generate random integer in a given closed interval
    function randomIntBetween(a, b) {
        return Math.floor(Math.random() * (b - a + 1)) + a;
    }
```

<br>

Assuming we've got those two, we can now write the code for the three steps of the algorithm:

<br>

### Step One - Initialisation

For the number of centroids desired (k), we generate a random integer point in the range of the data set provided and append it to an array. Each point in the array represents the position of a centroid.

```javascript
    function initialiseCentroidsRandomly(data, k) {
        var ranges = rangesOf(data);
        var centroids = [];
        for (let i = 0; i < k; i++) {
            var centroid = [];
            for (var r in ranges) {
                centroid.push(randomIntBetween(ranges[r].min, ranges[r].max));
            }
            centroids.push(centroid);
        }
        return centroids;
    }
```

<br>

### Step Two - Assignment

This is where we assign data points to clusters. For each point, we check its distance to each centroid and, when we find the nearest, we add the point to the associated cluster.

```javascript
    function clusterDataPoints(data, centroids) {
        var clusters = [];
        centroids.forEach(function () {
            clusters.push([]);
        });
        data.forEach(function (point) {
            var nearestCentroid = centroids[0];
            centroids.forEach(function (centroid) {
                if (euclideanDistance(point, centroid) < euclideanDistance(point, nearestCentroid)) {
                    nearestCentroid = centroid;
                }
            });
            clusters[centroids.indexOf(nearestCentroid)].push(point);
        });
        return clusters;
    }
```

<br>

### Step Three - Update

For each cluster, we calculate the mean of its enclosing data points and set it as the associated centroid's position.

```javascript
    function getNewCentroids(clusters) {
        var centroids = [];
        clusters.forEach(function (cluster) {
            centroids.push(stats.meanPoint(cluster));
        });
        return centroids;
    }
```

<br>

### Oh, one more thing

There is one issue with this algorithm, and it's that sometimes the clusters become empty. There isn't much of a consensus on what to do when this happens, but some possible approaches are to:

- Remove the cluster (a bit silly)
- Assign a random data point to the cluster
- Assign the closest data point to the cluster
- Restart the algorithm and hope it doesn't happen again

Although the last option seems like a bit of a bodge, the algorithm (with the random initialisation) is non-deterministic and so simply restarting does work quite well. After all, this method is pretty much a heuristic and heuristics by definition are 'a bit of a bodge' ... at least compared to a solid *algorithm*.

<br>

## Pulling it all together

Now that we've got the basic functionality defined, all that's left is to write some code to call each function when required as defined by the algorithm.

I'll leave that to you as an exercise, but, if you really want to, you can see my implementation of it as a web app [here](https://github.com/xanderlewis/colour-palettes). 🌈

<br>

***

<br>

![](/content/images/2017/10/colour-palettes-screenshot.png)
Here's what it looks like after being fed the cover of a [Ryan Hemsworth](http://ryanhemsworth.com/) song.
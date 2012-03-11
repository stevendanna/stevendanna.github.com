---
layout: post
title: "Simple Tools: do.times and summarize"
date: 2012-03-10
comments: true
categories: unix shell R
---

Recently, a coworker has been sharing some higher-order shell
functions he has been writing, inspired by the first of these
articles:

- [Higher Order Functions in Zsh](http://yannesposito.com/Scratch/en/blog/Higher-order-function-in-zsh)
- [Higher-order shell](http://conway.rutgers.edu/~ccshan/wiki/blog/posts/Higher-order_shell/)

Both discuss small scripts and shell functions that allow you to more
easily compose simple unix tools to complete complex task.

My own `~/bin` contains similar higher-order functions and other
simple tools.  Like most unix tools, they do a single task and are
composable with other tools on the command line.  However, I am often
surprised by the number of heavy command-line users I see who don't
regularly encapsulate repetitive tasks or constructions into shell
functions or aliases.

In my experience, the majority of such tools

* take less than 5 minutes to write,
* require almost no maintenance, and
* prove repeatably useful after their initial creation.

This post shares two tools in my `~/bin` that I used this weekend.  I
created both while ago for entirely different purposes, but was still
able to easily use them together without modification.

## do.times N COMMAND

`do.times` executes COMMAND, N times.  For example,

```
$ do.times 5 echo hello
hello
hello
hello
hello
hello
```

The [tool itself](https://gist.github.com/2014470) is little more than
a wrapper to a shell `for` loop, but the typing it saves and the
semantic value it provides when constructing a command line has proven
valuable on numerous occasions.

## summarize [OPTIONS] [FILE]

`summarize` reads columns of data from either a file given as an
argument or its standard input and provides basic summary statistics.
I originally wrote this to do quick spot checks of data files I was
working on before sending them off to collaborators.

Here is an example of its output:
```
$ summarize --sep="," --header data_file
    wombats            frogs
 Min.   :0.01911   Min.   :0.02979
 1st Qu.:0.22101   1st Qu.:0.32504
 Median :0.51178   Median :0.50530
 Mean   :0.51318   Mean   :0.51650
 3rd Qu.:0.79354   3rd Qu.:0.71484
 Max.   :0.99561   Max.   :0.99827
 It requires
Covariance Matrix:
            wombats       frogs
wombats 0.084647845 0.006553818
frogs   0.006553818 0.074872790

Column Sums:
 wombats    frogs
51.31770 51.65021

Number of Rows:
[1] 100

First 6 Rows:
     wombats     frogs
1 0.15767016 0.3545217
2 0.42022851 0.5092576
3 0.22144630 0.6796292
4 0.84320607 0.6749124
5 0.07491303 0.3430995
6 0.65910419 0.3053271
```

While it doesn't seem like much, these basic statistics are often all
one needs to ensure they are sending the correct data to a teammate or
to quickly answer a basic question.

[summarize](https://gist.github.com/2014462) uses Rscript, an
executable shipped with R that allows one to create scripts using R.
Recently, I added a dependency on the CRAN package `optparse` to make
handling options a bit more straightforward.  With the exception of
the option parsing, the R code itself is likely easily understood by
anyone who has used R.

# Smoke Testing Speed Improvements

This weekend I have been working on a small set of improvements to
make some not-so-simple tools I use on a regular basis a bit
faster. Combining `do.times` and `summarize` allowed me to quickly
generate a smoke test for whether the speed improvements I was
implementing were working.  To protect the innocent, I'll use `git
--version` as an example of the command I wanted to test:

```
$ do.times 100 time git --version 2>&1 | awk '/real/ {print substr($2,3,5)}' | summarize
       V1
 Min.   :0.00100
 1st Qu.:0.00100
 Median :0.00100
 Mean   :0.00116
 3rd Qu.:0.00100
 Max.   :0.00400

Covariance Matrix:
             V1
V1 2.771717e-07

Column Sums:
   V1
0.116

Number of Rows:
[1] 100

First 6 Rows:
     V1
1 0.004
2 0.004
3 0.001
4 0.002
5 0.001
6 0.001
```

While this isn't scientific testing by any means, it was enough to
keep me moving in the right direction, and simple enough that the cost
of creating and running the test was practically zero.

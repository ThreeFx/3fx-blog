---
title: "Long monotone trails"
date: 2019-12-10T10:10:10+01:00
tags: graph-theory, short-and-sweet
---

During a recent dinner discussion I was presented with a beautiful proof to the
following problem: Given an arbitrary graph <math><mi>G</mi> <mo>=</mo> (<mi>V</mi>, <mi>E</mi>)</math>, and an
ordering <math><mi>ϕ</mi></math> of <math><mi>E</mi></math>, what is the longest monotone trail
in <math><mi>G</mi></math> that is guaranteed to exist?

<!--more-->

A trail in a graph is a sequence of edges such that

 * adjacent edges share an incident vertex and
 * no edge is traversed twice

Take a few moments to think about this problem before reading the solution, as
it such a simple seeming problem is quite tricky: Clearly there are graphs where
there exists no trail of length greater than zero, namely the empty graph. Thus
the solution must somehow correlate with some graph property.

### The solution

It turns out that the longest monotone trail has length equal to the average
degree <math><mover><mi>d</mi> &#x203E;</mover></math> of <math><mi>G</mi></math>. The proof is truly
astouding:

Put a person on each vertex of <math><mi>G</mi></math>, and for each edge
<math><mi>e</mi></math> in <math><mi>E</mi></math>, traversed in order of
<math><mi>ϕ</mi></math>, exchange the two people standing at vertices incident
to <math><mi>e</mi></math>. Each person will walk a monotone trail on
<math><mi>G</mi></math>.

All people together traversed a distance of <math><mi>2</mi> <mo>\*</mo> |E|</math> units,
and since there are <math>|<mi>V</mi>|</math> people the average distance
traveled is <math><mi>2</mi> <mo>\*</mo> |<mi>E</mi>| / |<mi>V</mi>|</math>, which is
exactly the average degree of <math><mi>G</mi></math>. By the [pigeonhole
principle](https://en.wikipedia.org/wiki/Pigeonhole_principle) at least one
person must have traveled <math><mi>2</mi> <mo>\*</mo> |<mi>E</mi>| / |<mi>V</mi>|</math>
units, which concludes our proof.

It is not difficult to see that this result is best possible. If
<math><mi>G</mi></math> is a matching on <math><mi>2</mi> <mi>n</mi></math>
vertices, then the average degree in <math><mi>G</mi></math> is <math>1</math>,
and the longest monotone trail also has length <math>1</math>.

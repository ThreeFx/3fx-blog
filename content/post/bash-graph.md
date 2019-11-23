---
title: "Solving graph problems in bash"
date: 2019-11-22T18:30:40+01:00
---

Many interesting problems in computer science are expressible as problems on
graphs: Finding the shortest path between two vertices, calculating a spanning
tree, finding a large independent set, etc. Often in Theoretical Computer
Science we only look for a theoretical solution - once we know how to model a
specific problem as graph we give an algorithm in pseudocode and leave it at
that. We will try an unconventional method practically solving these problems
using bash.

<!--more-->

At ETH, Computer Science Master students learn how to efficiently (and
practically) solve graph problems in the Algorithms Lab, using
[C++](https://en.wikipedia.org/wiki/C%2B%2B) and
[BGL](https://www.boost.org/doc/libs/1_66_0/libs/graph/doc/index.html)[^6]. In
this course it is not enough to come up with a theoretical algorithm. One also
have to implement and pass testcases as part of his solution. The BGL offers
many sophisticated graph algorithms, each one cleverly optimized in order to
make the generated code as fast as possible. It does its job well: It makes it
(relatively) easy to use sophisticated graph theoretic algorithms in practice.
But it is hugo and complicated. Is there maybe an easier way to solve graph
problems on a computer, without having to implement most of the boring
algorithms and without installing a bazillion dependencies?

It turns out that we can (ab)use just about any Linux system for this, and we do
not even need a C++ compiler, or any other compiler for that matter. All we need
is any run-of-the-mill shell (bash will do just fine), and some standard system
utilities such as `find(1)`, `tr(1)`, `paste(1)`, `cut(1)`, etc.[^1], which (for
GNU/Linux systems) are packaged in the so-called [GNU
`coreutils`](https://www.gnu.org/software/coreutils/). Most other systems
provide similar, if not the same, commands - they may be packaged differently
however.

### Pathfinding in trees

We will start with a simple problem: Given an undirected, acyclic and connected
graph, commonly known as _tree_, find the unique path between two vertices i and
j of said tree.

The first problem we have to solve is *representing* our graph. Recall from that
there are may possible ways to represent graphs: Adjacency lists, (for each
vertex list all its neighbors), Adjacency matrices (entry (i, j) is 1 iff edge
(i, j) exists), and many more. The most obvious solution would be to put all of
that information in a text file and then operate on that, but then we would be
no further than C++, since we would still have to implement most of the
algorithms by hand. So let us go up a step in the hierarchy: If files are bad,
can we maybe use directories instead?

It turns out that this works nicely, but only when our graph is a *rooted
tree*. In a rooted tree we choose an arbitrary vertex as root and then list all
vertices ordered by their distance from the root: First comes the root itself,
then the root's neighbors, then the root's neighbors' neighbors, etc. Let
us have a look at an example, a tree rooted at vertex 0:

{{< figure src="/post/images/rooted-tree.svg" width="300px" >}}

We can easily translate this tree into a directory structure:

```bash
mkdir -p 0/1/3/{4,5}
mkdir -p 0/2/6
```

How do we display our tree? The best utility I know for this task is aptly named
`tree(1)`. Sadly, it is not part of the GNU coreutils and you most likely have
to install it with your favorite package manager. Running `tree` then shows us
the tree we constructed:

```
% tree
.
└── 0
    ├── 1
    │   └── 3
    │       ├── 4
    │       └── 5
    └── 2
        └── 6
```

Now we can answer questions such as "How can I get from 0 to i?", where i is an
arbitrary vertex other[^4] of our tree using the `find(1)` command. The `find`
command traverses all files in a given directory, and can optionally search for
specific criteria, for example directory names. If we wanted to know how to get
from vertex 0 to vertex 3 we can run the following command:

```
% find 0 -name 3
0/1/3
```

which tells us that we can go from 0 to 3 via vertex 1. In fact, we can
generalize this to arbitrary vertices i and j: First we check how to get from 0
to i, and then from 0 to j, and finally we have to find the common ancestor of
both paths. Then we reverse the path from 0 to i (now it is a path from i to 0),
walk along it until we hit the common ancestor of i and j, and then walk down
the path from the common ancestor to j.

The most difficult part of this procedure is finding the common ancestor. Lucky
for us the package `diffutils` provides the `cmp(1)` command, which can tell us
the first byte at which two files differ. The output of cmp is a bit verbose,
but using `cut(1)` and `tr(1)` from the coreutils we can extract the
relevant information:

{{< highlight bash >}}
% cmp <(echo "testa") <(echo "testb")
/proc/self/fd/11 /proc/self/fd/12 differ: byte 5, line 1
% cmp <(echo "testa") <(echo "testb") | cut -d " " -f 5 | tr -cd "[:digit:]"
5
{{< /highlight >}}

How does this work? The program `cut` takes a delimiter (`-d`) and some position
number (`-f`) and returns the string at the given position (1-indexed) when
splitting on the delimiter specified by `-d`. We use `tr` to delete (`-d`) all
non-digits (`-c` stands for the complement of the character set) to ensure that
we get `5` as result, and not `5,`. So far so good, let us look where we are
now:

{{< highlight bash >}}
# In Bash arguments are passed implicitly and referenced
# using $1, $2, etc. We use $1 as i and $2 as j
path_between() {
    path_to_i=$(find -name $1)
    path_to_j=$(find -name $2)

    first_diff_byte=$(cmp <(echo $path_to_i) <(echo $path_to_j) \
            | cut -d " " -f 5 \
            | tr -c -d '[:digit:]')
{{< /highlight >}}

It remains to find the common ancestor of i and j. Since we know where the paths
differ, we know up to which character they are identical. Using [bash variable
expansion](https://www.tldp.org/LDP/abs/html/parameter-substitution.html) we can
identify the common prefix, and using `cut` extract the vertex they have in
common. The `rev` command from the `util-linux` package reverses a string of
characters:

{{< highlight bash >}}
    common_prefix=${path_to_i:0:(( $first_diff_byte - 1 ))}
    common_ancestor=$(echo -n $common_prefix | rev | cut -d / -f 2 | rev)
{{< /highlight >}}

The part in the double parenthesis is subject to the `expr(1)` command[^2], which
and evaluates the expression as arithmetic expression. Now all we have left to
do is to stick all of it together:

{{< highlight bash >}}
    # if we omit local here we set $PATH...
    local path=$(echo ${path_to_i#"$common_prefix"} \
        | tr '/' $'\n' \
        | tac \
        | paste -s -d '/')
    path=$path/$common_ancestor/${path_to_j#"$common_prefix"}

    echo $path
}
{{< /highlight >}}

Using the `tr`, `tac` and `paste` commands (all part of coreutils) we can
reverse the path from i to the common ancestor of i and j. We do this by
changing the slashes which delimit our vertices to newlines and using
`tac`[^3] to reverse the order of our vertices. Finally
we can use `paste` to put the reversed vertices back together in a single string
and then append the path from the common ancestor to j.

Testing it on our input graph gives the following output:

```
% path_between 4 5
4/3/5
% path_between 5 4
5/3/4
% path_between 0 5
/.//1/3/5
% path_between 3 4
/1//4
```

It seems like it fails when the common ancestor is either i or j. We can remedy
thisby handling these cases separately: If i is the common ancestor, we can
simply take the path from i to j minus the path from 0 to the ancestor of i.
Else if j is the common ancestor then we have to first reverse the path from j
to i (so we get the path from i to j) and the remove the path to j's ancestor.

{{< highlight bash >}}
if [[ $common_prefix == $path_to_i ]]; then
    local path=$1${path_to_j#"$common_prefix"}
elif [[ $common_prefix == $path_to_j ]]; then
    local path=$(echo ${path_to_i#"$common_prefix"} \
        | tr '/' $'\n' \
        | tac \
        | paste -s -d /)
    path=$path$2
else
    # Do what we did before
fi
{{< /highlight >}}

Now it gives correct results for any query:

```
% path_between 3 4
3/4
% path_between 4 3
4/3
% path_between 6 4
6/2/0/1/3/4
% path_between 0 1
0/1
% path_between 1 0
1/0
```

That's it. In 20 lines of bash we wrote a pathfinding algorithm for trees. Not
only that, but we did it without needing to compile any code, only using tools
available on most Linux systems! I find that quite impressive.

### Pathfinding in general graphs

Restricting ourselves to rooted trees sounds pretty limiting doesn't it? Can we
generalize our model to arbitrary graphs? At first it seems like we cannot,
since then our file system (which is where we store our graph) does not support
cycles. After all, no directory can be a (grand-)parent of itself. But it turns
out that using [symbolic links](https://en.wikipedia.org/wiki/Symbolic_link) we
can "point" to other directories, which can in turn point to other directories,
which can point to other directories, etc. We can create symlinks using the `ln`
command (incidentally also found in coreutils), by writing `ln -s <directory
point to> <name of the link>`. For each vertex we want to create a directory and
in that directory create symbolic links to all of its neighbors.  If you recall
the different forms of graph representation before this corresponds to adjacency
lists. Let us try this with a simple example:

{{< figure src="/post/images/graph.svg" width="100%" >}}

We can create the graph as follows:

```
% mkdir `seq 0 5`
% ln -s ../0 1
% ln -s ../1 0
% ln -s ../2 0
% ln -s ../0 2
% <continue for each edge>
% tree
.
├── 0
│   ├── 1 -> ../1
│   └── 2 -> ../2
├── 1
│   ├── 0 -> ../0
│   └── 2 -> ../2
├── 2
│   ├── 0 -> ../0
│   ├── 1 -> ../1
│   └── 4 -> ../4
├── 3
│   └── 4 -> ../4
├── 4
│   ├── 2 -> ../2
│   ├── 3 -> ../3
│   └── 5 -> ../5
└── 5   
    └── 4 -> ../4
```

Now let us look at the same problem as before: We want to specify two vertices i
and j, and want to find a path between them (if it exists).  Last time we used
the `find(1)` utility to find these. This time we will explicitly tell `find` in
which vertex i we would like to start by specifying it as the root of our
search:

```
% find 0/ -name 1
0/1
% find 0/ -name 3
<nothing>
```

Hmm, it seems as if `find` does not follow the symbolic links. Looking at
`find`'s manual page (`man find`) we see the following

```
OPTIONS The  -H, -L and -P options control the treatment of symbolic links.

[...]

-P     Never  follow  symbolic  links.  This is the default behaviour.  When
       find examines or prints information a file, and the file  is  a  sym‐
       bolic  link,  the information used shall be taken from the properties
       of the symbolic link itself.

-L     Follow symbolic links.  When  find  examines  or  prints  information
       about  files, the information used shall be taken from the properties
       of the file to which the link points, not from the link  itself  (un‐
       less  it  is  a broken symbolic link or find is unable to examine the
       file to which the link points).  Use of this option implies  -noleaf.
       If  you later use the -P option, -noleaf will still be in effect.  If
       -L is in effect and find discovers a symbolic link to a  subdirectory
       during  its  search, the subdirectory pointed to by the symbolic link
       will be searched.
```

Aha, we have to specify `find -L` if we want to follow symlinks. But our
symlinks induce cycles in the file system, what happens if `find` encounters a
cycle? Again the manpage has the answer for us:

```
The POSIX standard requires that find detects loops:

       The find utility shall detect infinite loops;  that  is,  entering  a
       previously visited directory that is an ancestor of the last file en‐
       countered.  When it detects an infinite loop, find shall write a  di‐
       agnostic message to standard error and shall either recover its posi‐
       tion in the hierarchy or terminate.
```

So we are good to go!

```
% find -L 0/ -name 3
find: File system loop detected; ‘0/2/4/5/4’ is part of the same file system loop as ‘0/2/4’.
0/2/4/3
find: File system loop detected; ‘0/2/4/3/4’ is part of the same file system loop as ‘0/2/4’.
find: File system loop detected; ‘0/2/4/2’ is part of the same file system loop as ‘0/2’.
find: File system loop detected; ‘0/2/1/2’ is part of the same file system loop as ‘0/2’.
find: File system loop detected; ‘0/2/1/0’ is part of the same file system loop as ‘0/’.
find: File system loop detected; ‘0/2/0’ is part of the same file system loop as ‘0/’.
find: File system loop detected; ‘0/1/2/4/5/4’ is part of the same file system loop as ‘0/1/2/4’.
0/1/2/4/3
find: File system loop detected; ‘0/1/2/4/3/4’ is part of the same file system loop as ‘0/1/2/4’.
find: File system loop detected; ‘0/1/2/4/2’ is part of the same file system loop as ‘0/1/2’.
find: File system loop detected; ‘0/1/2/1’ is part of the same file system loop as ‘0/1’.
find: File system loop detected; ‘0/1/2/0’ is part of the same file system loop as ‘0/’.
find: File system loop detected; ‘0/1/0’ is part of the same file system loop as ‘0/’.
```

We get two paths and a whole lot of errors that find detected a loop. We can
just ignore these:

```
% find 0/ -name 3 2>/dev/null
0/2/4/3
0/1/2/4/3
```

Perfect! Now the find command returns all the paths between 0 and 3 in the
graph. And it is much simpler than our previous solution: `find` does all the
work for us.

This approach can easily be extended to directed graphs by only symlinking in
the direction the edge goes, which means that we can find all paths between
vertices i and j in any graph in *a single line of bash*. That's pretty amazing,
don't you think? We can even improve the solution a bit: What if we only want *a*
path, and not all paths? The manual page of `find` once again has the answer:

```
-quit  Exit immediately.  No child processes will be left  running,  but  no
       more  paths specified on the command line will be processed.  For ex‐
       ample, find /tmp/foo /tmp/bar -print -quit will print only  /tmp/foo.
       Any  command  lines  which  have been built up with -execdir ... {} +
       will be invoked before find exits.  The exit status may or may not be
       zero, depending on whether an error has already occurred.
[...]
-print True; print the full file name on the standard output, followed by  a
       newline.
```

If we specify `-print` and `-quit` the `find` command will immediately exit when
it finds the first path:

```
% find -L 0/ -name 3 -print -quit 2>/dev/null
0/2/4/3
```

This works on any graph, directed or undirected, and will print a path if it
finds one. Since `find` will never continue running if it found a loop this is
successful even if no paths exist between the given vertices:

```
% find -L 0/ -name 6 -print -quit 2>/dev/null
<nothing>
```

Note that `find` operates by [depth first
search](https://en.wikipedia.org/wiki/Depth-first_search), and thus might not
find the shortest path between two vertices. The last thing we will show is how
to (ab)use the `mindepth` and `maxdepth` options in order to force find to do a
[breadth first search](https://en.wikipedia.org/wiki/Breadth-first_search),
which always finds the shortest path in unweighted graphs:

{{< highlight bash >}}
find_shortest() {
    local i=0
    local p=""
    while [[ -z $p ]]; do
        p=$(find -L $1 -mindepth $i -maxdepth $i -name $2 -print -quit 2>/dev/null)
        i=$(( i+1 ))
    done
    echo $p
}

# Output
% find_shortest 0 3
0/2/4/3
{{< /highlight >}}

While we have not found a path, we tell find to extend its search radius by one,
and once we found our desired path we output it. This approach is guaranteed to
always return the shortest path, although it is computationally more expensive
since we often traverse the same parts, and it also won't detect if no path
exists between these vertices. We could extend this program to add termination
detection, however then it would lose its elegance, which is what I wanted to
show in this post.

### Closing remarks

Interestingly enough, somebody [actually implemented
`bfs(1)`](https://github.com/tavianator/bfs), which behaves like `find` and
internally uses breadth-first search, but it is not (and probably will never) be
part of coreutils.

I did use this algorithm in one of my courses to solve small pathfinding
problems[^5]. Implementing this was quite fun
and not as much pain as I'd feared. Sometimes these `clever' solutions have their
merit - although for more sophicated problems I'd rather stick to C++ and BGL.

[^1]: The numbers in parenthesis correspond to the man page section of the programs. Most programs have their manpage in section 1, which is for user commands, If you are interested in more details or the other sections `man man` gives great insight.
[^2]: Also a part of the coreutils package
[^3]: Like `cat` but reverses the order of the lines
[^4]: Actually the case where we input the same number twice is tedious since our solution depends on the paths being different, as you will see later.
[^5]: and so far I haven't been kicked out
[^6]: short for Boost Graph Library

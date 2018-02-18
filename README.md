# Clojush EDN->CSV exercise

In our research work we use a genetic programming system called
[Clojush](https://github.com/lspector/Clojush) that has options to output
_large_ amounts of data (details about every individual in every generation
in the evolutionary system) in the
[extensible data notation (EDN) format](https://github.com/edn-format/edn).
EDN is essentially what you would get if you just printed the information
from Clojure, along with a nice tagging system that allows you to identify
things has being of complex, high-level types (e.g., a `User` or an
`Invoice`).

Since Clojush is written in Clojure, and a lot of our research
analysis tools are also written in Clojure, EDN was a natural choice as an
output format. A key analytical tool for us, however, has been
[the Neo4j graph database system](https://neo4j.com/). We need to be able to
import our big run dumps into Neo4j pretty efficiently. There is [a Neo4j
batch import tool](https://neo4j.com/blog/bulk-data-import-neo4j-3-0/) that
we can use to pull all the data into Neo4j pretty efficiently, but it requires
that the data be in the form of
[a collection of CSV files](https://neo4j.com/docs/operations-manual/current/tools/import/file-header-format/)
with different kinds of nodes and edges all being in separate files.

So we need a tool to convert one of our EDN output files into the necessary
set of CSV files for the Neo4j `import` program. This is a process somewhat
similar to the word count examples in _Seven concurrency models in seven
weeks_, so you should be able to use those examples to help guide your way on
this project.

In what follows I'll:

* Provide a brief description of the EDN files and how we'll deal with them
  in Clojure
* Describe the various CSV files we want to construct
* Discuss different ways we might parallelize the work

Your job, then, is to extend this starter code into a fully functional
EDN to CSV transformer that uses appropriate Clojure parallelization tools to
take advantage of multiple cores.

## EDN format

The EDN files have one "object" per line, where some of the lines are quite
long because the objects are large. All the lines _except the first_ (which is
a bit of an oddball which I'll say a little bit about shortly) has the tag
`#clojush/individual`, indicating that it represents a single individual from
the run. If a run has 11,000 individuals (for example, 1,000 individuals per
generation for 11 generations) then the EDN file will have 11,001 lines
(one for each individual, plus the oddball first line).

The first line has the tag `#clojush/run` and is different from all the other
lines. It simply captures all the parameters and settings for this run, and
we'll ignore that for now. In full application we'd want to create a CSV
containing all the data in that line, but that's more tedious than interesting,
and doesn't contribute anything interesting to the parallelization, so we'll
just ignore it.

Each `clojush.individual` is essentially a Clojure map containing numerous
key-value pairs describing that particular individual. This is not unlike
having a JSON map describing an object, but EDN uses a Clojure style syntax
instead of a JavaScript style syntax. Happily you don't have to understand the
details as long as you can extract the relevant elements and print them out
in the correct file. That said, here are the key components of an individual:

* `:uuid`, a [universally unique
  identifier (UUID)](https://en.wikipedia.org/wiki/Universally_unique_identifier)
  for this individual, which will turn out to be important when we deal
  with parent-child edges. Happily, EDN and Clojure work together to
  deal with these very nicely with no effort on our part; they have a
  `#uuid` tag in EDN, and Clojure automatically parses them into an internal
  UUID format when it reads the EDN file.
* `:generation`, which indicates which generation this individual was in
* `:location`, which indicates which position this individual had in collection
  of individuals in its generation
* `:parent-uuids`, a sequence of the UUIDs of the parent or parents of this
  individual. Individuals in generation 0 have no parents, so
  their `:parent-uuids` field has the value `nil`; starting with generation 1,
  though, that should be a list of one or two UUIDs.
* `:genetic-operators`, either a single keyword or a vector of keywords
  indicating what genetic operators were used to construct this individual
  using the genetic information in its parents.
* `:program`, the program (in a cool/strange language called Push) that this
  individual encodes. These programs are possibly nested lists of instructions,
  and can be _quite_ long, so the value associated with this key can go on for
  quite a ways.
* `:genome`, is a vector of genes, each of which is a small map of its own.
  The genome is the genetic representation encoded in individuals, and is the
  thing that is mutated and crossed-over to construct new individuals. We'll
  need to process each gene in the `:genome` separately in generating the CSV files. Each gene map has four fields that we care about here:
   * `:close`, a small integer that plays an important role in deciding when
     code blocks end in the Push program encoded by this genome.
   * `:instruction`, the Push instruction represented by this gene
   * `:uuid`, a UUID for this individual gene. This is different from the UUID
     of the individual containing this gene.
   * `:parent-uuid`, the UUID of the _gene_ (not individual) that was the
     "parent" gene of this gene.
* `:total-error`, the total error for this individual on the test suite for
  the target problem
* `:errors`, a vector of the individual error values on each of the test cases
  for the target problem. There are typically several hundred of these, and
  their sum will be the same as `:total-error`. We'll need to process each of
  these separately in generating out CSV files.

## CSV files and formats

The Neo4j `import` tool takes multiple CSV files. Some of these specify
different kinds of nodes, while others specify different kinds of edges.
Each different kind of node (e.g., an `Individual` vs. a `Gene`) has
different fields associated with it (e.g., `Individual`s have `generation`s,
while `Gene`s have `instruction`s), so each different kind of node has
its own CSV file (e.g., `Individuals.csv` and `Genes.csv`) since they'll
have different columns for their different fields. The same is true for
edges; different types of edges will each have their own CSV file.

Below is a description of several of the desired CSV files that we want to
generate for a given EDN file. There are some additional files that we would
actually want in production, but including them isn't particularly instructive
and ends up becoming more like "busy work" than useful in the context of a
class exercise.

### Node files

Every row in every node CSV file has to have a unique identifier in it's first
column. In some cases we already have that (usually a UUID from the EDN file),
but there are cases where we don't. If we don't, we'll need to construct one;
the simplest solution is to [generate a new UUID using
Clojure](https://gist.github.com/gorsuch/1418850) (or you can use
[the `clj-uuid` library](https://danlentz.github.io/clj-uuid/)).

#### Individuals.csv

This is the file created by the sample code, and has one line per individual
in the EDN file. It has four columns:

* `UUID` for the individual. For technical reasons having to do with the
way Neo4j's `import` tool works, the header for this column needs to
be `UUID:ID(Individual)`.
* `Generation:int`
* `Location:int`
* `:LABEL`, which is required by Neo4j `import` so that it knows what type
of node this row represents. Every row in this file should have the value
`Individual` in this column because every row represents an `Individual` node
in the resulting graph.

#### Semantics.csv

The `Semantics` nodes represent the semantics (or behavior) of individuals,
which in this situation is the combination of the `:total-error` and
the values in the `:errors` vector. One of the complications here is that
many individuals can have the same semantics, and we only want _one_ instance
of each semantics in this file. So we'll essentially have to maintain a
_set_ of semantics as we process the EDN file, and only output one line
to `Semantics.csv` for each item in that set.

This file has three columns:

* `UUID:ID(Sematics)`; you'll need to construct a UUID here
* `TotalError:int`
* `:LABEL`, which will always have the value `Semantics` for every row in this file

#### Errors.csv

For each (unique) semantics, we need to iterate through all the errors
in the `:errors` vector and create an `Error` node for each of these,
eventually (see below) connecting it to the `Semantics` node it was part
of.

This file will have four columns:

* `UUID:ID(Error)`; you'll need to construct a UUID here
* `ErrorValue:int`, which is the value for this entry in the `:errors` vector
* `Position:int`, which is the (zero-based) index of this error in the `:errors`
  vector
* `:LABEL`, which should be `Error` for every row in this file

### Edge files

#### ParentOf_edges.csv

This indicates edges that connect parent individuals to child individuals.
These are _directed_ edges that go from the parent to the child.
It has four columns:

* `:START_ID(Individual)`, which is the UUID of the _parent_ node
* `GeneticOperator`, which is the value (as a string) of the `:genetic-operator`
field in the _child_ individual
* `:END_ID(Individual)`, which is the UUID of the _child_ node
* `:TYPE`, which should be `PARENT_OF` for every row (edge) in this file

#### Individual_Semantics_edges.csv

These are edges that connect individuals to their semantics and should have
three columns:

* `:START_ID(Individual)`, which is the UUID of the individual
* `:END_ID(Semantics)`, which is the UUID of the semantics node. Note that
  since we only want one `Semantics` node for each unique semantics, we'll
  have to get the UUID of the one unique instances of this semantics in the
  set of semantics we build up as we go through the population.
* `:TYPE`, which should be `HAS_SEMANTICS` for every row in this file

#### Semantics_Error_edges.csv

These are edges that connect semantics to the error nodes representing the
values in their `:errors` vector. It should have three columns:

* `START_ID(Semantics)`, which is the UUID of the semantics
* `:END_ID(Error)`, which is the UUID of the `Error` node
* `:TYPE`, which should be `HAS_ERROR` for every row in this file

## Usage

You'll need to start by forking this repository, and then doing your work
in that fork.

The `-main` method in the starter code takes an EDN file name such as
`log1.edn` as an argument and generates the `Individuals` file with a
name such as `log1_Individuals.csv`. The `-main` method also allows you
to specify one of three strategies for processing the EDN file:

* `sequential`, which doesn't use any parallelism
* `pmap`, which uses a couple of applications of `pmap` to provide
  parallelism. (You might explore the impact of the two uses of `pmap`.
  Are they both important? Is one more effective on it's own than the other?)
* `reducers`, which uses reducers along with [the
  `iota` library](https://github.com/thebusby/iota) to provide the parallelism.

The code uses `sequential` by default if you don't provide a strategy
argument. So you can call this on command line using things like:

* `lein run ../data/log1.edn` (which uses `sequential` by default)
* `lein run ../data/log1.edn pmap`
* `lein run ../data/log1.edn reducers`

## License

Copyright © 2018 Nic McPhee

Distributed under the MIT License.

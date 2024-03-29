Copyright (c) 2020-2021 The University of Edinburgh.

This software was developed as part of the
EPSRC funded project ASiMoV (Project ID: EP/S005072/1)

Licensed under the Apache License, Version 2.0 (the "License"); you may not use this file except in
compliance with the License. You may obtain a copy of the License at

http://www.apache.org/licenses/LICENSE-2.0
Unless required by applicable law or agreed to in writing, software distributed under the License is
distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or
implied. See the License for the specific language governing permissions and limitations under the
License.

# PATtools

PATtools is a Python utility for extracting information generated by CrayPAT.

## Setup

After cloning, ensure that the directory containing pattools is on `PYTHONPATH`.
For example:

```
cd ~/src/python
git clone pattools
export PYTHONPATH=~/src/python:${PYTHONPATH}
```

you can confirm this by running `import pattools` at the python interpreter.

## Usage

Basic usage is to generate a profile using CrayPAT following a procedure similar to:

```
module load perftools-base
module load perftools

make clean
make [FLAGS]

pat_build -o prog+pat prog
```

Replace the name of the program in any job submission scripts with `prog+pat` and run, this will
produce either a file like `prog.xf` or a directory `prog.s/`.
These are then processed using pat_report to generate csv-formatted information as follows:

```
pat_report -P -s show_data="csv" -o prog.pat-csv prog.xf
```

replacing `prog.xf` with `prog.s/` as appropriate.
Unfortunately the csv-formatted data is embedded in a text report and thus not (easily) machine or
human readable, this is where pattools comes in!

The program `pat2csv.py` simply runs through the file generated by `pat_report` and extracts each
table into a separate `.csv` file:

```
python /path/to/pat2csv.py -i prog.pat-csv
```

this will generate multiple files of the form `TableX-YYYY.csv` that can then be processed in
whatever manner you like.

### Producing call graphs

`pat_report` is also capable of producing call graphs, `pat2dot.py` converts these into a `.dot`
file for later processing by the `dot` program from graphviz:

```
pat_report -P -O ct -s show_data="csv" -o prog.ct-csv prog.xf
python /path/to/pat2dot.py -i prog.ct-csv -o prog.dot
```

The dot file can then be used to create a graphical callgraph in multiple formats, for example:

```
dot -Tpng -o prog.png prog.dot
```

For very large call graphs it may be useful to show only the first `n` levels, `pat2dot.py` supports
this by optionally passing `-l n` for integer `n`, by default all levels are shown.

### Measuring communication ratios

After running `pat_report` the profile directory can be opened in Apprentice2 (`app2` on the command
line) to explore graphically.
One of the reports presented by Apprentice2 is the "communications mosaic" which displays various
MPI metrics between processors.
Using the export to csv functionality this can then be processed by the `patmat.py` tool to compute
the on-node/total ratio for the given metric, to do so it requires the generated .csv file and the
size of each node in terms of MPI ranks, e.g. for a 128-rank per node system:
```
python patmat.py -i mosaic-data.csv -n 128
```
this reports the per-node and min, max, mean and standard deviation of the metric ratios.

### Plotting communication ratios

Alternatively you can plot the communications mosaic, to do so, reuse the `patmat.py` program and
pass the additional flags `-m plot` to set the mode (default is `-m ratio`) and `-m image.fmt`:
```
python patmat.py -i mosaic-data.csv -n 128 -m plot -o mosaic.png
```
this will output a mosaic like the one displayed in Apprentice2 with a grid overlay showing the node
extents, any image format supported by matplotlib should also be supported by this tool.

#### Plotting communication ratio deltas

When comparing two partitioning options you can plot the difference between the two as follows:
```
python patmat.py -i mosaic-data.csv -n 128 -m delta -s mosaic2-data -o mosaic-delta.png
```

#### Plotting very large communication graphs

For very large communication graphs you can coarsen the plot to plot only the inter-node
communications metric by adding the `-c` flag.

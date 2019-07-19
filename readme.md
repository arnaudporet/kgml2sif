# Converting KGML-encoded KEGG pathways to the SIF file format

Copyright 2019 [Arnaud Poret](https://github.com/arnaudporet)

This work is licensed under the [Apache License Version 2.0](http://www.apache.org/licenses/LICENSE-2.0) (the "License"). You may not use this work except in compliance with the License. You may obtain a copy of the License at http://www.apache.org/licenses/LICENSE-2.0.

## kgml2sif

Converts KGML-encoded KEGG pathways to the SIF file format.

For information about the KGML file format, please see at the end of this readme file.

For information about the SIF file format, please see at the end of this readme file.

### Requirements

* [Python 3](https://www.python.org)
* a unix-based operating system

### Usage

`kgml2sif [-h] [-s FILE] [-c FILE] [-l] kgmlfile [kgmlfile ...]`

Ensure that `kgml2sif` is executable: `chmod ugo+x kgml2sif`.

Positional arguments:
    * `kgmlfile`: a KGML-encoded KEGG pathway

Optional arguments:
    * `-s/--symbol <file>`: a conversion table for translating KEGG gene IDs to gene symbols (see the `kegg2symbol.csv` file provided with kgml2sif in the `conv` folder)
    * `-c/--compound <file>`: a conversion table for translating compound IDs to compound names (see the `compound2name.csv` file provided with kgml2sif in the `conv` folder)
    * `-l/--license`: print the [Apache License Version 2.0](http://www.apache.org/licenses/LICENSE-2.0) under which kgml2sif is
    * `-h/--help`: print help

Currently:
    * kgml2sif is more suitable for processing [KEGG signaling pathways](https://www.genome.jp/kegg/pathway.html#environmental). However, extending it to better handle other types of KEGG pathways (such as [metabolic ones](https://www.genome.jp/kegg/pathway.html#metabolism)) is envisioned.
    * the following node types are considered:
        * gene
        * compound
        * group (in KGML, groups are complexes)
    * the following relation types are considered (in KGML, relations are edges):
        * PPrel (protein-protein)
        * GErel (gene expression)
        * PCrel (protein-compound)
        * ECrel (enzyme-compound)

In the output SIF file:
    * relation names (i.e. edge names) are suffixed with their corresponding type, ex:
        * `phosphorylation_PPrel`
        * `repression_GErel`
    * if necessary, edges having multiple types are split in order to obtain one type per edge (__warning:__ it can create multi-edges)
    * the non-KGML `membership_CPXrel` relation is added to indicate when a node is component of a complex (CPXrel stands for relations involving complexes, an added non-KGML relation type)
    * complexes are named as follows: `cp1::cp2::cp3` where `cp1`, `cp2` and `cp3` are the complex components. By the way, in this example of a
3-components complex, the following edges would be added as explain above:
        * `cp1    membership_CPXrel    cp1::cp2::cp3`
        * `cp2    membership_CPXrel    cp1::cp2::cp3`
        * `cp3    membership_CPXrel    cp1::cp2::cp3`

If `-s/--symbol` or `-c/--compound` are used:
    * the conversion tables must be 2-columns CSV files with semicolon as field separator (see the `conv` folder)
    * unmatched IDs are left unchanged in the output SIF file

### Cautions

* currently, the file `kegg2symbol.csv` provided with kgml2sif in the `conv` folder only works for KEGG __human__ gene IDs.

## Examples

These examples come from downloaded human KEGG pathways.

### ErbB signaling pathway:

```sh
./kgml2sif -s conv/kegg2symbol.csv -c conv/compound2name.csv examples/ErbB_signaling_pathway/ErbB_signaling_pathway.xml
```

### Insulin signaling pathway:

```sh
./kgml2sif -s conv/kegg2symbol.csv -c conv/compound2name.csv examples/Insulin_signaling_pathway/Insulin_signaling_pathway.xml
```

Note that kgml2sif raises warnings when processing `Insulin_signaling_pathway.xml` (see `Insulin_signaling_pathway-warnings.txt`):

```
PCrel: 61 --> 17: missing name, skipping
```

This warning indicates that a PCrel edge linking the node 61 to the node 17 (identified by their IDs in the KGML file) has no name (i.e. no information about the modeled biological interaction) and is therefore skipped by kgml2sif.

### p53 signaling pathway:

```sh
./kgml2sif -s conv/kegg2symbol.csv -c conv/compound2name.csv examples/p53_signaling_pathway/p53_signaling_pathway.xml
```

Note that kgml2sif indicates that there are multi-edges in `p53_signaling_pathway.xml` (see `p53_signaling_pathway-multi.sif`):

```
TP53    activation_PPrel    MDM2
TP53    expression_GErel    MDM2
```

When processing `p53_signaling_pathway.xml`, kgml2sif identifies two edges linking TP53 to MDM2. Theses two edges are therefore multi-edges. However, they are not considered invalid by kgml2sif since it can not determine if one of them is wrong, or which is the most relevant, or if they are both meaningful (kgml2sif is an algorithm, not a human expert able to perform such judgments).

Consequently, because these edges are not considered invalid by kgml2sif, they are left inside the output SIF file and the user is warned about the presence of such edges.

## The KGML file format

KGML stands for KEGG Markup Language. It is an [XML](https://www.w3.org/XML/) representation of KEGG pathways and is a file format in which KEGG pathways can be downloaded, either from [KEGG Pathway](https://www.genome.jp/kegg/pathway.html) or using the [KEGG API](https://www.kegg.jp/kegg/rest/keggapi.html).

For full explanation of the KGML file format, please see https://www.kegg.jp/kegg/xml/docs/.

## The SIF file format

In a SIF file encoding a network, each line encodes an edge as follows:

```
source \t interaction \t target
```

Note that the field separator is the tabulation `\t`: the SIF file format is the tab-separated values format (TSV) with exactly 3 columns.

For example, the edge representing the _activation_ of _RAF1_ by _HRAS_ is a line of a SIF file encoded as follows:

```
HRAS \t activation \t RAF1
```
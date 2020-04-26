# Converting KGML encoded KEGG human pathways to the SIF file format

Copyright (C) 2019-2020 [Arnaud Poret](https://github.com/arnaudporet)

This work is licensed under the [GNU General Public License](https://www.gnu.org/licenses/gpl.html).

To view a copy of this license, visit https://www.gnu.org/licenses/gpl.html.

## kgml2sif

Convert KGML encoded KEGG human pathways to the SIF file format.

For explanations about the KGML and SIF file formats, see at the end of this readme file.

Currently:

* the following node types are considered:
    * `gene` (in KGML, genes also stand for the corresponding proteins/gene products)
    * `compound`
    * `group` (in KGML, groups stand for complexes)
* the following relation types are considered (in KGML, relations stand for edges):
    * `PPrel`: protein-protein, protein-complex or complex-complex links
    * `GErel`: protein-gene or complex-gene links (i.e. gene expression regulations)
    * `PCrel`: protein-compound or complex-compound links
    * `ECrel`: enzyme-enzyme links sharing a common compound (as product for the first enzyme and as substrate for the second one)

In the output SIF file:

* edge names (i.e. relation names) are suffixed with their corresponding type, ex:
    * `phosphorylation_PPrel`
    * `repression_GErel`
* edges for which the name is missing are automatically named `unknown`, suffixed with their corresponding type, ex:
    * `unknown_PCrel`
    * `unknown_PPrel`
* edges are automatically checked regarding their type and the nodes they link:
    * `PPrel`: `(source=protein OR source=complex) AND (target=protein OR target=complex)`
    * `GErel`: `(source=protein OR source=complex) AND target=gene`
    * `PCrel`: `((source=protein OR source=complex) AND target=compound) OR (source=compound AND (target=protein OR target=complex))`
    * `ECrel`: `(source=protein OR source=complex) AND (target=protein OR target=complex)`
* if necessary, edges having multiple types are split in order to obtain one type per edge (__warning: it can create multiedges__)
* multiedges, if any, are left inside the output SIF file and kgml2sif warns about it
* the non KGML `membership_CPXrel` edges are added in order to indicate when nodes are component of complexes (`CPXrel` is an added non KGML relation type)
* complexes are named as follows:
    * `cp1&cp2&cp3`
    * where `cp1`, `cp2` and `cp3` are the complex components
* by the way, in this example of a 3 components complex, the following edges would be added as explain above:
    * `cp1    membership_CPXrel    cp1&cp2&cp3`
    * `cp2    membership_CPXrel    cp1&cp2&cp3`
    * `cp3    membership_CPXrel    cp1&cp2&cp3`

Node name prefixes in KGML:

* specify the node type
* the name of nodes representing human genes is prefixed with `hsa:`
* the name of nodes representing compounds is prefixed with `cpd:`
* the name of nodes representing glycans (a subtype of compounds) is prefixed with `gl:`
* the name of nodes representing drugs (a subtype of compounds) is prefixed with `dr:`

### Requirements

* [Python 3](https://www.python.org)
* a unix based operating system

### Usage

Ensure that `kgml2sif` is executable:

```sh
chmod ugo+rx kgml2sif
```

Usage: `kgml2sif [-h] [-g <tsvFile>] [-c <tsvFile>] <kgmlFile> [<kgmlFile> ...]`

Positional arguments:

* `<kgmlFile>`: a KGML encoded KEGG human pathway

Optional arguments:

* `-h`, `--help`: print help
* `-g <tsvFile>`, `--geneSymbol <tsvFile>`: a conversion table for translating KEGG human gene IDs to gene symbols (see the file `gene2symbol.tsv` provided with kgml2sif in the `conv` folder)
* `-c <tsvFile>`, `--compoundName <tsvFile>`: a conversion table for translating KEGG compound IDs to compound names (see the file `compound2name.tsv` provided with kgml2sif in the `conv` folder)

If `-g/--geneSymbol` or `-c/--compoundName` is used:

* kgml2sif attempts to name gene nodes (`-g/--geneSymbol`) or compound nodes (`-c/--compoundName`) according to a provided conversion table "KEGG ID to name"
* the conversion table must be a 2 columns TSV file
* unmatched KEGG IDs are left unchanged in the output SIF file and kgml2sif warns about it

## Examples

These examples come from downloaded human [KEGG pathways](https://www.genome.jp/kegg/pathway.html).

The SIF files produced by kgml2sif from these examples are also in the SVG file format for visualization purpose.

### Apoptosis

```sh
./kgml2sif -g conv/gene2symbol.tsv -c conv/compound2name.tsv examples/Apoptosis/Apoptosis.xml
```

### ErbB signaling pathway

```sh
./kgml2sif -g conv/gene2symbol.tsv -c conv/compound2name.tsv examples/ErbB_signaling_pathway/ErbB_signaling_pathway.xml
```

### Insulin signaling pathway

```sh
./kgml2sif -g conv/gene2symbol.tsv -c conv/compound2name.tsv examples/Insulin_signaling_pathway/Insulin_signaling_pathway.xml
```

Note that, when processing the KGML file `Insulin_signaling_pathway.xml`, kgml2sif has encountered an invalid edge (see the warn file `Insulin_signaling_pathway.warns.txt`):

```
PCrel: (68,65): (gene,gene): invalid edge, skipping
```

This edge links the node 68 to the node 65 (node IDs inside `Insulin_signaling_pathway.xml`) which are both of type `gene` (which also stands for the associated protein/gene product).

In addition, this edge is of type `PCrel`, meaning that it is intended to link genes/proteins to compounds (or vice versa).

However, neither the node 68 nor the node 65 is of type `compound`: they are both of type `gene`.

Consequently, kgml2sif has skipped this edge and has logged the corresponding warns in the warn file `Insulin_signaling_pathway.warns.txt`.

### p53 signaling pathway

```sh
./kgml2sif -g conv/gene2symbol.tsv -c conv/compound2name.tsv examples/p53_signaling_pathway/p53_signaling_pathway.xml
```

Note that, when processing the KGML file `p53_signaling_pathway.xml`, kgml2sif has encountered multiedges (see the SIF file `p53_signaling_pathway.multiedges.sif`):

```
hsa:TP53    activation_PPrel    hsa:MDM2
hsa:TP53    expression_GErel    hsa:MDM2
```

These two edges link TP53 to MDM2: they are therefore multiedges.

However, they are not considered invalid by kgml2sif because it can not determine if one of them is wrong, or which is the most relevant, or if they are both meaningful: kgml2sif is not a human able to perform such judgments.

Consequently, these edges are left inside the output SIF file `p53_signaling_pathway.sif` and kgml2sif warns about the presence of such edges.

### TNF signaling pathway

```sh
./kgml2sif -g conv/gene2symbol.tsv -c conv/compound2name.tsv examples/TNF_signaling_pathway/TNF_signaling_pathway.xml
```

## The KGML file format

KGML stands for KEGG Markup Language. It is an [XML](https://www.w3.org/XML/) representation of [KEGG pathways](https://www.genome.jp/kegg/pathway.html) and is a file format in which KEGG pathways can be downloaded, either from the KEGG web site or using the [KEGG API](https://www.kegg.jp/kegg/rest/keggapi.html).

For a complete explanation of the KGML file format, see https://www.kegg.jp/kegg/xml/docs/.

## The SIF file format

In a SIF file encoding a network, each line encodes an edge as follows:

```
source \t interaction \t target
```

Note that the field separator is the tabulation: the SIF file format is the tab separated values format (TSV) with exactly 3 columns.

For example, the edge representing the activation of RAF1 by HRAS is a line of a SIF file encoded as follows:

```
HRAS \t activation \t RAF1
```

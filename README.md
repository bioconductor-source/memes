
<!-- README.md is generated from README.Rmd. Please edit that file -->

# memes

<!-- badges: start -->

[![Lifecycle:
experimental](https://img.shields.io/badge/lifecycle-experimental-orange.svg)](https://www.tidyverse.org/lifecycle/#experimental)
[![Project Status: Active – The project has reached a stable, usable
state and is being actively
developed.](https://www.repostatus.org/badges/latest/active.svg)](https://www.repostatus.org/#active)
[![Codecov test
coverage](https://codecov.io/gh/snystrom/dremeR/branch/master/graph/badge.svg)](https://codecov.io/gh/snystrom/dremeR?branch=master)
[![R build
status](https://github.com/snystrom/dremeR/workflows/R-CMD-check/badge.svg)](https://github.com/snystrom/dremeR/actions)
<!-- badges: end -->

An R interface to the [MEME Suite](http://meme-suite.org/) family of
tools, which provides several utilities for performing motif analysis on
DNA, RNA, and protein sequences. memes works by detecting a local
install of the MEME suite, running the commands, then importing the
results directly into R.

## Installation

### Development Version

You can install the development version of memes from
[GitHub](https://github.com/snystrom/memes) with:

``` r
if (!requireNamespace("remotes", quietly=TRUE))
  install.packages("remotes")
# memes currently requires the development version of universalmotif:
remotes::install_github("bjmt/universalmotif")
remotes::install_github("snystrom/memes")
```

### Docker Container

``` shell
# Get development version from dockerhub
docker pull snystrom/memes_docker:devel
# the -v flag is used to mount an analysis directory, 
# it can be excluded for demo purposes
docker run -e PASSWORD=<password> -p 8787:8787 -v <path>/<to>/<project>:/tmp/<project> snystrom/memes_docker:devel
```

## Detecting the MEME Suite

memes relies on a local install of the [MEME
Suite](http://meme-suite.org/). For installation instructions for the
MEME suite, see the [MEME Suite Installation
Guide](http://meme-suite.org/doc/install.html?man_type=web).

memes needs to know the location of the `meme/bin/` directory on your
local machine. You can tell memes the location of your MEME suite
install in 4 ways. memes will always prefer the more specific definition
if it is a valid path. Here they are ranked from most- to
least-specific:

1.  Manually passing the install path to the `meme_path` argument of all
    memes functions
2.  Setting the path using `options(meme_bin = "/path/to/meme/bin/")`
    inside your R script
3.  Setting `MEME_BIN=/path/to/meme/bin/` in your `.Renviron` file
4.  memes will try the default MEME install location `~/meme/bin/`

If memes fails to detect your install at the specified location, it will
fall back to the next option.

To verify memes can detect your MEME install, use `check_meme_install()`
which uses the search herirarchy above to find a valid MEME install. It
will report whether any tools are missing, and print the path to MEME
that it sees. This can be useful for troubleshooting issues with your
install.

``` r
library(memes)

# Verify that memes detects your meme install
# (returns all green checks if so)
# (I have MEME installed to the default location)
check_meme_install()
#> checking main install
#> ✓ /opt/meme/bin
#> checking util installs
#> ✓ /opt/meme/bin/dreme
#> ✓ /opt/meme/bin/ame
#> ✓ /opt/meme/bin/fimo
#> ✓ /opt/meme/bin/tomtom
#> ✓ /opt/meme/bin/meme
```

``` r
# You can manually input a path to meme_path
# If no meme/bin is detected, will return a red X
check_meme_install(meme_path = 'bad/path')
#> checking main install
#> x bad/path
```

## The Core Tools

| Function Name |              Use               | Sequence Input | Motif Input | Output                                                  |
| :-----------: | :----------------------------: | :------------: | :---------: | :------------------------------------------------------ |
| `runDreme()`  | Motif Discovery (short motifs) |      Yes       |     No      | data.frame w/ `motifs` column                           |
|  `runAme()`   |        Motif Enrichment        |      Yes       |     Yes     | data.frame (optional: `sequences` column)               |
|  `runFimo()`  |         Motif Scanning         |      Yes       |     Yes     | GRanges of motif positions                              |
| `runTomTom()` |        Motif Comparison        |       No       |     Yes     | data.frame w/ `best_match_motif` and `tomtom` columns\* |
|  `runMeme()`  | Motif Discovery (long motifs)  |      Yes       |     No      | data.frame w/ `motifs` column                           |

\* **Note:** if `runTomTom()` is run using the results of `runDreme()`,
the results will be joined with the `runDreme()` results as extra
columns. This allows easy comparison of *de-novo* discovered motifs with
their matches.

**Sequence Inputs** can be any of:

1.  Path to a .fasta formatted file
2.  `Biostrings::XStringSet` (can be generated from GRanges using
    `get_sequence()` helper function)
3.  A named list of `Biostrings::XStringSet` objects (generated by
    `get_sequence()`)

**Motif Inputs** can be any of:

1.  A path to a .meme formatted file of motifs to scan against
2.  A `universalmotif` object, or list of `universalmotif` objects
3.  A `runDreme()` results object (this allows the results of
    `runDreme()` to pass directly to `runTomTom()`)
4.  A combination of all of the above passed as a `list()`
    (e.g. `list("path/to/database.meme", "dreme_results" = dreme_res)`)

**Output Types**:

`runDreme()`, and `runTomTom()` return data.frames with special columns.
The `motif` column contains `universalmotif` objects, with 1 entry per
row. The remaining columns describe the properties of each returned
motif. The following column names are special in that their values are
used when running `update_motifs` to alter the properties of the motifs
stored in the `motif` column. Be careful about changing these values.

  - name
  - altname
  - family
  - organism
  - strand
  - nsites
  - bkgsites
  - pval
  - qval
  - eval

memes is built around the [universalmotif
package](https://www.bioconductor.org/packages/release/bioc/html/universalmotif.html)
which provides a framework for manipulating motifs in R. The `motif`
columns from `runDreme()` and `runTomTom()` can be used natively with
all `universalmotif` functions (eg
`universalmotif::view_motifs(dreme_results$motif)`).

`runTomTom()` returns a special column: `tomtom` which is a `data.frame`
of all match data for each input motif. This can be expanded out using
`tidyr::unnest(tomtom_results, "tomtom")`.

## Quick Examples

### Motif Discovery with DREME

``` r
suppressPackageStartupMessages(library(magrittr))
suppressPackageStartupMessages(library(GenomicRanges))

# Example transcription factor peaks as GRanges
data("example_peaks", package = "memes")

# Genome object
dm.genome <- BSgenome.Dmelanogaster.UCSC.dm6::BSgenome.Dmelanogaster.UCSC.dm6
```

The `get_sequence` function takes a `GRanges` or `GRangesList` as input
and returns the sequences as a `BioStrings::XStringSet`, or list of
`XStringSet` objects, respectively. `get_sequence` will name each fasta
entry by the genomic coordinates each sequence is from.

``` r
# Generate sequences from 200bp about the center of my peaks of interest
sequences <- example_peaks %>% 
  resize(200, "center") %>% 
  get_sequence(dm.genome)
```

`runDreme()` accepts XStringSet or a path to a fasta file as input. You
can use other sequences or shuffled input sequences as the control
dataset.

``` r
# runDreme accepts all arguments that the commandline version of dreme accepts
# here I set e = 50 to detect motifs in the limited example peak list
dreme_results <- runDreme(sequences, control = "shuffle", e = 50)
```

memes is built around the
[universalmotif](https://www.bioconductor.org/packages/release/bioc/html/universalmotif.html)
package. The `motif` column of the runDreme results object contains
`universalmotif` objects which can be used natively in all
`universalmotif` functions.

``` r
library(universalmotif)
view_motifs(dreme_results$motif)
```

![](man/figures/README-unnamed-chunk-7-1.png)<!-- -->

### Matching motifs using TOMTOM

Discovered motifs can be matched to known TF motifs using `runTomTom()`,
which can accept as input a path to a .meme formatted file, a
`universalmotif` list, or the results of `runDreme()`.

TomTom uses a database of known motifs which can be passed to the
`database` parameter as a path to a .meme format file, or a
`universalmotif` object.

Optionally, you can set the environment variable `MEME_DB` in
`.Renviron`, or the `meme_db` value in `options` to a valid .meme format
file and memes will use that file as the database. memes will always
prefer user input to the function call.

``` r
options(meme_db = system.file("extdata/db/fly_factor_survey_id.meme", package = "memes"))
m <- create_motif("CMATTACN", altname = "testMotif")
tomtom_results <- runTomTom(m)
```

``` r
tomtom_results
#>    name   altname family organism consensus alphabet strand icscore nsites
#> 1 motif testMotif   <NA>     <NA>  CMATTACN      DNA     +-      13     NA
#>   bkgsites pval qval eval best_match_name best_match_altname
#> 1       NA   NA   NA   NA      prd_FlyReg      FBgn0003145_2
#>                                                       best_match_motif
#> 1 <S4 class 'universalmotif' [package "universalmotif"] with 20 slots>
#>           best_db_name best_match_offset best_match_pvalue best_match_evalue
#> 1 fly_factor_survey_id                 0          7.38e-06           0.00449
#>   best_match_qvalue best_match_strand
#> 1            0.0079                 +
#>                                                                  motif
#> 1 <S4 class 'universalmotif' [package "universalmotif"] with 20 slots>
#>                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                tomtom
#> 1 prd_FlyReg, Ubx_FlyReg, Dr_SOLEXA, ftz_FlyReg, Hmx_SOLEXA, Dll_SOLEXA, BH1_SOLEXA, Exex_SOLEXA, Tup_Cell, Tup_SOLEXA, ovo_FlyReg, NK7.1_SOLEXA, Hmx_Cell, en_FlyReg, Bsh_Cell, tup_SOLEXA_10, CG34031_Cell, Exex_Cell, Dr_Cell, Unc4_Cell, CG11085_Cell, CG13424_SOLEXA, Abd-A_FlyReg, NK7.1_Cell, Dll_Cell, Dfd_FlyReg, CG13424_Cell, CG32532_Cell, Odsh_Cell, pho_FlyReg, CG34031_SOLEXA, Ftz_SOLEXA, Scr_SOLEXA, C15_Cell, CG15696_Cell, CG15696_SOLEXA, BH2_Cell, Bsh_SOLEXA, En_SOLEXA, Slou_SOLEXA, Antp_SOLEXA, Slou_Cell, ap_FlyReg, C15_SOLEXA, AbdA_SOLEXA, Zen2_SOLEXA, E5_SOLEXA, Awh_SOLEXA, Ems_SOLEXA, Lab_SOLEXA, Repo_SOLEXA, Unpg_SOLEXA, Ubx_Cell, Unpg_Cell, Ap_SOLEXA, Btn_SOLEXA, Pb_SOLEXA, Dfd_SOLEXA, Odsh_SOLEXA, Unc4_SOLEXA, FBgn0003145_2, FBgn0003944_2, FBgn0000492_2, FBgn0001077_2, FBgn0085448_2, FBgn0000157_2, FBgn0011758_2, FBgn0041156_2, FBgn0003896, FBgn0003896_2, FBgn0003028, FBgn0024321_2, FBgn0085448, FBgn0000577_2, FBgn0000529, FBgn0003896_3, FBgn0054031, FBgn0041156, FBgn0000492, FBgn0024184, FBgn0030408, FBgn0034520_2, FBgn0000014_2, FBgn0024321, FBgn0000157, FBgn0000439_2, FBgn0034520, FBgn0052532, FBgn0026058, FBgn0002521, FBgn0054031_2, FBgn0001077_3, FBgn0003339_2, FBgn0004863, FBgn0038833, FBgn0038833_2, FBgn0004854, FBgn0000529_2, FBgn0000577_3, FBgn0002941_2, FBgn0000095_3, FBgn0002941, FBgn0000099_2, FBgn0004863_2, FBgn0000014_3, FBgn0004054_2, FBgn0008646_2, FBgn0013751_2, FBgn0000576_3, FBgn0002522_2, FBgn0011701_2, FBgn0015561_2, FBgn0003944, FBgn0015561, FBgn0000099_3, FBgn0014949_2, FBgn0051481_2, FBgn0000439_3, FBgn0026058_2, FBgn0024184_2, <S4 class 'universalmotif' [package "universalmotif"] with 20 slots>, <S4 class 'universalmotif' [package "universalmotif"] with 20 slots>, <S4 class 'universalmotif' [package "universalmotif"] with 20 slots>, <S4 class 'universalmotif' [package "universalmotif"] with 20 slots>, <S4 class 'universalmotif' [package "universalmotif"] with 20 slots>, <S4 class 'universalmotif' [package "universalmotif"] with 20 slots>, <S4 class 'universalmotif' [package "universalmotif"] with 20 slots>, <S4 class 'universalmotif' [package "universalmotif"] with 20 slots>, <S4 class 'universalmotif' [package "universalmotif"] with 20 slots>, <S4 class 'universalmotif' [package "universalmotif"] with 20 slots>, <S4 class 'universalmotif' [package "universalmotif"] with 20 slots>, <S4 class 'universalmotif' [package "universalmotif"] with 20 slots>, <S4 class 'universalmotif' [package "universalmotif"] with 20 slots>, <S4 class 'universalmotif' [package "universalmotif"] with 20 slots>, <S4 class 'universalmotif' [package "universalmotif"] with 20 slots>, <S4 class 'universalmotif' [package "universalmotif"] with 20 slots>, <S4 class 'universalmotif' [package "universalmotif"] with 20 slots>, <S4 class 'universalmotif' [package "universalmotif"] with 20 slots>, <S4 class 'universalmotif' [package "universalmotif"] with 20 slots>, <S4 class 'universalmotif' [package "universalmotif"] with 20 slots>, <S4 class 'universalmotif' [package "universalmotif"] with 20 slots>, <S4 class 'universalmotif' [package "universalmotif"] with 20 slots>, <S4 class 'universalmotif' [package "universalmotif"] with 20 slots>, <S4 class 'universalmotif' [package "universalmotif"] with 20 slots>, <S4 class 'universalmotif' [package "universalmotif"] with 20 slots>, <S4 class 'universalmotif' [package "universalmotif"] with 20 slots>, <S4 class 'universalmotif' [package "universalmotif"] with 20 slots>, <S4 class 'universalmotif' [package "universalmotif"] with 20 slots>, <S4 class 'universalmotif' [package "universalmotif"] with 20 slots>, <S4 class 'universalmotif' [package "universalmotif"] with 20 slots>, <S4 class 'universalmotif' [package "universalmotif"] with 20 slots>, <S4 class 'universalmotif' [package "universalmotif"] with 20 slots>, <S4 class 'universalmotif' [package "universalmotif"] with 20 slots>, <S4 class 'universalmotif' [package "universalmotif"] with 20 slots>, <S4 class 'universalmotif' [package "universalmotif"] with 20 slots>, <S4 class 'universalmotif' [package "universalmotif"] with 20 slots>, <S4 class 'universalmotif' [package "universalmotif"] with 20 slots>, <S4 class 'universalmotif' [package "universalmotif"] with 20 slots>, <S4 class 'universalmotif' [package "universalmotif"] with 20 slots>, <S4 class 'universalmotif' [package "universalmotif"] with 20 slots>, <S4 class 'universalmotif' [package "universalmotif"] with 20 slots>, <S4 class 'universalmotif' [package "universalmotif"] with 20 slots>, <S4 class 'universalmotif' [package "universalmotif"] with 20 slots>, <S4 class 'universalmotif' [package "universalmotif"] with 20 slots>, <S4 class 'universalmotif' [package "universalmotif"] with 20 slots>, <S4 class 'universalmotif' [package "universalmotif"] with 20 slots>, <S4 class 'universalmotif' [package "universalmotif"] with 20 slots>, <S4 class 'universalmotif' [package "universalmotif"] with 20 slots>, <S4 class 'universalmotif' [package "universalmotif"] with 20 slots>, <S4 class 'universalmotif' [package "universalmotif"] with 20 slots>, <S4 class 'universalmotif' [package "universalmotif"] with 20 slots>, <S4 class 'universalmotif' [package "universalmotif"] with 20 slots>, <S4 class 'universalmotif' [package "universalmotif"] with 20 slots>, <S4 class 'universalmotif' [package "universalmotif"] with 20 slots>, <S4 class 'universalmotif' [package "universalmotif"] with 20 slots>, <S4 class 'universalmotif' [package "universalmotif"] with 20 slots>, <S4 class 'universalmotif' [package "universalmotif"] with 20 slots>, <S4 class 'universalmotif' [package "universalmotif"] with 20 slots>, <S4 class 'universalmotif' [package "universalmotif"] with 20 slots>, <S4 class 'universalmotif' [package "universalmotif"] with 20 slots>, fly_factor_survey_id, fly_factor_survey_id, fly_factor_survey_id, fly_factor_survey_id, fly_factor_survey_id, fly_factor_survey_id, fly_factor_survey_id, fly_factor_survey_id, fly_factor_survey_id, fly_factor_survey_id, fly_factor_survey_id, fly_factor_survey_id, fly_factor_survey_id, fly_factor_survey_id, fly_factor_survey_id, fly_factor_survey_id, fly_factor_survey_id, fly_factor_survey_id, fly_factor_survey_id, fly_factor_survey_id, fly_factor_survey_id, fly_factor_survey_id, fly_factor_survey_id, fly_factor_survey_id, fly_factor_survey_id, fly_factor_survey_id, fly_factor_survey_id, fly_factor_survey_id, fly_factor_survey_id, fly_factor_survey_id, fly_factor_survey_id, fly_factor_survey_id, fly_factor_survey_id, fly_factor_survey_id, fly_factor_survey_id, fly_factor_survey_id, fly_factor_survey_id, fly_factor_survey_id, fly_factor_survey_id, fly_factor_survey_id, fly_factor_survey_id, fly_factor_survey_id, fly_factor_survey_id, fly_factor_survey_id, fly_factor_survey_id, fly_factor_survey_id, fly_factor_survey_id, fly_factor_survey_id, fly_factor_survey_id, fly_factor_survey_id, fly_factor_survey_id, fly_factor_survey_id, fly_factor_survey_id, fly_factor_survey_id, fly_factor_survey_id, fly_factor_survey_id, fly_factor_survey_id, fly_factor_survey_id, fly_factor_survey_id, fly_factor_survey_id, 0, 0, 0, 2, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1, 1, 0, 0, 3, 0, 1, 0, 1, 0, 0, 1, 0, 0, 0, 4, 0, 0, 0, 0, 0, 0, 1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 7.38e-06, 4.24e-05, 0.00111, 0.00116, 0.00118, 0.00123, 0.00137, 0.00137, 0.00166, 0.00174, 0.00201, 0.00287, 0.00352, 0.00357, 0.00403, 0.00403, 0.00498, 0.00498, 0.00513, 0.0055, 0.00574, 0.00579, 0.00662, 0.00662, 0.00711, 0.00721, 0.00782, 0.00831, 0.00831, 0.00841, 0.00845, 0.00895, 0.00895, 0.0091, 0.0091, 0.00977, 0.0102, 0.0103, 0.0103, 0.0103, 0.0105, 0.0105, 0.0106, 0.0112, 0.0119, 0.0126, 0.0134, 0.0142, 0.0142, 0.0142, 0.0142, 0.0142, 0.0147, 0.0147, 0.0151, 0.0151, 0.0151, 0.0157, 0.0157, 0.0157, 0.00449, 0.0258, 0.673, 0.705, 0.719, 0.751, 0.83, 0.83, 1.01, 1.06, 1.22, 1.74, 2.14, 2.17, 2.45, 2.45, 3.03, 3.03, 3.12, 3.35, 3.49, 3.52, 4.02, 4.02, 4.32, 4.38, 4.75, 5.05, 5.05, 5.12, 5.14, 5.44, 5.44, 5.53, 5.53, 5.94, 6.19, 6.27, 6.27, 6.27, 6.37, 6.37, 6.46, 6.81, 7.26, 7.64, 8.13, 8.65, 8.65, 8.65, 8.65, 8.65, 8.96, 8.96, 9.21, 9.21, 9.21, 9.57, 9.57, 9.57, 0.0079, 0.0227, 0.183, 0.183, 0.183, 0.183, 0.183, 0.183, 0.186, 0.186, 0.196, 0.256, 0.258, 0.258, 0.258, 0.258, 0.258, 0.258, 0.258, 0.258, 0.258, 0.258, 0.258, 0.258, 0.258, 0.258, 0.258, 0.258, 0.258, 0.258, 0.258, 0.258, 0.258, 0.258, 0.258, 0.258, 0.258, 0.258, 0.258, 0.258, 0.258, 0.258, 0.258, 0.266, 0.273, 0.273, 0.273, 0.273, 0.273, 0.273, 0.273, 0.273, 0.273, 0.273, 0.273, 0.273, 0.273, 0.273, 0.273, 0.273, +, +, -, +, -, -, -, -, -, -, +, -, -, -, -, -, -, -, +, -, -, -, +, -, +, -, -, -, -, -, -, -, -, -, -, -, -, -, -, -, -, -, +, -, -, -, -, -, -, -, -, -, -, -, -, -, -, -, -, -
```

### Using runDreme results as TOMTOM input

`runTomTom()` will add its results as columns to a `runDreme()` results
data.frame.

``` r
full_results <- dreme_results %>% 
  runTomTom()
```

### Motif Enrichment using AME

AME is used to test for enrichment of known motifs in target sequences.
`runAme()` will use the `MEME_DB` entry in `.Renviron` or
`options(meme_db = "path/to/database.meme")` as the motif database.
Alternately, it will accept all valid inputs as `runTomTom()`.

``` r
ame_results <- runAme(sequences, control = "shuffle", evalue_report_threshold = 30)
ame_results
#> # A tibble: 2 x 17
#>    rank motif_db motif_id motif_alt_id consensus  pvalue adj.pvalue evalue tests
#>   <int> <chr>    <chr>    <chr>        <chr>       <dbl>      <dbl>  <dbl> <int>
#> 1     1 /usr/lo… Eip93F_… FBgn0013948  ACWSCCRA… 5.14e-4     0.0339   20.6    67
#> 2     2 /usr/lo… Cf2-PB_… FBgn0000286… CSSHNKDT… 1.57e-3     0.04     24.3    26
#> # … with 8 more variables: fasta_max <dbl>, pos <int>, neg <int>,
#> #   pwm_min <dbl>, tp <int>, tp_percent <dbl>, fp <int>, fp_percent <dbl>
```

## Visualizing Results

`view_tomtom_hits` allows comparing the input motifs to the top hits
from TomTom. Manual inspection of these matches is important, as
sometimes the top match is not always the correct assignment. Altering
`top_n` allows you to show additional matches in descending order of
their rank.

``` r
full_results %>% 
  view_tomtom_hits(top_n = 1) %>% 
  # Here I use the cowplot library simply to combine each plot into a 4-panel grid
  cowplot::plot_grid(plotlist = .)
```

![](man/figures/README-unnamed-chunk-12-1.png)<!-- -->

It can be useful to view the results from `runAme()` as a heatmap.
`ame_plot_heatmap()` can create complex visualizations for analysis of
enrichment between different region types (see vignettes for details).
Here is a simple example heatmap.

``` r
ame_results %>% 
  ame_plot_heatmap()
```

![](man/figures/README-unnamed-chunk-13-1.png)<!-- -->

## Importing Data from previous runs

memes also supports importing results generated using the MEME suite
outside of R (for example, running jobs on
[meme-suite.org](meme-suite.org), or running on the commandline).

| MEME Tool |    Function Name    | File Type  |
| :-------: | :-----------------: | :--------: |
|   Dreme   | `importDremeXML()`  | dreme.xml  |
|  TomTom   | `importTomTomXML()` | tomtom.xml |
|    AME    |    `importAme()`    | ame.tsv\*  |
|   FIMO    |   `importFimo()`    |  fimo.tsv  |
|   Meme    |   `importMeme()`    |  meme.txt  |

  - `importAME()` can also use the “sequences.tsv” output when AME used
    `method = "fisher"`, this is optional.

# FAQs

### How do I use memes/MEME on Windows?

The MEME Suite does not currently support Windows, although it can be
installed under [Cygwin](https://www.cygwin.com/) or the [Windows Linux
Subsytem](https://docs.microsoft.com/en-us/windows/wsl/install-win10)
(WSL). Please note that if MEME is installed on Cygwin or WSL, you must
also run R inside Cygwin or WSL to use memes.

# Citation

memes is a wrapper for a select few tools from the MEME Suite, which
were developed by another group. In addition to citing memes, please
cite the MEME Suite tools corresponding to the tools you use.

If you use `runDreme()` in your analysis, please cite:

Timothy L. Bailey, “DREME: Motif discovery in transcription factor
ChIP-seq data”, Bioinformatics, 27(12):1653-1659, 2011. [full
text](https://academic.oup.com/bioinformatics/article/27/12/1653/257754)

If you use `runTomTom()` in your analysis, please cite:

Shobhit Gupta, JA Stamatoyannopolous, Timothy Bailey and William
Stafford Noble, “Quantifying similarity between motifs”, Genome Biology,
8(2):R24, 2007. [full text](http://genomebiology.com/2007/8/2/R24)

If you use `runAme()` in your analysis, please cite:

Robert McLeay and Timothy L. Bailey, “Motif Enrichment Analysis: A
unified framework and method evaluation”, BMC Bioinformatics, 11:165,
2010, <doi:10.1186/1471-2105-11-165>. [full
text](http://www.biomedcentral.com/1471-2105/11/165)

If you use `runFimo()` in your analysis, please cite:

Charles E. Grant, Timothy L. Bailey, and William Stafford Noble, “FIMO:
Scanning for occurrences of a given motif”, Bioinformatics,
27(7):1017-1018, 2011. [full
text](http://bioinformatics.oxfordjournals.org/content/early/2011/02/16/bioinformatics.btr064.full)

## Licensing Restrictions

The MEME Suite is free for non-profit use, but for-profit users should
purchase a license. See the [MEME Suite Copyright
Page](http://meme-suite.org/doc/copyright.html) for details.

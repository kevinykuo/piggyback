
<!-- README.md is generated from README.Rmd. Please edit that file -->

[![lifecycle](https://img.shields.io/badge/lifecycle-experimental-orange.svg)](https://www.tidyverse.org/lifecycle/#experimental)
[![Travis-CI Build
Status](https://travis-ci.org/cboettig/piggyback.svg?branch=master)](https://travis-ci.org/cboettig/piggyback)
[![Coverage
status](https://codecov.io/gh/cboettig/piggyback/branch/master/graph/badge.svg)](https://codecov.io/github/cboettig/piggyback?branch=master)
[![AppVeyor build
status](https://ci.appveyor.com/api/projects/status/github/cboettig/piggyback?branch=master&svg=true)](https://ci.appveyor.com/project/cboettig/piggyback)

# piggyback

`piggyback` is basically a poor soul’s [git
LFS](https://git-lfs.github.com/). GitHub rejects commits containing
files larger than 50 Mb. `git LFS` is not only expensive, it also
[breaks GitHub’s collaborative
model](https://medium.com/@megastep/github-s-large-file-storage-is-no-panacea-for-open-source-quite-the-opposite-12c0e16a9a91).
(Someone wants to submit a PR with a simple edit to your docs, they
cannot fork) Unlike git LFS, `piggyback` doesn’t take over your standard
`git` client, it just perches comfortably on the shoulders of your
existing GitHub API. Data can be versioned by `piggyback`, but relative
to `git LFS` versioning is less strict: uploads can be set as a new
version or allowed to overwrite previously uploaded data. `piggyback`
works with both public and private repositories.

`piggyback` is inspired by
[datastorr](https://github.com/ropenscilabs/datastorr) from
[@richfitz](http://github.com/richfitz). While `datastorr` provides your
data in the R “data package” model of `storr` (your data lives inside an
R package that you install and is automatically loaded into memory),
`piggyback` tries to imitate the LFS model where your data files are
simply stored as part of your repository. This implementation uses R
wrappers to the GitHub API, but an identical client could be implemented
in any language with bindings to the GitHub API.

`piggyback` is designed around two possible workflows: (1) A high-level
workflow that parallels `git lfs`, and (2) a simple interface for
directly uploading and downloading individual data files to a GitHub
release.

## Installation

You can install the development version from
[GitHub](https://github.com/) with:

``` r
# install.packages("devtools")
devtools::install_github("cboettig/piggyback")
```

## Authentication

No authentication is required to download data from *public* GitHub
repositories using `piggyback`. To upload data to any repository, or to
download data from *private* repositories, you will need to authenticate
first. To do so, add your [GitHub Token]() to an environmental variable,
e.g. in a `~/.Renviron` file in your home directory (or some place
private you won’t upload), or simply set it in the R console using:

``` r
Sys.setenv(GITHUB_TOKEN="xxxxxx")
```

Try to avoid writing `Sys.setenv()` in scripts – remember, the goal here
is to avoid writing your private token in any file that might be shared,
even privately.

## Basic Interface

`piggyback` provides two interfaces: a basic, more low-level interface
for uploading and downloading individual files, and a `git-lfs` type
interface to `pull` or `push` a list of tracked files. The basic
interface is often suitable for working with single files and gives more
flexibility.

### Downloading data

Download the latest version or a specific version of the data:

``` r
library(piggyback)
pb_download("cboettig/piggyback", "mtcars.tsv.gz")
```

Or a specific version:

``` r
pb_download("cboettig/piggyback", "mtcars.tsv.gz", tag = "v0.0.4")
```

Or simply omit the file name to download all assets connected with a
given release. you can also always specify a destination directory to
download.

``` r
dir.create("data")
pb_download("cboettig/piggyback", dest="data/")
```

### Uploading data

If your GitHub repository doesn’t have any
[releases](https://help.github.com/articles/creating-releases/) yet,
`piggyback` will help you quickly create one. Create new releases to
manage multiple versions of a given data file. While you can create
releases as often as you like, making a new release is by no means
necessary each time you upload a file. If maintaining old versions of
the data is not useful, you can stick with a single release and upload
all of your data there.

``` r
gh_new_release("cboettig/piggyback", "v0.0.1")
```

Once we have at least one release available, we are ready to upload. By
default, `pb_upload` will attach data to the latest release.

``` r
## We'll need some example data first.
## Pro tip: compress your tabular data to save space & speed upload/downloads
readr::write_tsv(mtcars, "mtcars.tsv.gz")

pb_upload("cboettig/piggyback", "mtcars.tsv.gz")
```

You can also simply overwrite the a previous version of the file on an
existing release, rather than creating a new release every time:

``` r
pb_upload("cboettig/piggyback", "mtcars.tsv.gz")
```

This is useful in scripts that may automatically upload their results,
or whenever a previous version of a data file is disposable.

## git-style

For an even simpler way to sync data files to GitHub, `piggyback`
provides LFS-like `push` and `pull` methods. These methods are simple
wrappers around the basic upload and download interface that streamline
the process of managing a potentially large number of data files of a
given type or location.

By default, specify a `glob` pattern of files to track. `pb_track` can
also track all files at a given path.

``` r
pb_track("*.tsv.gz")
pb_track(path = "data/")
```

Upload all tracked data to GitHub. By default, will attach data to the
`latest` release of the current project:

``` r
pb_push()
```

Download all tracked data:

``` r
pb_download()
```

Or download all tracked data from a particular previous release. Will
overwrite local data files by the same name, allowing you to quickly
switch between versions of the data:

``` r
pb_download(tag = "v0.0.1")
```

To save bandwidth / transfer times, files which are identical on both
GitHub and local filesystem are not transferred:

``` r
pb_push() 
pb_push() 
```

## A Note on GitHub Releases vs Data Archiving

`piggyback` is not intended as a data archiving solution. Importantly,
bare in mind that there is nothing special about multiple “versions” in
releases, as far as data assets uploaded by `piggyback` are concerned.
The data files `piggyback` attaches to a Release can be deleted or
modified at any time – creating a new release to store data assets is
the functional equivalent of just creating new directories `v0.1`,
`v0.2` to store your data. (GitHub Releases are always pinned to a
particular `git` tag, so the code/git-managed contents associated with
repo are more immutable, but remember our data assets just piggyback on
top of the repo).

Permanent, published data should always be archived in a proper data
repository with a DOI, such as [zenodo.org](https://zenodo.org). Zenodo
can freely archive public research data files up to 50 GB in size, and
data is strictly versioned (once released, a DOI always refers to the
same version of the data, new releases are given new DOIs). `piggyback`
is meant only to lower the friction of working with data during the
research process. (e.g. provide data accessible to collaborators or
continuus integration systems during research process, including for
private repositories.)

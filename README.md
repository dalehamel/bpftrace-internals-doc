# Template repository for new Pandoc reports

This is meant to make it easy to add a new report to pandoc.

## Getting started

1. Create a copy of this repo in your name on github by clicking the "Use this template button" 
1. Add docs to the docs folder in pandoc flavored markdown (they will be built in lexicographical order)
1. Run `make` to get your output

# Features

## Output formats

By default, this can output pdf (via pdflatex), epub (via pandoc) and html (via pandoc).

For html, you can apply a style. We use github by default.

## Code annotation

You can easily annotate code thanks to the inclusion of https://github.com/owickstrom/pandoc-include-code

## References

Citing work and giving people credit where it is due is very important! It also helps to build a community, help search-engines raise the profile of work that is linked to, and make it easy for readers to follow up on the sources for their own discovery.

We use pandoc-citeproc to parse bibliography.yml, and use the ieee reference format, though you cat substitute in anything that works with a sort of bibtex csl style.

# Dependencies

A \*nix system, with a modern gnu `make` and a `docker` build environment

[![Website](https://img.shields.io/badge/sqr--112-lsst.io-brightgreen.svg)](https://sqr-112.lsst.io)
[![CI](https://github.com/lsst-sqre/sqr-112/actions/workflows/ci.yaml/badge.svg)](https://github.com/lsst-sqre/sqr-112/actions/workflows/ci.yaml)

# Docverse documentation hosting platform design

## SQR-112

For a decade, Rubin Observatory has hosted its documentation sites with its LSST the Docs service. That service provided excellent capabilities and performance, but its implementation is now out of step with our current application design practices and some long-standing bugs and features have been difficult to retrofit. Docverse is the next iteration of LSST the Docs that retains the qualities of hosting versioned, static web documentation, but resolves the feature gaps, performance issues, and bugs we experienced with LSST the Docs.

**Links:**

- Publication URL: https://sqr-112.lsst.io
- Alternative editions: https://sqr-112.lsst.io/v
- GitHub repository: https://github.com/lsst-sqre/sqr-112
- Build system: https://github.com/lsst-sqre/sqr-112/actions/


## Build this technical note

You can clone this repository and build the technote locally if your system has Python 3.11 or later:

```sh
git clone https://github.com/lsst-sqre/sqr-112
cd sqr-112
make init
make html
```

Repeat the `make html` command to rebuild the technote after making changes.
If you need to delete any intermediate files for a clean build, run `make clean`.

The built technote is located at `_build/html/index.html`.

## Publishing changes to the web

This technote is published to https://sqr-112.lsst.io whenever you push changes to the `main` branch on GitHub.
When you push changes to a another branch, a preview of the technote is published to https://sqr-112.lsst.io/v.

## Editing this technical note

The main content of this technote is in `index.md` (a Markdown file parsed as [CommonMark/MyST](https://myst-parser.readthedocs.io/en/latest/index.html)).
Metadata and configuration is in the `technote.toml` file.
For guidance on creating content and information about specifying metadata and configuration, see the Documenteer documentation: https://documenteer.lsst.io/technotes.

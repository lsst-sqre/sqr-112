# Claude instructions

This project is a Rubin Observatory technical note describing the design of the Docverse versioned documentation hosting platform. Docverse is the successor to the LSST the Docs (LTD) platform described in https://sqr-006.lsst.io/.

This technote uses [Myst-Parser markdown](https://myst-parser.readthedocs.io/en/latest/) to support features like including other markdown files and rendering reStructuredText directives. The main content is organized into separate markdown files that are included into the main `index.md` file.

The configuration for the technote is provided by a technote.toml file; the schema for that is documented at https://documenteer.lsst.io/technotes/configuration.html. Follow https://documenteer.lsst.io/technotes/ for documentation on Rubin technotes.

To build the HTML version of this technote locally, run `uv run tox run -e html`.

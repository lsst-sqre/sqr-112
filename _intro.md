## Introduction

Rubin Observatory's technical documentation has long followed the docs-like-code model where documentation is authored in Git repositories, often alongside and in conjunction with code.
The LSST the Docs (LTD) application ({sqr}`006`) has played a critical role in this ecosystem by providing a platform for hosting versioned documentation sites.
For users, LTD's design ensures documentation is served directly through a CDN and object store hosted in the public cloud, so that performance and reliability are excellent even under high load and aren't affected by application-level issues.
For project staff, LTD provides a seamless experience for integrating documentation hosting with their development and deployment workflows.
When new branches or tags are pushed to a project, LTD creates documentation editions for those corresponding versions automatically.

### Lessons learned from LTD

After a decade of operating LTD, though, we have identified a number of area where we either wish to improve the platform, or resolve issues or limitations in the existing implementation:

- The LTD codebase is a Flask (synchronous) Python application, whereas we now build applications with FastAPI and use asyncio throughout our codebases.
- Edition updates need to be faster, near instant, and never produce 404s for users. With LTD, an edition update for a large documentation site could take nearly 15 minutes.
- We need greater flexibility in how LTD is configured and operates. For example, publishing projects as subpaths rather than subdomains.
- We needed to update and refine the project edition dashboards more easily.
- Build uploads are slow for large documentation sites.
- Need to be able to purge outdated draft editions and help projects ensure that their readers are using the default edition unless they explicitly choose a different one.

### From LTD to Docverse

Since around 2022 we started to work on a second version of LTD that filled some of these gaps.
However, in the scope of that work we wanted to retain the existing Flask codebase and maintain compatibility with the existing API, with only a limited set of new API endpoints.
At this point, we've realized that a more comprehensive reimplementation is needed to fully address the design goals.
Rather than maintain compatibility with LTD, we will migrate existing documentation sites and projects to the new platform.

### Key Docverse features and changes from LTD

- Implementation with FastAPI and Safir
- Queue system built on a backend-agnostic abstraction layer, with [Arq](https://arq-docs.helpmanual.io/) (via [Safir](https://safir.lsst.io/)) and Redis as the initial implementation, replacing the Celery system in LTD. The abstraction enables future evaluation of alternative queue backends without disrupting application logic.
- Works with Gafaelfawr tokens for authentication and group membership, replacing the custom token system in LTD
- Organization models to support multiple organizations with separate documentation domains and configurations hosted from the same Docverse instance
- Support for Cloudflare to provide instant edition updates.
- Improved build upload performance by allowing for multipart tarball uploads rather than requiring clients to upload individual files.
- Support for deleting draft editions, including automation with GitHub webhooks to delete draft editions for deleted branches.
- Edition dashboards are built from templates that are synchronized from GitHub so that the edition dashboards are easier to update and customizable by organization members.
- Support for the `versions.json` files used by pydata-sphinx-theme and other documentation themes to power client-side edition switching.

### Organization of this technote

This technote dives into the design of Docverse at a fairly technical level to provide a record of design decisions and a reference for implementation.

- [Organizations](#organizations) discusses the organization model, including how organization configuration is supported by the API and database schemas.
- [Authentication and authorization](#auth) describes how Docverse uses Gafaelfawr both to protect API endpoints for different roles, and to provide user and goup information for fine-grained access control.
- [Projects, editions, and builds](#projects) describes the the core data model that Docverse carries over from LTD, but with substantial improvements to capabilities and performance.
- [Documentation hosting](#documentation-hosting) explores CDN and edge compute architectures for serving documentation, explains how pointer mode eliminates the S3 copy-on-publish bottleneck, and how organizations configure hosting infrastructure.
- [Dashboard templating system](#dashboards) describes the new dashboard templating system that allows edition dashboards to be built from templates stored in GitHub repositories.
- [Code architecture](#code-architecture) describes Docverse's layered architecture, factory pattern for multi-tenant client construction, protocol-based abstractions for object stores and CDN providers, and the client-server monorepo structure including the Python client library and CLI.
- [Queue system](#queue) describes the queue system for processing edition updates and build uploads, built on a backend-agnostic abstraction with Arq as the initial implementation.
- [REST API design](#api) describes the API design and schema definitions for Docverse.
- [GitHub Actions action](#github-action) describes the native JavaScript GitHub Action for uploading documentation builds from GitHub Actions workflows.
- [Migration from LSST the Docs](#migration) covers the data and client migration plan for moving existing LTD deployments to Docverse, including migration tooling, phased rollout, and risk mitigation.

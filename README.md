# IDP.to developer help

This folder holds the source content for the IDP.to developer help site. It is written as self-contained Markdown with no dependencies on the main application, so it can be published on its own. Each topic lives in its own subdirectory with an `index.md` entry point (for example, [`oidc/`](oidc/) covers OIDC integration), and pages link to each other using relative paths so they stay valid wherever the site is hosted.

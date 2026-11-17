# theserver

Architecture and design for the AWS server-side components of **Take-off**.

Take-off is a web app built as a static single-page application (Svelte 5 + SvelteKit 2,
`adapter-static`). This repository does not contain the front-end code. It holds the
requirements, architecture decisions and designs for everything Take-off needs in AWS:

- **Hosting** the static SPA.
- **Authentication** of users through Google or Facebook accounts, or through
  an email-based account created for Take-off.
- **A secured API** the SPA calls to save and retrieve user data.
- **Data storage** where each user can reach only their own data.

## Contents

| Path | Purpose |
| --- | --- |
| [`high-level-requirements.md`](high-level-requirements.md) | What the server side must do, and the constraints it must meet. |
| [`development-plan.md`](development-plan.md) | Build order: scripts, AWS resources and tests for each stage. |
| [`architecture/`](architecture/) | Architecture diagrams and design documents. |
| [`architecture/deployment-diagram.md`](architecture/deployment-diagram.md) | Deployment diagram (Mermaid). |
| [`architecture/decisions.md`](architecture/decisions.md) | Log of architecture decisions. |
| [`architecture/data-design.md`](architecture/data-design.md) | S3 layout, project list format, main flows and API outline. |

## Status

Early design. Core AWS services are chosen (see [`architecture/decisions.md`](architecture/decisions.md)). Detailed design questions are still open; see
[`high-level-requirements.md`](high-level-requirements.md#5-open-questions).

## Viewing diagrams

Diagrams use [Mermaid](https://mermaid.js.org/). GitHub, GitLab and VS Code (with a Mermaid
preview extension) render them inline.

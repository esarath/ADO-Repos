# ADO-Repos

Base repo indexing multiple Azure DevOps (ADO) pipeline/project examples. Each subdirectory is one project, imported via `git subtree` so full upstream history is preserved.

## Projects

| Folder | Source | Description |
|---|---|---|
| [pipeline-JavaScript](./pipeline-JavaScript) | [Azure-Samples/js-e2e-express-server](https://github.com/Azure-Samples/js-e2e-express-server) | Node.js/Express sample app for Azure DevOps pipeline testing |
| [pipeline-dotNet](./pipeline-dotNet) | [MicrosoftDocs/pipelines-dotnet-core](https://github.com/MicrosoftDocs/pipelines-dotnet-core) | ASP.NET Core sample app for Azure DevOps pipeline testing |

## Docs

- [HLD](./docs/HLD.md) — architecture overview and design decisions (diagrams, trust model, best practices)
- [LLD](./docs/LLD.md) — full command reference, grouped by concern, for repeating the pattern on a new project/environment
- [OCP environment setup runbook](./docs/ocp-environment-setup.md) — step-by-step walkthrough for wiring dev/stg/prod deploy stages to lab.ocp.local namespaces with manual approval gates

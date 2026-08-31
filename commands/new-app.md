---
description: Create a new Kinotic application end to end
---

Create a new Kinotic application by following the workflow in the kinotic `create-app`
skill exactly: help the user sign up for Kinotic OS (or sign in) and connect the
kinotic-os MCP server, create the Application and its first Project, handle GitHub
linking and repository provisioning, clone the repository, verify the scaffold, define a
first entity, and push — then confirm the deployment reached `RUNNING` with
`Project Service Find Deployment`.

Use `$ARGUMENTS` as the application name if provided; otherwise ask the user for a name
and a one-line description before starting.

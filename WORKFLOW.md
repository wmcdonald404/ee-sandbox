# Overview
Basic documentation on the Github Actions workflows in the repository.

# Workflows
A description of each of the individual workflows, inc. context and usage where required.
## debug.yml
This is generally a playground for forking about with workflow syntax.
## ee-build.yml
Preps the Github runner for `a`nsible-builder``, builds a baseline Ansible execution environment, pushes to ACR, then rinse/repeat with an Azure-specific execution environment. 
## ee-run.yml
Logs in to ACR, constructs a simple dynamdic inventory, playbook, secrets and triggers a test run.
## matrix-demo.yml
This workflow demonstrates how to use a Github config variable as input into a matrix strategy.
The sample config variable is set at the environment level and contains:
`{"vms":[{"name":"vm1","az":1},{"name":"vm2","az":2},{"name":"vm3","az":3}]}`
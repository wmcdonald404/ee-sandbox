# Overview

# Workflows

## debug.yml
## ee-build.yml
## ee-run.yml
## matrix-demo.yml
This workflow demonstrates how to use a Github config variable as input into a matrix strategy.
The sample config variable is set at the environment level and contains:
`{"vms":[{"name":"vm1","az":1},{"name":"vm2","az":2},{"name":"vm3","az":3}]}`
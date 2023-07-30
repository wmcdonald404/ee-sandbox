# ee-sandbox
Messing about with Ansible Execution Environments and Github Actions. First we'll walk through the manual steps, end-to-end, to understand the process to automate. Then we'll demonstrate the same workflow but in an automated Github Actions workflow.  

This is will document the process to:
1. Manually prepare a demonstration environment to run `ansible-builder`
2. Manually build a baseline execution environment (EE) with the bare minimum `ansible-core` and `ansible-runner`
3. Push the resultant container image to a container registry
4. Wrap the preceding steps in a Github Action workflow
5. Extend the Github Action workflow to build an Azure-specific EE on top of that baseline EE, with the `azure.azcollection` and `azure-cli`. Providing a suitable environment for `ansible-runner` to target Azure 

Once a base EE workflow has been built and tested next we will outline the process to:
1. Configure Azure federated credentials to allow Github Actions authentication to Azure using OIDC
2. Test `ansible-runner` invoking the Azure EE execution environment

## Ubuntu Setup
First setup a build environment. This is running on Ubuntu 22.04 running on WSL2 for convenience, but should be easily modifiable for other environments. `ansible-builder` requires an RPM-based container image like Fedora, CentOS Stream or RHEL UBI in order to build its EEs.
- Update the apt cache
```
$ sudo apt-get update
```
- Install Python prequisites
```
$ sudo apt install python3 python3-pip3 python3-venv
```
- Install Podman as the container runtime
```
$ sudo apt install podman
```
- Create a Python virtual environment (venv)
```
$ mkdir -p ~/venv/ee
$ python3 -m venv ~/venv/ee/
```
- Activate the Python venv
```
$ . ~/venv/ee/bin/activate
```
- Upgrade pip
```
$ python3 -m pip install --upgrade pip
```
- Install Ansible and its execution environment tooling
```
$ pip install ansible ansible-builder ansible-runner ansible-navigator
```
***Note:*** only `ansible-builder` is **required** here but it's useful to have the other components in the local Python venv for reference.
- Grab a base container (`ansible-builder` will do this automatically, this is just in case it proves useful to spin container instances in advance to validate contents and/or behaviour.)
```
$ podman pull registry.fedoraproject.org/fedora:38
```
- Invoke a container to test
```
$ podman run --rm -it --name fedora-demo fedora:38
```
(If running the container with `-dit`, `podman attach <container-id>`, CTRL-P, CTRL-Q to detach and leave running if needed.)
## Manual Baseline Execution Environment Preparation & Build
- Set the initial environment variables
```
$ export EE_BASELINE=ee-baseline
```
- Create a directory
```
$ mkdir -p ~/ee/${EE_BASELINE}
```
- Create an execution environment definition
```
$ cat > ~/ee/${EE_BASELINE}/${EE_BASELINE}.yml <<EOF
version: 3
images:
  base_image:
    name: registry.fedoraproject.org/fedora:38
build_arg_defaults:
  ANSIBLE_GALAXY_CLI_COLLECTION_OPTS: '-vvv'
dependencies:
  ansible_core:
    package_pip: ansible-core
  ansible_runner:
    package_pip: ansible-runner
EOF
```
- Build the baseline execution environment
```
$ ansible-builder build -f ~/ee/${EE_BASELINE}/${EE_BASELINE}.yml -c ~/ee/${EE_BASELINE}/context/ -t ${EE_BASELINE} -v3
```
- Inspect the build contents
```
$ podman images
```
- Test the baseline execution environment
```
$ podman run --rm -it ${EE_BASELINE}
bash-5.2$ ansible --version
ansible [core 2.15.1]
  config file = None
  configured module search path = ['/runner/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/local/lib/python3.11/site-packages/ansible
  ansible collection location = /runner/.ansible/collections:/usr/share/ansible/collections
  executable location = /usr/local/bin/ansible
  python version = 3.11.3 (main, May 24 2023, 00:00:00) [GCC 13.1.1 20230511 (Red Hat 13.1.1-2)] (/usr/bin/python3)
  jinja version = 3.1.2
  libyaml = True

bash-5.2$ ls -l /usr/local/bin/ansible*
-rwxr-xr-x 1 root root  216 Jul  3 14:10 /usr/local/bin/ansible
-rwxr-xr-x 1 root root  217 Jul  3 14:10 /usr/local/bin/ansible-config
-rwxr-xr-x 1 root root  246 Jul  3 14:10 /usr/local/bin/ansible-connection
-rwxr-xr-x 1 root root  218 Jul  3 14:10 /usr/local/bin/ansible-console
-rwxr-xr-x 1 root root  214 Jul  3 14:10 /usr/local/bin/ansible-doc
-rwxr-xr-x 1 root root  217 Jul  3 14:10 /usr/local/bin/ansible-galaxy
-rwxr-xr-x 1 root root  220 Jul  3 14:10 /usr/local/bin/ansible-inventory
-rwxr-xr-x 1 root root  219 Jul  3 14:10 /usr/local/bin/ansible-playbook
-rwxr-xr-x 1 root root  215 Jul  3 14:10 /usr/local/bin/ansible-pull
-rwxr-xr-x 1 root root  222 Jul  3 14:10 /usr/local/bin/ansible-runner
-rwxr-xr-x 1 root root 1700 Jul  3 14:10 /usr/local/bin/ansible-test
-rwxr-xr-x 1 root root  216 Jul  3 14:10 /usr/local/bin/ansible-vault

bash-5.2$ ansible-doc -l | cat | head
ansible.builtin.add_host               Add a host (and alternatively a grou...
ansible.builtin.apt                    Manages apt-packages
ansible.builtin.apt_key                Add or remove an apt key
ansible.builtin.apt_repository         Add and remove APT repositories
<snipped for clarity>
```
Note that only ansible.builtin core modules are currently installed. The execution environment will need additional collections to be more useful.
At this stage, we can push the build container image to a registry, or build on top of it with additional collections to create purpose-specific execution environments (Azure, AWS, or GCP specific EEs, for example.)

## Github Action Execution Environments Preparation & Build
Now we understand the manual process, we can wrap this into a Github Actions workflow. [The workflow](https://github.com/wmcdonald404/ee-sandbox/blob/main/.github/workflows/ee-build.yml) currently runs a job `ee-build` which can be broken down into:
### Github Action Common Preamble & Prerequisites
- name: Log in to registry
- name: Install ansible-builder Python requirements

See: https://github.com/wmcdonald404/ee-sandbox/blob/main/.github/workflows/ee-build.yml#L15-L24
### Github Action Baseline EE Prep, Build, Push
- name: Prepare Baseline Execution Environment config
- name: Build Baseline Execution Environment image
- name: Push Baseline Execution Environment image

See: https://github.com/wmcdonald404/ee-sandbox/blob/main/.github/workflows/ee-build.yml#L26-L56
### Github Action Azure EE Prep, Build, Push
- name: Prepare Azure Execution Environment config
- name: Build Azure Execution Environment image
- name: Push Azure Execution Environment image

See: https://github.com/wmcdonald404/ee-sandbox/blob/main/.github/workflows/ee-build.yml#L26-L56
## Azure OIDC Preparation for Github Actions
### Create an Azure AD Application and Service Principal
- Log in to Azure
```
$ az login
```
- Set a default subscription
```
$ az account set -s <subscription-id>
```
- Create an Azure AD application registration
```
$ az ad app create --display-name aad-app-github-actions-oidc
```
***Note*** the "appId" returned in the JSON as your `<app-client-id>`

***Note*** the "id" returned in the JSON as your `<app-object-id>`
- Create a service principal for the Azure AD application
```
$ az ad sp create --id <app-client-id>
```
***Note*** the "id" returned in the JSON as your `<sp-id>`

***Note*** the "appOwnerOrganizationId" as your `<tenant-id>`
- Assign the contributor role to the service principal for the subscription
```
$ az role assignment create --role contributor --subscription <subscription-id> --assignee-object-id  <sp-id> --assignee-principal-type ServicePrincipal --scope /subscriptions/<subscription-id>
```
***Note*** the upstream example demonstrates limiting the scope for this role to a specific resource group
### Add Federated Credentials
- Set some temporary environment variables
```
$ export APPLICATION-OBJECT-ID=<app-object-id>
$ export CRED_NAME=aad-app-github-actions-oidc-fcred
$ export SUBJECT=repo:wmcdonald404/ee-sandbox:environment:test
```
- Create JSON for the federated credential creation parameters
```
$ cat > ~/credentials.json <<EOF
{
    "name": "${CRED_NAME}",
    "issuer": "https://token.actions.githubusercontent.com",
    "subject": "${SUBJECT}",
    "description": "Github Action Test Federated Credential",
    "audiences": [
        "api://AzureADTokenExchange"
    ]
}
EOF
```
- Create the federated credential
```
$ az ad app federated-credential create --id <app-object-id> --parameters credential.json
```
### Create Github Environment & Secrets
- Set the default repository for the Github CLI
```
$ export GH_REPO=wmcdonald404/ee-sandbox
```
- Create the test Github deployment environment
```
$ gh api -X PUT /repos/wmcdonald404/ee-sandbox/environments/test 
```
- Create the secrets needed for OIDC-based authentication
```
$ gh secret set AZURE_CLIENT_ID -e test -a actions -b <client-id>
$ gh secret set AZURE_TENANT_ID -e test -a actions -b <app-tenant-id>
$ gh secret set AZURE_SUBSCRIPTION_ID -e test -a actions -b <subscription-id>
```

## Running Github Actions
- List the Github Actions workflows
```
$ export GH_REPO=wmcdonald404/ee-sandbox
$ gh workflow list
Build Ansible Execution Environments  active  64655926
Run Ansible execution environment     active  64655927
```
- View the details of a workflow
```
$ gh workflow view 64655927
Run Ansible execution environment - ee-run.yml
ID: 64655927
```
- Run a workflow (either of the following are synonymous)
```
$ gh workflow run ee-run.yml
$ gh workflow run 64655927
```
- Watch the workflow execute
  - `gh run watch` will automatically watch the most recent run
  - `gh run list` and then `gh run watch <run-id>` will watch a specific run
```
gh run watch
```
- View the detailed output of the run
```
gh run view
gh run view --job=<job-id>
gh run view --job=<job-id> --log
```

# References
## Execution Environment References
- https://ansible.readthedocs.io/projects/navigator/installation/#requirements-windows
- https://www.ansible.com/blog/automating-execution-environment-image-builds-with-github-actions
- https://github.com/cloin/ee-builds
## Github Actions for Azure References
- https://github.com/marketplace/actions/azure-login
- https://github.com/marketplace/actions/azure-login#configure-a-federated-credential-to-use-oidc-based-authentication
- https://learn.microsoft.com/en-us/azure/developer/github/connect-from-azure?tabs=azure-portal%2Clinux
- https://learn.microsoft.com/en-us/azure/active-directory/workload-identities/workload-identity-federation-create-trust?pivots=identity-wif-apps-methods-azp#github-actions
## Github REST API and CLI
- https://docs.github.com/en/rest/deployments/environments?apiVersion=2022-11-28
- https://cli.github.com/manual/gh
# Notes
- "Here documents" in Github Actions are very particular about trailing whitespace for the end token.
- Pay careful attention to the execution environment schema version. Copying v1 examples for a v3 install will cause confusion.
# Documentation Snippets
- Snapshot the packages for future reference
```
$ pip freeze > ~/repos/ee-sandbox/ansible-base-packages.txt
```
- Install Python poetry since `pip search` no longer works.
```
See: https://stackoverflow.com/questions/17373473/how-do-i-search-for-an-available-python-package-using-pip
$ sudo apt install python3-poetry
```
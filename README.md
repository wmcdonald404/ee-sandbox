# ee-sandbox
Messing about with Ansible Execution Environments and Github Actions. 

This is intended to outline the process to:

1. Prepare an environment to run `ansible-builder` in order to...
2. First build a baseline execution environment (EE) with the bare minimum `ansible-core` and `ansible-runner` and then...
3. Then build an Azure-specific EE on top of that baseline, with the `azure.azcollection` and `azure-cli` in order to provide a suitable environment for `ansible-runner` targeting Azure 
4. Test `ansible-runner` invoking the execution environment
5. Wrap the process into a Github actions (GHA) pipeline

First we'll walk through the manual steps, end-to-end, to understand the process to automate. Then we'll demonstrate the same workflow but in an automated GHA pipeline.  

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
nb: only `ansible-builder` is **required** here but it's useful to have the other components in the local Python venv for reference.

- Grab a base container (`ansible-builder` will do this automatically, this is just in case it proves useful to spin container instances in advance to validate contents and/or behaviour.)
```
$ podman pull registry.fedoraproject.org/fedora:38
```
- Invoke a container to test
```
$ podman run -it --name fedora-demo fedora:38
```
(If running the container with `-dit`, CTRL-P, CTRL-Q to detach and leave running.)

## Manual Baseline Execution Environment Preparation
These manual steps are now baked into the pipeline in .github/workflows/ee-deploy.yml and will execute self-contained in the Github Action runner when triggered.

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

## Github Action Baseline Execution Environment Preparation

Now we understand the manual process, we can wrap this into a Github Actions workflow. [The workflow](https://github.com/wmcdonald404/ee-sandbox/blob/main/.github/workflows/ee-deploy.yml#L19-L105) currently runs a job 'build' which includes the steps below:

- name: Install ansible-builder python requirements
- name: Prepare baseline execution environment config
- name: Build baseline execution environment image
- name: Push baseline execution environment image
- name: Prepare Azure execution environment config
- name: Build azure execution environment image


# References

- https://ansible.readthedocs.io/projects/navigator/installation/#requirements-windows
- https://www.ansible.com/blog/automating-execution-environment-image-builds-with-github-actions
- https://github.com/cloin/ee-builds

Install Python poetry since `pip search` no longer works.
See: https://stackoverflow.com/questions/17373473/how-do-i-search-for-an-available-python-package-using-pip
$ sudo apt install python3-poetry


# Notes

- "Here documents" in Github Actions are very particular about trailing whitespace for the end token.
- Pay careful attention to the execution environment schema version. Copying v1 examples for a v3 install will cause confusion.


# Documentation Snippets

- Snapshot the packages for future reference
```
$ pip freeze > ~/repos/ee-sandbox/ansible-base-packages.txt
```
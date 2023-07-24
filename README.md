# ee-sandbox
Messing about with Ansible Execution Environments

## Ubuntu Setup
Run from Ubuntu 22.04 (running on WSL2).

Fedora, CentOS Stream or RHEL UBI required for EEs.

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

- Snapshot the packages for future reference
```
$ pip freeze > ~/repos/ee-sandbox/ansible-base-packages.txt
```

- Grab a base container (`ansible-builder` will do this for us, this is just in case we want to spin it up beforehand to validate contents.)

```
$ podman pull registry.fedoraproject.org/fedora:38
```

- Invoke a container and attach to it
```
$ podman run -dit --name fedora-demo fedora:38
$ podman attach fedora-demo
```

(CTRL-P, CTRL-Q to detach and leave running.)

## Execution environment prep

- Create a directory
```
$ mkdir -p ~/ee-sandbox/ee-baseline
```

- Create an execution environment definition
```
$ cat > ~/ee-sandbox/ee-baseline/execution-environment.yml <EOF
---
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

- Build an execution environment
```
$ ansible-builder build -f ~/repos/ee-sandbox/ee-baseline/execution-environment.yml -t ee-baseline:latest -v3
```

- Test the execution environment

```
$ podman run --rm -it ee-baseline
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

# References

- https://ansible.readthedocs.io/projects/navigator/installation/#requirements-windows
- https://www.ansible.com/blog/automating-execution-environment-image-builds-with-github-actions
- https://github.com/cloin/ee-builds

Install Python poetry since `pip search` no longer works.
See: https://stackoverflow.com/questions/17373473/how-do-i-search-for-an-available-python-package-using-pip
$ sudo apt install python3-poetry

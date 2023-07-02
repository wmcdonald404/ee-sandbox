# ee-sandbox
Messing about with Ansible Execution Environments

## setting up
Based on Ubuntu 22.04 (running on WSL2)

- Update the apt cache
$ sudo apt-get update

- Install Python prequisites
$ sudo apt install python3 python3-pip3 python3-venv

- Install Podman as the container runtime
$ sudo apt install podman

- Create a Python virtual environment (venv)
$ mkdir ~/venv/ee
$ python3 -m venv ~/venv/ee/
$ . ~/venv/ee/bin/activate

- Activate the Python venv
$ python3 -m pip install --upgrade pip

- Install Ansible and its execution environment tooling
$ pip install ansible ansible-builder ansible-runner ansible-navigator

- Snapshot the packages for future reference
$ pip freeze > ~/repos/ee-sandbox/ansible-base-packages.txt


# References

- https://ansible.readthedocs.io/projects/navigator/installation/#requirements-windows
- https://www.ansible.com/blog/automating-execution-environment-image-builds-with-github-actions
- https://github.com/cloin/ee-builds

Install Python poetry since `pip search` no longer works.
See: https://stackoverflow.com/questions/17373473/how-do-i-search-for-an-available-python-package-using-pip
$ sudo apt install python3-poetry

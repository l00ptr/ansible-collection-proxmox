# Contribution Guide

Below you will find the information needed to contribute to this project.

Note that by contributing to this collection, you agree with the code of conduct you can find [here.](/CODE_OF_CONDUCT.md)

## Getting Started

To begin development on this collection, you need to have the following dependencies installed:

- A python version that supports the release of `ansible-core` in `requirements.txt` (see ["Control node Python" here](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html#node-requirement-summary))

To get started quickly:

1. Fork the repository and clone it to your local machine
2. Run `./scripts/setup.sh` to configure a local dev environment (virtualenv) with all required dependencies
3. Activate the virtualenv with `source .venv/bin/activate`
4. Make your changes and commit them to a new branch
5. Run the tests locally by running `./tests/test-<target>`. See below for more details on these testing scripts
6. Once you're done, commit your changes (make sure that you are in the venv).
   Pre-commit will format your code and check for any obvious errors when you do so.

## Developing Modules

Modules in this collection typically wrap around endpoints in the Proxmox VE and PBS REST APIs.
Feel free to add new modules for endpoints not yet covered by this collection, but try to prevent
overlap with the existing modules in `community.general` (such as `proxmox` and `proxmox_kvm`, see [here](https://docs.ansible.com/ansible/latest/collections/community/general/index.html#plugin-index) for a complete list).

Here are some general hints for module development:

- All modules should target Python 3.6 as the minimum supported version
- Read the [Ansible module conventions](https://docs.ansible.com/ansible/latest/dev_guide/developing_modules_best_practices.html)
- Use the pre-existing `doc_fragments` and `module_utils` python modules where applicable. Feel free to use an existing module as a base
- Try to make the calls idempotent where possible. Modules should always support check mode

## Developing Roles

Each role in this collection performs a complex task to bring a remote host into a desired state.
If you want to write a new role, look to the existing ones for inspiration.

Some general guidelines:

- Try to support most common Linux distributions (including Ubuntu, Debian, Fedora and Rockylinux)
- Keep the configuration for the user simple and try to provide sensible defaults where possible
- Try to avoid using complex data structures as role variables/parameters, use simple values that can be composed easily instead.

## Testing Infrastructure

Writing tests for your contributions ensure that they continue working in the future.
In addition to the testing venv, you will also need the following:

- `podman` 4.0 or newer for both ansible-test and molecule (see [here](#setting-up-podman) for a setup guide).

We use `tox` in conjunction with `tox-ansible` to run tests against both modules and roles with multiple Ansible versions.
If you set up your environment as described [above](#getting-started), you should already have everything installed.

To run tests, use the wrapper scripts in the `tests` directory from the collection root:

- [`./tests/test-modules`](./tests/test-modules) will run ansible-test against the modules in the collection and verify them
    - [`test-modules-sanity`](./tests/test-modules-sanity) performs basic syntax and validation checks
    - [`test-modules-integration`]((./tests/test-modules-integration)) runs the module integration [test targets](./tests/integration/targets/).
- [`./tests/test-roles`](./tests/test-roles) will use `molecule` to run tests on all roles in the collection (this might take a long time!)
    - To limit the scope to a single role, add a filter parameter: `./tests/test-roles pbs_client`
    - Alternatively, you can use `tox -l` to list all available scenarios and then select a single one with `tox -e <scenario-name-here>`

### Setting up Podman

To run role or module tests, you will need the following:

- `podman` version 4+ (as it comes with the new netvark networking stack)
- `aardvark-dns`, a plugin for netvark which provides DNS between containers in the same network

On Archlinux, you can install these with this command: `sudo pacman -S podman aardvark-dns`.
Other distributions may not have these versions [available in their repositories](https://pkgs.org/search/?q=podman), you might have install things manually.

**NOTE:** If you previously used an older (`<=3.x`) version of `podman` you will have to migrate to the new networking stack fist. This can be done with `podman system reset`

Finally, make sure that your user has a subuid/subgid configuration associated with them so that you can run rootless containers.
Check the `/etc/subuid` and `/etc/subgid` files for entries corresponding to your username.
If none are found, you can add them like so: `sudo usermod --add-subuids 100000-165535 --add-subgids 100000-165535 <USERNAME>` (make sure that the range is not  already taken by another user in `/etc/subuid`/`/etc/subgid`).

Once you have applied your changes, run `podman system migrate` to force `podman` to pick up the new configuration.

That's it! Podman should now be working! To test it, you can run a container just like with docker: `podman run --rm -it ubuntu bash`


### Module Tests

The sanity tests should "just work" - they can give you valuable feedback and uncover accidental mistakes like mismatches between your module docs and argspec.

There are currently no integration tests due to difficulties with spinning up a complete PVE instance on-the-fly - Vagrant might be an option to look into in the future here.

### Role Tests

Testing Ansible roles ensures that they do exactly what we want them to and nothing more.
We perform role tests using `molecule` with the `molecule-podman` driver with rootless containers.

---

Why not just `molecule-docker`? Simple: This collection needs systemd inside its test containers to start applications among other things, and modern versions of [systemd do not work inside docker containers anymore](https://github.com/ansible-community/molecule/discussions/3108).
Meanwhile, `podman` has official support for passing through the host systemd and with the 4.0 release, podman networking is now mature enough that we can use it in our role tests. In addition, `podman` offers daemonless, rootless containers, which improves security considerably.

---

Before you run any tests, make sure you follow the instructions in [the following section](#setting-up-podman).

Once you're done, you can:

- See all available scenarios with `tox -l` and look for the scenarios starting with `ansible-py*`.
- Run all molecule scenarios with `tests/test-roles`
- Pass a (partial) scenario name as a filter to limit execution (for example, `/tests/test-roles pbs_client` to limit molecule scenarios to the ACME role only)

#### Writing Role Tests

There are tons of good guides online for how to write tests using molecule.
Alternatively, you can always look at the existing molecule scenarios in this collection

When creating a new molecule scenario, your directory structure should look like this:

```
some_role/
  defaults/
  meta/
  molecule/
    default/
      converge.yml
      molecule.yml
      prepare.yml
      requirements.txt # --> symlink to /requirements-molecule.txt
      verify.yml
    another-scenario/
      ...
  tasks
  ...
```

The `requirements.txt` symlink is used by `tox-ansible` when running tests via `tox` to install a specific, known-good version of `molecule` and the `molecule-podman` driver.

Below is a shortened example `molecule.yml` that you can include in your own scenarios:

```yaml
---
dependency:
  name: galaxy
driver:
  name: podman
platforms:
  # Example for a systemd-enabled test container
  - name: pbs-client-ubuntu-22
    # We use the images provided by geerlingguy where possible, as they provide out-of-the-box
    # support for Ansible (pre_build_image=true, speeds up testing).
    image: "docker.io/geerlingguy/docker-ubuntu2204-ansible"
    # Grouping your hosts makes it easier to address them in your playbooks
    groups:
      - ubuntu
    # By default, molecule overrides the container start command with a while-true loop.
    # Instead, we want systemd to be the init command, which is already the case for geerlingguy images
    override_command: false
    # When false, this causes molecule to modify the container so that ansible can connect to it.
    # This is not needed for geerlingguy containers
    pre_build_image: true
    # Force podman systemd integration. By default, podmans systemd detection only uses a few specific paths,
    # which might get not detected on some distros
    systemd: always
    # Assign the container to a non-default network. This is required when you want inter-container communication.
    network: molecule-pbs-client

#... more platforms here
provisioner:
  name: ansible
  env:
    # This is required for podman to function: https://github.com/ansible-community/molecule-podman/issues/2
    ANSIBLE_PIPELINING: false
    #ANSIBLE_VERBOSITY: 3 # enable for debugging
  #inventory:
    #group_vars:
      #all:
        # set global inventory vars here

scenario:
  test_sequence:
    - lint
    - destroy
    - dependency
    - syntax
    - create
    - prepare
    - converge
    - idempotence
    - check # also run check mode in regular tests
    - side_effect
    - verify
    - destroy
verifier:
  name: ansible
```

## Collection Docs

In addition to the `README.md`s, we use `antsibull-docs` to generate sphinx documentation for both modules and roles (from the `meta/argument_specs.yml` file).
See [here](https://docs.ansible.com/ansible/latest/dev_guide/developing_collections_documenting.html) for more information about the build process.

The CI also builds the docs to ensure they don't break silently.

## Misc Lifecycle Maintainer Information

- This project uses sematic versioning. Version numbers and  releases/changelogs are automatically generated using [release-drafter](https://github.com/release-drafter/release-drafter), utilizing pull request labels.
- When merging a pull request, make sure to select an appropriate label (pr-bugfix, pr-feature, etc.).
  Release-drafter will automatically update the draft release changelog and the galaxy.yml version will be bumped if needed.
- Once a draft release is actually published, collection packages will be added to the release and ansible-galaxy automatically.
- If you need to manually bump the collection version, run the `update-version` script and adjust the test versions if required.

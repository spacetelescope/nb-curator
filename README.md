# nb-curator

DRAFT DRAFT DRAFT  -- subject to weekly/daily change top to bottom

## Overview

nb-curator is designed to streamline the process of curating JupyterLab notebooks,
their associated runtime environments, and ultimately to support automatically
building and testing Docker images focused the requirements of specific sets of
notebooks.  Towards that end, nb-curator supports these functions:

- bootstrapping a dedicated environment where nb-curator runs
- loading, saving, and validating notebook curation specs
- cloning associated notebook and image build repos
- creating a dedicated target environment to install notebook package requirements
- compiling loose notebook requirements.txt files into fully versioned dependency requirements for the target environment
- installing notebook dependencies in the target environment
- explicitly testing all top-level notebook imports in the installed target environment
- running notebooks headless in the target environment
- injecting relevant package, test, and notebook outputs into an external notebook image build system
- submitting a completed spec and/or related image build PR to trigger an automatic build
- various cleanup tasks removing clones, packages, environments, etc.

There are a couple of relatively new foundational tools being used:

- micromamba -- a self-contained little brother of mamba (the better free OSS version of conda)
- uv -- a whole new pip-like system written in Rust leading to faster dependency solutions and package installs

The intent of `nb-curator` is to install 2-3 dedicated environments under `$HOME/.nb-curator`:

- micromamba -- self-contained minimalistic install tool, not a base environment
- nbcurator  -- a true micromamba environment in which nb-curator runs with required dependencies
- <target environment> -- the notebook environment we're curating defined by the YAML spec (or, possibly, CLI)

These environments are interdependent but fully independent of your other pre-existing Python environments.

## Installing

Bootstrapping the system will create the .nb-curator dir and nbcurator environment under $HOME.

```bash
curl https://raw.githubusercontent.com/spacetelescope/nb-curator/refs/heads/main/nb-curator >nb-curator
chmod +x nb-curator
./nb-curator bootstrap
source ./nb-curator environment
```

After that, the nb-curator "curation" environment can be re-activated by:

```bash
source ./nb-curator environment
```

Consider putting the nb-curator bash script in your path.
Consider putting that in your shell RC file.

The target environment can be activated by:

```bash
source nb-curator activate ENVIRONMENT_NAME
```

and either nbcurator or the target environment can be deactivated by:

```bash
source nb-curator deactivate
```

## Example Usage

Curator prepares custom version of prototype_protocol.yaml
Curator prepares a curation Python environment with the spec'ed version of Python
Then:

```bash
nb-curator  spec.yaml  --init-env

nb-curator  spec.yaml   --compile

nb-curator  spec.yaml   --install

nb-curator  spec.yaml   --test
```

## Basic Flow

The basic flow of the curator is to command different steps of the overall
process to execute or not on a per-run basis.  Eventually this enables skipping
over aspects of the process which have already been successfully completed and
iterating on the current task, e.g. not constantly recompiling and
re-installing pacackages while iterating over failing notebook tests and
notebook updates.  If any step in the sequence fails, the process will exit
with an error status.  The following features/steps are generally gated by CLI
switches.

- Loads, validates, updates, and saves the YAML notebook specification.
  Validation is currently incomplete but checks for required keywords.

- Optionally clones the git repositories for the notebooks if a
  local clone does not already exist,  otherwise it updates the existing clones
  from their repos or does nothing if --clone is not specified.  --repos-dir is
  used to specify the directory where the git repositories are cloned and/or
  already exist, defaulting to a notebook-repos subdir of the current directory.

- Searches for relevant notebooks based on the notebook directory paths and
  include/exclude patterns.

- Searches for requirements.txt files which specify Python package version
  constraints at a granular level of single notebooks.  Exactly what to include
  is a WIP,  but at a minimum one optional requirements.txt per notebook.

- Automatically creates a basic Python environment in which packages will be
  installed and tested.   The overall paradigm of the nb-curator tool is
  that it installs packages and tests notebooks with respect to the current
  Python environment.

- Optionally initializes (--init-env) a target environment to support
  compilation of requirements, package installation, and testing.  In addition
  to installing a handful of utility packages, it creates a JupyterLab kernel
  for the environment that is required for notebook testing or using it in
  JupyterLab.  This is useful even if the curator chooses to use their own
  custom environment (TBD supported or not) as the target since these packages
  and kernel setup are required regardless.

- If --compile is specified, it will create both a conda environment .yml file
  and a locked pip requirements.txt file based on compiling all the discovered
  notebook requirements.txt simultaneously, with the goal of creating a package
  version spec suitable for running ALL of the notebooks.  If --compile is not
  specified it will continue to use the last set of compiled packages from the
  spec.

- If --install is specified, it will install the compiled versions of packages
  in the conda environment, which XXXXX again at this time is the runtime
  environment. After installation, it will attempt to import any package which
  is explicitlylisted in a notebook file as a basic sanity check.

- If --test-notebooks is specified, run notebooks matching any of the
subsequent comma separated list of notebook names or regular expressions.  If
no notebooks or regexps are specified, it will run all notebooks.  This is a
headless crash test which runs up to --jobs [n] notebooks in parallel using a
--timeout [seconds] to kill runaway notebooks.

- If --delete-clones is specified,  it will remove all cloned repositories.

- If --reset-spec is specified it will remove the output section of the spec.

- If --delete-env is specified,  it will remove the entire target environment.
  This dedicated environment approach prevents contamination
  between iterations of the tool as packages come-and-go from the spec but are
  never removed from the target environment.

- If optional/proposed --submit-for-build is specified,  the spec is forwarded
  to the CI chain,  key information is supplied to the build framework, and a
  corresponding image is automatically built and pushed to the hub assuming all
  goes well.   This is all still TBD pending interest/approval of the above.

- If --spi-inject is specified,  writes key output information from spec like
mamba and pip requirements, import tests, and supported notebooks into a clone
of the science platform images build enabling a manual build as if a dev had
manually updated all the input requirements.

# Imperial College HPC Action

**This action is currently extremely experimental and will only work when run
using a dedicated self-hosted runner provided by Imperial College. To discuss
using this action please contact c.cave-ayland@imperial.ac.uk.**

This GitHub Action can be used to run workloads on the Imperial College Research
Computing Service (RCS) via batch jobs. Batch jobs are currently limited to 8
cpus and 96gb memory for 30 minutes.

This action is designed to support the adoption of Continuous Integration (CI)
as a software engineering practice for developers of High Performance Computing
(HPC) codes or general users of the RCS. It is not intended to be used as a
facility to run production simulation or data analysis workloads. Potential uses
as part of a CI pipeline include:

* Memory or compute intensive compilations
* Building/testing codes with complex dependencies e.g. MPI, FFTW, HDFS
* Access to proprietary dependencies e.g. Intel compilers, Gaussian
* Testing of multi-process/multi-thread applications
* Running memory/computationally intensive test cases
* Periodic checks that a code continues to work within the RCS environment

## Example Usage

The following would be saved as a YAML file within `.github/workflows` for the
repository that uses the action.

```yaml
on: push
jobs:
  hpc:
    runs-on: hpc
    steps:
      - uses: actions/checkout@v2
      - name: HPC Step
        uses: imperialcollegelondon/hpc_github_action@main
        with:
          job-script-contents: |
            # setup the environment
            conda install -y python=3.9
            git clone https://github.com/essex-lab/ProtoMS.git
            pip install -r ProtoMS/requirements.txt
            module load cmake/3.18.2 intel-suite/2020.2 mpi/intel-2019.6.166

            # build the software
            mkdir ProtoMS/build && cd ProtoMS/build
            FC=ifort cmake ..
            make -j 8
            make install
            cd ..

            # run an MPI test workload
            export PROTOMSHOME=$(pwd)
            cd tutorial/gcmc
            python $PROTOMSHOME/tools/make_gcmcbox.py -b 32.0 7.0 2.0 3.5 4.0 8.0 -o gcmc_box.pdb
            python $PROTOMSHOME/protoms.py -s gcmc -sc protein_pms.pdb --gcmcbox gcmc_box.pdb --adams {-24..-17} --nequil 0 --nprod 100

            mpirun -n 8 $PROTOMSHOME/protoms3 run_gcmc.cmd
```

## Details

This action works via a specialised self-hosted GitHub Actions Runner. Access to
the runner must be enabled **by request** for each repository that uses it. The
action has one required input argument `job-script-contents` that provides the
commands to be run inside a batch job on the RCS cluster.

Batch jobs run in a dedicated user environment on the cluster. This provides
access to the usual selection of modules and utilities. An empty Conda
environment is created and activated for each job. Subsequent Conda commands can
be used to install packages into the environment as required. In the example
above, Conda is used to install Python 3.9 then additional dependencies are
installed with pip.

Versioned releases of this action are not provided and you should run using the
latest `main` branch (as in the provided example). This is because the action
will periodically need to be updated in response to configuration changes on the
cluster. Only the latest `main` branch should be expected to work.

## Caveats/Constraints

* This action is currently highly experimental and subject to change or
  withdrawal without notice.
* As noted above, use of this action requires access to a specialised
  self-hosted runner provided on request for individual repositories. At time of
  writing, access to the runner is not available from the College's enterprise
  GitHub deployment (github.ic.ac.uk).
* There are potential security risks to using this action. This action allows
  custom code execution from authorised repositories. It may be possible for
  other users of this action to access any files placed on the self-hosted
  runner. This includes any versions of your repository used in a workflow with
  this action or any files created during said workflow.
* Workflows using the `hpc_github_action` should not use any other actions
  (besides the `checkout` action).
* Workflows that do not include the `hpc_github_action` MUST NOT be run using
  the specialised self-hosted runner (i.e. must not include `runs-on: hpc`).
* Access to different computational resources (e.g. gpus, multinode) is not
  possible but please register your interest with c.cave-ayland@imperial.ac.uk.
* Batch jobs are run under a dedicated user account so access to user
  configuration profiles, shared project spaces, etc. are not possible. If you
  have a use case that would benefit from these (e.g. access to a large
  reference dataset or pre-configuration of complex dependencies) then please
  get in touch to discuss.
* Besides the below noted exception your job must not make any changes outside
  its working directory. For instance your job should not create any new Conda
  environments or install packages into user locations e.g you should not use
  `pip install --user`.
* Your job may create files in $TMPDIR or $EPHEMERAL. To minimise the chance of
  interfering with other jobs, files in $EPHEMERAL should be placed in a
  dedicated sub-directory. Good practice is to use a random directory name e.g.
  `job_tmp=$(mktemp --directory --tmpdir="$EPHEMERAL")`.
* Jobs executed using the action must queue before execution. Typical queue
  times are a few minutes but this may be longer during periods of high usage.

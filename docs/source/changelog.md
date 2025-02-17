(changelog)=

# Changelog

Changes in IPython Parallel

## 7.0.0

**prerelease** there are some big things coming! This is currently a prerelease to get some things out there for testing

- **Require Python 3.6**
- Fix compatibility issues with ipykernel 6, jupyter-client 7
- New {class}`.Cluster` API for managing clusters from Python,
  including support for signaling and restarting engines.
- New {meth}`.Client.signal_engines` for sending signals to single engines.
- New KernelNanny process for signaling and monitoring engines
  for improved responsiveness of handing engine crashes.
- New prototype {class}`.BroadcastScheduler` with vastly improved scaling in 'do-on-all' operations
  on large numbers of engines,
  c/o Tom-Olav Bøyum's Master's thesis at University of Oslo.
- New {meth}`.Client.wait_for_engines(n)` method to wait for engines to be available.
- Nicer progress bars for interactive waits, such as {meth}`.AsyncResult.wait_interactive`.
- Add {meth}`.AsyncResult.stream_output` context manager for streaming output.
  Stream output by default in parallel magics.
- Launchers registered via entrypoints for better support of third-party Launchers.
- New JupyterLab extension (enabled by default) based on dask-labextension
  for managing clusters.

The repo has been updated to use pre-commit, black, myst, and friends and GitHub Actions for CI, but this should not affect users, only making it a bit nicer for contributors.

## 6.3.0

- **Require Python 3.5**
- Fix compatibility with joblib 0.14
- Fix crash recovery test for Python 3.8
- Fix repeated name when cluster-id is set
- Fix CSS for notebook extension
- Fix KeyError handling heartbeat failures

## 6.2.5

- Fix compatibility with Python 3.8
- Fix compatibility with recent dask

## 6.2.4

- Improve compatibility with ipykernel 5
- Fix `%autopx` with IPython 7
- Fix non-local ip warning when using current hostname

## 6.2.3

- Fix compatibility for execute requests with ipykernel 5 (now require ipykernel >= 4.4)

## 6.2.2

- Fix compatibility with tornado 4, broken in 6.2.0
- Fix encoding of engine and controller logs in `ipcluster --debug` on Python 3
- Fix compatiblity with joblib 0.12
- Include LICENSE file in wheels

## 6.2.1

- Workaround a setuptools issue preventing installation from sdist on Windows

## 6.2.0

- Drop support for Python 3.3. IPython parallel now requires Python 2.7 or >= 3.4.
- Further fixes for compatibility with tornado 5 when run with asyncio (Python 3)
- Fix for enabling clusters tab via nbextension
- Multiple fixes for handling when engines stop unexpectedly
- Installing IPython Parallel enables the Clusters tab extension by default,
  without any additional commands.

## 6.1.1

- Fix regression in 6.1.0 preventing BatchSpawners (PBS, etc.) from launching with ipcluster.

## 6.1.0

Compatibility fixes with related packages:

- Fix compatibility with pyzmq 17 and tornado 5.
- Fix compatibility with IPython ≥ 6.
- Improve compatibility with dask.distributed ≥ 1.18.

New features:

- Add {attr}`namespace` to BatchSpawners for easier extensibility.
- Support serializing partial functions.
- Support hostnames for machine location, not just ip addresses.
- Add `--location` argument to ipcluster for setting the controller location.
  It can be a hostname or ip.
- Engine rank matches MPI rank if engines are started with `--mpi`.
- Avoid duplicate pickling of the same object in maps, etc.

Documentation has been improved significantly.

## 6.0.2

Upload fixed sdist for 6.0.1.

## 6.0.1

Small encoding fix for Python 2.

## 6.0

Due to a compatibility change and semver, this is a major release. However, it is not a big release.
The main compatibility change is that all timestamps are now timezone-aware UTC timestamps.
This means you may see comparison errors if you have code that uses datetime objects without timezone info (so-called naïve datetime objects).

Other fixes:

- Rename {meth}`Client.become_distributed` to {meth}`Client.become_dask`.
  {meth}`become_distributed` remains as an alias.
- import joblib from a public API instead of a private one
  when using IPython Parallel as a joblib backend.
- Compatibility fix in extensions for security changes in notebook 4.3

## 5.2

- Fix compatibility with changes in ipykernel 4.3, 4.4
- Improve inspection of `@remote` decorated functions
- {meth}`Client.wait` accepts any Future.
- Add `--user` flag to {command}`ipcluster nbextension`
- Default to one core per worker in {meth}`Client.become_distributed`.
  Override by specifying `ncores` keyword-argument.
- Subprocess logs are no longer sent to files by default in {command}`ipcluster`.

## 5.1

### dask, joblib

IPython Parallel 5.1 adds integration with other parallel computing tools,
such as [dask.distributed](https://distributed.readthedocs.io) and [joblib](https://joblib.readthedocs.io).

To turn an IPython cluster into a dask.distributed cluster,
call {meth}`~.Client.become_distributed`:

```
executor = client.become_distributed(ncores=1)
```

which returns a distributed {class}`Executor` instance.

To register IPython Parallel as the backend for joblib:

```
import ipyparallel as ipp
ipp.register_joblib_backend()
```

### nbextensions

IPython parallel now supports the notebook-4.2 API for enabling server extensions,
to provide the IPython clusters tab:

```
jupyter serverextension enable --py ipyparallel
jupyter nbextension install --py ipyparallel
jupyter nbextension enable --py ipyparallel
```

though you can still use the more convenient single-call:

```
ipcluster nbextension enable
```

which does all three steps above.

### Slurm support

[Slurm](https://hpc.llnl.gov/training/tutorials/livermore-computing-linux-commodity-clusters-overview-part-one) support is added to ipcluster.

### 5.1.0

[5.1.0 on GitHub](https://github.com/ipython/ipyparallel/milestones/5.1)

## 5.0

### 5.0.1

[5.0.1 on GitHub](https://github.com/ipython/ipyparallel/milestones/5.0.1)

- Fix imports in {meth}`use_cloudpickle`, {meth}`use_dill`.
- Various typos and documentation updates to catch up with 5.0.

### 5.0.0

[5.0 on GitHub](https://github.com/ipython/ipyparallel/milestones/5.0)

The highlight of ipyparallel 5.0 is that the Client has been reorganized a bit to use Futures.
AsyncResults are now a Future subclass, so they can be `yield` ed in coroutines, etc.
Views have also received an Executor interface.
This rewrite better connects results to their handles,
so the Client.results cache should no longer grow unbounded.

```{seealso}

- The Executor API {class}`ipyparallel.ViewExecutor`
- Creating an Executor from a Client: {meth}`ipyparallel.Client.executor`
- Each View has an {attr}`executor` attribute
```

Part of the Future refactor is that Client IO is now handled in a background thread,
which means that {meth}`Client.spin_thread` is obsolete and deprecated.

Other changes:

- Add {command}`ipcluster nbextension enable|disable` to toggle the clusters tab in Jupyter notebook

Less interesting development changes for users:

Some IPython-parallel extensions to the IPython kernel have been moved to the ipyparallel package:

- {mod}`ipykernel.datapub` is now {mod}`ipyparallel.datapub`
- ipykernel Python serialization is now in {mod}`ipyparallel.serialize`
- apply_request message handling is implememented in a Kernel subclass,
  rather than the base ipykernel Kernel.

## 4.1

[4.1 on GitHub](https://github.com/ipython/ipyparallel/milestones/4.1)

- Add {meth}`.Client.wait_interactive`
- Improvements for specifying engines with SSH launcher.

## 4.0

[4.0 on GitHub](https://github.com/ipython/ipyparallel/milestones/4.0)

First release of `ipyparallel` as a standalone package.

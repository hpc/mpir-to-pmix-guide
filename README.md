# Resources for Tools Transitioning from MPIR to PMIx

The [MPIR Process Acquisition Interface](https://www.mpi-forum.org/docs/) is an [MPI Forum](https://www.mpi-forum.org/) defined interface for debuggers to interact with MPI programs. The [PMIx Standard](https://pmix.github.io/) contains an alternative and more extensible tool interface.

Some PMIx-enabled launchers do not support the MPIR interface, which can be problematic for tools that have not moved from MPIR to PMIx. This project is targeted at those tools as they make their transition. The MPIR to PMIx project has the following goals:
 * Provide links to resources that help tools transition from MPIR to the PMIx tool interface,
 * Maintain a MPIR Shim written in C (to match the language of OpenPMIx) that provides most of the MPIR interface backed by the PMIx tool interface, and
 * Maintain other example software that can aid tools transitioning from MPIR to the PMIx tool interface.

The MPIR Shim code in this repository is meant to be an example for tools to reference when transitioning to PMIx. As such the software attempts to be structured for readability and reference. It is a fully functional MPIR Shim (see usability notes below) with a rudimentary shared library.

The MPIR Shim does not provide debugging capabilities by itself. It merely provides the symbols and back end functionality to support tools that may choose to use those systems.

## Documentation

 * PMIx Standard v4 and later contain the necessary functionality for PMIx tools: https://github.com/pmix/pmix-standard/

## Building the MPIR Shim

What you will need:
 * [OpenPMIx](https://openpmix.github.io/) v4.x or later installation (though any PMIx v4 standard compliant implementation that includes the tool support can be used)

```
./configure --prefix=/path-to-install/mpir-shim --with-pmix=/path-to-openpmix-install
make
make install
```

This will create the `mpirc` binary which can be used to wrap the native launcher.

Additionally, a library is created (`libmpirshim` - both static and shared versions) that can be linked into a launcher library that wants to hide the use the shim from the user.


## Running the MPIR Shim

The MPIR Shim works in a few different modes, namely:
 * Proxy mode indirect launch
 * Non-proxy mode indirect launch
 * Attach mode

Both the _proxy_ and _non-proxy_ modes operate in what the PMIx Standard refers to as an _indirect launch_ model. This means that they rely on a third-party launcher to start the application. This is in contrast to the _direct launch_ model in which the `mpirc` tool would call `PMIx_Spawn` directly to launch the application without the assistance of a third-party launcher. At the moment, only the _indirect launch_ model is used by the `mpirc` shim, but future versions may be extended to support the _direct launch_ model.

### Running in Proxy Mode

**Proxy Mode** : Running the MPIR Shim in a runtime environment where there are no persistent daemons.
For example, `prterun` or `mpirun` used to launch a single job. Those launchers are responsible for launching the daemons, then the application, then cleaning it all up when the job is finished.

Example of MPIR Shim being used without a debugger attached:
```
mpirc mpirun -np 2 ./a.out
```

Legacy tools may want to just prefix the `mpirun` command with `mpirc` in their normal launch model. In the example below we use a fake tool called `mytool` for illustration purposes only.

```
# Launching without the MPIR Shim
mytool -- mpirun -n 8 ./a.out
# Launching with the MPIR Shim
mytool -- mpirc mpirun -n 8 ./a.out
```

### Running in Non-Proxy Mode

**Non-Proxy Mode** : Running the MPIR Shim in a runtime environment with a persistent daemon.
For example, if you started the PRRTE `prte` process to setup a persistent Distributed Virtual Machine (DVM) environment then use `prun` to launch jobs against the DVM environment.

```
prte --daemonize
mpirc -n prun -np 2 ./a.out
pterm
```

### Running in Attach Mode

**Attach Mode** : Running the MPIR Shim as a front end to attach to a running job by referencing the PMIx server by its PID.

Launch your application (possibly without the MPIR shim)
```
mpirun -np 2 ./a.out
```

Later, attach to the running job by using the PID of `mpirun` (using `1234` for illustration below):
```
mpirc -c 1234
# Or directly with a debugger
mytool -- mpirc -c 1234
```

The MPIR Shim will extract the `MPIR_proctable` and idle until the application terminates. You can then use a parallel debugger to connect to the `mpirc` process to read the process table and attach to the remote processes.

Note that Attach Mode assumes a Proxy Mode launch at this time. It may not work with the Non-Proxy mode.

## Using the MPIR Shim Module With Debuggers

The MPIR Shim module can be used with debuggers that are debugging applications in Proxy Mode or Attach Mode. Both modes will be demonstrated using
gdb and Open MPI below. Assume the MPIR shim module has been built and installed in /u/test/mpir.

### Using the MPIR Shim Module for Debugging in Proxy Mode

Use the debugger to launch the MPIR Shim tool

~~~
gdb /u/test/mpir/bin/mpirc
~~~

Set a breakpoint at ``MPIR_Breakpoint``. This breakpoint will be hit once the ``mpirun`` process has built the MPI process table and it is available
for use by a debugger. 

Once the breakpoint has been set, have gdb run the ``mpirun`` launcher, which starts the application under the control of the MPIR shim module. 

The operands to the gdb ``run`` command are as if the ``mpirun`` command was issued at a shell prompt. Any ``mpirun`` options are allowed as well as any parameters for the application program.

```
(gdb) b MPIR_Breakpoint
(gdb) run mpirun -n 2 /u/test/mpi/bin/hello
```

The launch will run until the MPIR process table in built. When the breakpoint at ``MPIR_Breakpoint`` is hit, you can display MPIR process table information.

```
Thread 1 "mpirc" hit Breakpoint 1, MPIR_Breakpoint () at mpirshim.c:219
219         MPIR_SHIM_DEBUG_ENTER("");
Missing separate debuginfos, use: yum debuginfo-install libevent-2.1.8-5.el8.ppc64le libgcc-8.4.1-1.el8.ppc64le nvidia-driver-cuda-libs-450.142.00-1.el8.ppc64le openssl-libs-1.1.1g-15.el8_3.ppc64le sssd-client-2.4.0-9.el8_4.2.ppc64le zlib-1.2.11-17.el8.ppc64le
(gdb) p MPIR_proctable_size
$1 = 2
(gdb) p MPIR_proctable[0]
$2 = {
  host_name = 0x20002400a0f0 "c656f7n05",
  executable_name = 0x100310e0 "/u/test/mpi/bin/hello",
  pid = 1501610
}
(gdb) p MPIR_proctable[1]
$3 = {
  host_name = 0x100310c0 "c656f7n05",
  executable_name = 0x100da940 "/u/test/mpi/bin/hello",
  pid = 1501611
}
```

Execution of mpirun and the target application continues after you enter the gdb **continue** command.

### Using the MPIR Shim Module for Debugging in Attach Mode

Launch the application program and note the process ID (PID) of the ``mpirun`` process. The application should run 
long enough that you can start a debugger and attach to the ``mpirun`` process.

```
c656f7n05:test> mpirun -n 2 hello &
[1]     1501960

```

Invoke gdb to run the MPIR shim module

```
gdb /u/test/mpir/bin/mpirc
```

Set a breakpoint at ``MPIR_Breakpoint`` then run the shim module, specifying the PID of the ``mpirun`` process.
```
(gdb) b MPIR_Breakpoint
(gdb) run mpirun -c 1501960
```

The launch will run until the MPIR process table in built. When the breakpoint at ``MPIR_Breakpoint`` is hit, you can display MPIR process table information as shown in the description of the ``mpirun`` launch case above.

## Support

If you have questions or need help post a GitHub issue.

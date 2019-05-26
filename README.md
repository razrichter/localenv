# Localenv

LocalEnv is a way to simply set up working environment directories, inspired by python's virtualenv and perl's local::lib, but generalized for other, often compiled additions.

Localenv wraps [conda](https://conda.io/en/latest/), [perlbrew](https://perlbrew.pl/), and a lightly modified version of [graft](http://peters.gormand.com.au/Home/tools/graft/) (a lightweight package manager that doesn't get enough love).

This means that, in one command, you can set up an environment with specific python and perl versions and libraries, and arbitrary packages.

For historical reasons, `localenv ...` commands that affect the current shell work correctly with tcsh, if you use the syntax

    eval `localenv <command> -c ...`

instead of the examples listed below. I can't recommend using cshells, though.

## For users of an already set up localenv

### Set up the base localenv environment variables in the current shell

    eval $(<localenv_root_dir>/bin/localenv init)

`localenv init` prints out the minimal environment variable settings to use localenv and its sub-modules (conda, perlbrew, and localenv-graft).

As an aside, localenv is (deliberately) not built as shell functions, so it will not run in the current shell process. By `eval`-ing it, it will set the environment variables correctly in your current shell. If you don't eval, you can see exactly what it *would* do if you did `eval` it. If you want to create a shell function that does the `eval` for you, I suggest

    le() {
        eval $(localenv "$@")
    }

which is what I do.

`localenv init` is not technically necessary, since `localenv activate` and `use` will `init` if it was not already done. On the other hand, init will warn you if it's overriding variables that would interfere with it correctly working (notably `$PERL5LIB`, `$PYTHON_PATH`, and the PERLBREW* and CONDA* variables), so, if you `init` first, you won't see the warning when you run it later.

If you do not want to `init` localenv (say, because you already have conda or perlbrew installed and don't want to mess that up), you will want to put the localenv command in your path:

    export PATH=<localenv_root_dir>/bin:${PATH:+:$PATH}

If you want to see the active environment (e.g to put it in your prompt), it's stored in `$LOCALENV_ACTIVE`.

### See what environments are defined

    localenv list

### Run a single command in an environment

    localenv use <environment> <command> ...

The `localenv use` command loads an environment and runs the command immediately, exiting the environment when it's done (just like any other command). Like ssh, if you don't specify a command, it will start a shell in that environment; use `^D` to exit the subshell.

### Use an enviroment in your current shell

    eval $(localenv activate <environment>)

`localenv activate` prints out all the environment variable settings necessary to use the virtual environment.

### Deactivate an environment

    eval $(localenv deactivate)

`localenv deactivate` prints out the environment variable settings with all the virtual environment variables removed. Essentially, this returns the environment to what it would have been just after `virtualenv init`

### Clear out all of localenv

    eval $(localenv off)

`localenv off` unsets *all* of localenv's environment variables. Sadly, it doesn't restore variables that were overridden before, so don't expect your environment to be back to exactly the way it was before you `init`-ed localenv. Start a new terminal, if that's what you want.

## Environment Maintenance

### Setting up a new installation

    bin/localenv-bootstrap <root_dir>

This will create a brand new root for localenv, copying in the code binaries, as well as installing conda and perlbrew. It's likely to take a while, since perlbrew will compile and install perl 5.24.1. Eventually, it will install the newest stable version of perl.

Once it's installed, you can `init` the new copy of localenv.

    eval $(<root_dir>/bin/localenv init)

### Create a new environment

    localenv-create <env_name> [<python_major_version>] [<perl_version>]

This creates and initializes a new environment. If you need python version 2, for \<python_major_version>, enter "2", otherwise, leave blank or enter "3". If you need to specify a non-default perl version, you will also need to specify the python version. \[You'll also need to `perlbrew install \<perl_version> first, if you haven't already].

### Remove an environment

    localenv-delete <env_name>

Removes (without confirmation) all of the sub-environments. `deactivate` the environment if you're in the one you want to delete. Bad things would happen if the script didn't try to prevent you.

## Adding/removing packages in an environment

First, `activate` the environment you want to work with.

    eval `localenv activate <env>`

For anaconda or perlbrew, you can simply run the command as usual.

For compiled programs, you'll need to do a bit more work. `localenv-graft` requires subdirectories under `$LOCALENV_ROOT/packages` (generally named by the packages and version -- e.g. `packages/bwa-0.7.12/`) to install from, and mirrors the tree under a subdirectory of packages into the environment ($LOCALENV_ROOT/envs/<env_name>) tree. This means that you *need* to follow the linux standard directory structure -- e.g executables in `bin/`, libraries in  `lib/`, etc. See "Building and installing..." below for details and tips for doing this.

Once the software is in place in the packages/ tree, install it into the environment.

    localenv-graft -i <package_and_version>

If you need to remove a package, run

    localenv-graft -d <package_and_version>

## Building and installing compiled/boutique packages

If it's a typical `autoconf` or `cmake` based install, set the install prefix to `$LOCALENV_ROOT/packages/<package_and_version>` -- e.g. `.../packages/bwa-0.7.12`. They should then install to the right places.

If it's not -- and most research software isn't -- I suggest putting all the mixed programs and other files into a path `$LOCALENV_ROOT/packages/<package_and_version>/share/<package>`, and then creating wrapper scripts into `$LOCALENV_ROOT/packages/<package_and_version>/bin`. My template for this looks like

    #!/bin/sh
    exec ../share/<package>/$(basename $0) "$@"

This is working on the assumption that you want the program as run to have *exactly* the same name as the program in the share directory.

Sometimes you'll need to modify this (e.g. if the package has executables in a `bin/` subdirectory, but assumes that all its working files are in the directory above it). Just change the path to the executable as needed.

If it's a java jar, do similarly; put the jar(s) in the `share/<package>` directory, then create a wrapper script in `bin` like

    #!/bin/sh
    java_args="-Xmx2g"
    if [ "$1" = "--java_args" ]; then
        java_args="$2"
        shift 2
    fi
    exec java "$java_args" -jar ../share/<package>/`basename $0`.jar "$@"

with the wrapper named the same as the jar file (minus .jar). I find altogether too often that I have to fiddle the memory requirements based on inputs, so that's why that's there.

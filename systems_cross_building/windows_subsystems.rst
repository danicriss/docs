.. _windows_subsystems:

Windows Subsystems
==================

On Windows, you can run different `subsystems` that enhance the operating system with UNIX capabilities.

Conan supports ``MSYS2``, ``CYGWIN``, ``WSL`` and in general any subsystem that is able to run a ``bash``
shell.

Many libraries use these subsystems in order to use the Unix tools like the ``Autoconf`` suite
that generates ``Makefiles``.

The difference between MSYS2 and CYGWIN is that MSYS2 is oriented to the development of native Windows
packages, while CYGWIN tries to provide a complete POSIX-like system to run any Unix application on it.

For that reason, we recommend the use of MSYS2 as a subsystem to be used with Conan.


Operation Modes
---------------

The ``MSYS2`` and ``CYGWIN`` can be used with different operation modes:

- You can use them together with  MinGW to build Windows-native software.
- You can use them together with any other compiler to build Windows-native software, even with Visual
  Studio.
- You can use them with MinGW to build specific software for the subsystem, with a dependency to a
  runtime DLL (``msys-2.0.dll`` and ``cygwin1.dll``)


If you are building specific software for the subsystem, you have to specify a value for the setting ``os.subsystem``,
if you are only using the subsystem for taking benefit of the UNIX tools but generating native Windows software, you
shouldn't specify it.


Running commands inside the subsystem
-------------------------------------


.. _conanfile_win_bash:

self.win_bash
_____________

This is an experimental feature introduced in Conan 1.39. It supersedes the ``run(..., win_bash=True)`` argument but
if the ``run(..., win_bash=True)`` is used, it will have priority so the compatibility with the previous behavior is
guaranteed.

The ``self.win_bash`` is an attribute of the conanfile, when set to ``True`` and only when running in Windows (you don't need
to check if you are in Windows), it will run the ``self.run()`` commands inside a bash shell.


.. note::

    The ``bash.exe`` that will run **is not auto-detected** or read from the ``CONAN_BASH_PATH`` anymore, neither the subsystem to be used.
    These are the config variables used:

    - ``tools.microsoft.bash:subsystem``: Values can be ``msys2``, ``cygwin``, ``msys`` and ``wsl``.
    - ``tools.microsoft.bash:path``: Path to the ``bash.exe``
    - ``tools.microsoft.bash:active``: Define if Conan is already running inside the specific subsystem.


The new :ref:`Autotools, AutotoolsToolchain, AutotoolsDeps and PkgConfigDeps<conan_tools_gnu>` will work automatically
when ``self.win_bash`` is set.


self.win_bash_run
_________________

This is identical to the above ``self.win_bash``, but applies to execution of commands in the "run" scope,
so ``self.run(cmd, scope="run")`` will run such ``<cmd>`` inside a bash shell if ``win_bash_run==True``.


self.run()
__________

In a Conan recipe, you can use the ``self.run`` method specifying the parameter ``win_bash=True``
that will call automatically to the tool :ref:`tools.run_in_windows_bash<tools_run_in_windows_bash>`.

It will use the **bash** in the path or the **bash** specified for the environment variable :ref:`CONAN_BASH_PATH<conan_bash_path_env>`
to run the specified command.

Conan will automatically escape the command to match the detected subsystem.
If you also specify the ``msys_mingw`` parameter to False, and the subsystem is ``MSYS2`` it will
run in Windows-native mode, the compiler won't link against the ``msys-2.0.dll``.


AutoToolsBuildEnvironment
_________________________

.. note::

    From Conan 1.39 the new :ref:`Autotools<conan_tools_gnu_build_helper>` build helper will use the ``self.win_bash``
    conanfile attribute (see above) to adjust automatically all the paths to the subsystem.

In the constructor of the build helper, you have the ``win_bash`` parameter. Set it to ``True`` to
run the ``configure`` and ``make`` commands inside a bash.


Controlling the build environment
---------------------------------

Building software in a Windows subsystem for a different compiler than MinGW can sometimes be painful.
The reason is how the subsystem finds your compiler/tools in your system.

For example, the `icu <http://site.icu-project.org>`_ library requires Visual Studio to be built in Windows, but also a subsystem
able to build the Makefile. A very common problem and example of the pain is the ``link.exe`` program.
In the Visual Studio suite, ``link.exe`` is the linker, but in the ``MSYS2`` environment the ``link.exe``
is a tool to manage symbolic links.

Conan is able to prioritize the tools when you use ``tool_requires``, and put the tools in the PATH in
the right order.

There are some packages you can use as ``tool_requires``:

- From ConanCenter:

    - **mingw-w64/8.1**: MinGW compiler installer as a Conan package.
    - **msys2/20190524@**: MSYS2 subsystem as a Conan package (Conan Center Index).
    - **cygwin_installer/2.9.0@bincrafters/stable**: Cygwin subsystem as a Conan package.

For example, create a profile and name it *msys2_mingw* with the following contents:

.. code-block:: text

    [tool_requires]
    mingw_installer/1.0@conan/stable
    msys2/20190524

    [settings]
    os_build=Windows
    os=Windows
    arch=x86_64
    arch_build=x86_64
    compiler=gcc
    compiler.version=4.9
    compiler.exception=seh
    compiler.libcxx=libstdc++11
    compiler.threads=posix
    build_type=Release

Then you can have a *conanfile.py* that can use ``self.run()`` with ``win_bash=True`` to run any
command in a bash terminal or use the ``AutoToolsBuildEnvironment`` to invoke ``configure/make``
in the ``subsystem``:

.. code-block:: python

   from conans import ConanFile
   import os


   class MyToolchainXXXConan(ConanFile):
       name = "mylib"
       version = "0.1"
       ...

       def build(self):
           self.run("some_command", win_bash=True)

           env_build = AutoToolsBuildEnvironment(self, win_bash=True)
           env_build.configure()
           env_build.make()

        ...

Apply the profile in your recipe to create a package using the MSYS2 and MINGW:

.. code-block:: bash

    $ conan create . user/testing --profile msys2_mingw

As we included in the profile the ``MinGW`` and then the ``MSYS2`` build_require, when we run a command, the PATH
will contain first the MinGW tools and finally the MSYS2.

What could we do with the Visual Studio issue with ``link.exe``? You can pass an additional parameter to ``run_in_windows_bash``
with a dictionary of environment variables to have more priority than the others:

.. code-block:: python

    def build(self):
        # ...
        vs_path = tools.vcvars_dict(self)["PATH"] # Extract the path from the vcvars_dict tool
        tools.run_in_windows_bash(self, command, env={"PATH": vs_path})

So you will get first the ``link.exe`` from the Visual Studio.

Also, Conan has a tool ``tools.remove_from_path`` that you can use in a recipe to temporarily remove a
tool from the path if you know that it can interfere with your build script:

.. code-block:: python

   class MyToolchainXXXConan(ConanFile):
       name = "mylib"
       version = "0.1"
       ...

       def build(self):
           with tools.remove_from_path("link"):
               # Call something
               self.run("some_command", win_bash=True)

        ...

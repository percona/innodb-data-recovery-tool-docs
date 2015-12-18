.. _building_the_tools:

==================
Building the Tools
==================

To build the tools, you will need a working build environment (*C* compiler, *make*, and so on). Download the `source tarbarll <https://launchpad.net/percona-data-recovery-tool-for-innodb/trunk/release-0.5/+download/percona-data-recovery-tool-for-innodb-0.5.tar.gz>`_ from `Launchpad <https://launchpad.net/percona-data-recovery-tool-for-innodb>`_, extract it, and change to the :file:`mysql-source` directory. Run ``./configure``, but do not run ``make``.

After the configure step finishes, change to the top-level directory extracted from the tarball, and run ``make``. This will compile the ``page_parser`` and the ``constraints_parser`` tool. We are not ready to use ``constraints_parser`` yet, and we will recompile it later, after creating table definitions. In the meantime it is useful to have the ``page_parser`` tool available to use.

Internally, the tool uses some of |InnoDB|'s low-level routines to work with pages and row structures. This is why a minimal version of the |MySQL| source code is included with the tool's source code. If you have any troubles compiling, it is likely that they are related to building the |MySQL| source code. If this is blocking your progress, you can obtain a copy of the |MySQL| source code from the |MySQL| website, and compile |MySQL| separately. Then you can copy the :file:`innobase/ut/libut.a` file to the toolkit's build directory, and try running *make* again.

On *Linux*, you may get a compilation error saying there is no ``isnumber() function in check_data.c``. The problem is that ``isnumber()`` is a *BSD* standard, but the *C* standard defines the ``isdigit()`` function instead. Just change the name, and you should be able to compile successfully.

If you'd like to build static versions of binaries, uncomment the ``CFLAGS=-static`` line in the :file:`Makefile`. This can be useful to create portable binaries.

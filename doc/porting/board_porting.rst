.. _board_porting_guide:

Board Porting Guide
###################

When building an application you must specify the target hardware and
the exact board or model. Specifying the board name results in a binary that
is suited for the target hardware by selecting the right Zephyr features and
components and setting the right Zephyr configuration for that specific target
hardware.

A board is defined as a special configuration of an SoC with possible additional
components.
For example, a board might have sensors and flash memory implemented as
additional features on top of what the SoC provides. Such additional hardware is
configured and referenced in the Zephyr board configuration.

The board implements at least one SoC and thus inherits all of the features
that are provided by the SoC. When porting a board to Zephyr, you should
first make sure the SoC is implemented in Zephyr.

Hardware Configuration Hierarchy
********************************

Hardware definitions in Zephyr follow a well-defined hierarchy of configurations
and layers, below are the layers from top to bottom:

- Board
- SoC
- SoC Series
- SoC Family
- CPU Core
- Architecture

This design contributes to code reuse and implementation of device drivers and
features at the bottom of the hierarchy making a board configuration as simple
as a selection of features that are implemented by the underlying layers. The
figures below shows this hierarchy with a few example of boards currently
available in the source tree:

.. figure:: board/hierarchy.png
   :width: 500px
   :align: center
   :alt: Configuration Hierarchy

   Configuration Hierarchy


Hierarchy Example

+------------+-----------+--------------+------------+--------------+---------+
|Board       |FRDM K64F  |nRF52 NITROGEN|nRF51XX     |Quark SE C1000|Arduino  |
|            |           |              |            |Devboard      |101      |
+============+===========+==============+============+==============+=========+
|SOC         |MK64F12    |nRF52832      |nRF51XX     |Quark SE C1000|Curie    |
+------------+-----------+--------------+------------+--------------+---------+
|SOC Series  |Kinetis K6x|Nordic NRF52  |Nordic NRF51|Quark SE      |Quark SE |
|            |Series     |              |            |              |         |
+------------+-----------+--------------+------------+--------------+---------+
|SOC Family  |NXP Kinetis|Nordic NRF5   |Nordic NRF5 |Quark         |Quark    |
+------------+-----------+--------------+------------+--------------+---------+
|CPU Core    |Cortex-M4  |Cortex-M4     |Cortex-M0+  |Lakemont      |Lakemont |
+------------+-----------+--------------+------------+--------------+---------+
|Architecture|ARM        |ARM           |ARM         |x86           |x86      |
+------------+-----------+--------------+------------+--------------+---------+


Architecture
============
If your CPU architecture is already supported by Zephyr, there is no
architecture work involved in porting to your board.  If your CPU architecture
is not supported by the Zephyr kernel, you can add support by following the
instructions available at :ref:`architecture_porting_guide`.

CPU Core
========

Some OS code depends on the CPU core that your board is using. For
example, a given CPU core has a specific assembly language instruction set, and
may require special cross compiler or compiler settings to use the appropriate
instruction set.

If your CPU architecture is already supported by Zephyr, there is no CPU core
work involved in porting to your platform or board. You need only to select the
appropriate CPU in your configuration and the rest will be taken care of by the
configuration system in Zephyr which will select the features implemented
by the corresponding CPU.

Platform
========

This layer implements most of the features that need porting and is split into
three layers to allow for code reuse when dealing with implementations with
slight differences.

SoC Family
----------

This layer is a container of all SoCs of the same class that, for example
implement one single type of CPU core but differ in peripherals and features.
The base hardware will in most cases be the same across all SoCs and MCUs of
this family.

SoC Series
----------

Moving closer to the SoC, the series is derived from an SoC family. A series is
defined by a feature set that serves the purpose of distinguishing different
SoCs belonging to the same family.

SoC
---

Finally, an SoC is actual hardware component that is physically available on a
board.

Board
=====

A board implements an SoC with all its features, together with peripherals
available on the board that differentiates the board with additional interfaces
and features not available in the SoC.

Default board configuration
***************************

When porting Zephyr to a board, you must provide the board's default
Kconfig configuration, which is used in application builds unless explicitly
overridden.

In order to provide consistency across the various boards and ease the work of
users providing applications that are not board specific, the following
guidelines should be followed when porting a board:

- Provide pin and driver configuration that matches the board's valuable
  components such as sensors, buttons or LEDs, and communication interfaces
  such as USB, Ethernet connector, or Bluetooth/WiFi chip.

- When a well-known connector is present (such as used on an Arduino or
  96board), configure pins to fit this connector.

- Configure components that enable the use of these pins, such as
  configuring and SPI instance for Arduino SPI.

- Configure an output for the console.

- Propose and configure a default network interface.

- Enable all GPIO ports.

.. _setting_configuration_values:

Setting configuration values
============================

Kconfig symbols can be set to their ``BOARD``-specific values in one of two
ways. The right method to use depends on whether the symbol is *visible* or
not.


Visible and invisible Kconfig symbols
-------------------------------------

Kconfig symbols come in two varieties:

- A Kconfig symbol defined with a prompt is *visible*, and can be configured from
  the ``menuconfig`` configuration interface.

- A Kconfig symbol defined without a prompt is *invisible*. The user has no
  direct control over its value.

Here are some examples of visible and invisible symbols:

.. code-block:: none

    config NOT_VISIBLE
    	bool
    	default FOO

    config VISIBLE_1
    	bool "Enable stuff"

    config VISIBLE_2
    	string
    	prompt "Foo value"


Configuring visible Kconfig symbols
-----------------------------------

Default ``BOARD``-specific configuration values for visible Kconfig symbols
*should* be given in :file:`boards/ARCHITECTURE/BOARD/BOARD_defconfig`, which
uses the standard Kconfig :file:`.config` file syntax.


Configuring invisible Kconfig symbols
-------------------------------------

``BOARD``-specific configuration values for invisible Kconfig symbols *must* be
given in :file:`boards/ARCHITECTURE/BOARD/Kconfig.defconfig`, which uses
Kconfig syntax.

.. note::

    Assignments in :file:`.config` files have no effect on invisible symbols,
    so this scheme is not just an organizational issue.

Assigning values in :file:`Kconfig.defconfig` relies on being able to define a
Kconfig symbol in multiple locations. As an example, say we want to set
``FOO_WIDTH`` below to 32:

.. code-block:: none

    config FOO_WIDTH
    	int

To do this, we extend the definition of ``FOO_WIDTH`` as follows, in
:file:`Kconfig.defconfig`:

.. code-block:: none

    if BOARD_MY_BOARD

    config FOO_WIDTH
    	default 32

    endif

.. note::

    Since the type of the symbol (``int``) has already been given at the first
    definition location, it does not need to be repeated here.

``default`` properties from :file:`Kconfig.defconfig` files override
``default`` properties given on the "base" definition of the symbol. Zephyr
uses a custom Kconfig patch that makes Kconfig prefer later defaults, and
includes any :file:`Kconfig.defconfig` file last. See the
:ref:`zephyr-specific_kconfig_behavior_for_defaults` section.

If you want a symbol to only be user-configurable on some boards, make its base
definition have no prompt, and then add a prompt to it in the
:file:`Kconfig.defconfig` files of the boards where it should be configurable.

.. note::

    Prompts added in :file:`Kconfig.defconfig` files show up at the location of
    the :file:`Kconfig.defconfig` file in the ``menuconfig`` interface, rather
    than at the location of the base definition of the symbol.

Motivation
----------

One motivation for this scheme is to avoid making fixed ``BOARD``-specific
settings configurable in the ``menuconfig`` interface. If all
configuration were done via :file:`boards/ARCHITECTURE/BOARD/BOARD_defconfig`,
all symbols would have to be visible, as values given in
:file:`boards/ARCHITECTURE/BOARD/BOARD_defconfig` have no effect on invisible
symbols.

Having fixed settings be user-configurable might be confusing, and would allow
the user to create broken configurations.

.. _kconfig_extensions_and_changes:

Kconfig extensions and changes
==============================

Zephyr uses the `Kconfiglib <https://github.com/ulfalizer/Kconfiglib>`_
implementation of `Kconfig
<https://www.kernel.org/doc/Documentation/kbuild/kconfig-language.txt>`_. It
simplifies how environment variables are handled, and adds some extensions.

Environment variables in ``source`` statements are expanded directly in
Kconfiglib, meaning no ``option env="ENV_VAR"`` "bounce" symbols need to be
defined. If you need compatibility with the C Kconfig tools for an out-of-tree
Kconfig tree, you can still add such symbols, but they must have the same name
as the corresponding environment variables.

.. note::

    As of writing, there are plans to remove ``option env`` from the C tools as
    well.

The following Kconfig extensions are available:

- The ``gsource`` statement, which includes each file matching a given wildcard
  pattern.

  Consider the following example:

  .. code-block:: none

      gsource "foo/bar/*/Kconfig"

  If the pattern ``foo/bar/*/Kconfig`` matches the files
  :file:`foo/bar/baz/Kconfig` and :file:`foo/bar/qaz/Kconfig`, the statement
  above is equivalent to the following two ``source`` statements:

  .. code-block:: none

      source "foo/bar/baz/Kconfig"
      source "foo/bar/qaz/Kconfig"

  .. note

      The wildcard patterns accepted are the same as for the Python `glob
      <https://docs.python.org/3/library/glob.html>`_ module.

  If no files match the pattern, ``gsource`` has no effect. This means that
  ``gsource`` also functions as an "optional" include statement (similar to
  ``-include`` in Make):

  .. code-block:: none

      gsource "foo/include-if-exists"

  .. note::

     Wildcard patterns that do not include any wildcard symbols (e.g., ``*``)
     only match exactly the filename given, and only match it if the file
     exists.

  It might help to think of *g* as standing for *generalized* rather than
  *glob* in this case.

  .. note::

      Only use ``gsource`` if you need it. Trying to ``source`` a non-existent
      file produces an error, while ``gsource`` silently ignores missing files.
      ``source`` also makes it clearer which files are being included.

- The ``rsource`` statement, which includes a file specified with a relative
  path.

  The path is relative to the directory of the :file:`Kconfig` file that has
  the ``rsource`` statement.

  As an example, assume that :file:`foo/Kconfig` is the top-level
  :file:`Kconfig` file, and that :file:`foo/bar/Kconfig` has the following
  statements:

  .. code-block:: none

      source "qaz/Kconfig1"
      rsource "qaz/Kconfig2"

  This will include the two files :file:`foo/qaz/Kconfig1` and
  :file:`foo/bar/qaz/Kconfig2`.

  ``rsource`` can be used to create :file:`Kconfig` "subtrees" that can be
  moved around freely.

- The ``grsource`` statement, which combines ``gsource`` and ``rsource``.

  For example, the following statement will include :file:`Kconfig1` and
  :file:`Kconfig2` from the current directory (if they exist):

  .. code-block:: none

      grsource "Kconfig[12]"

.. _zephyr-specific_kconfig_behavior_for_defaults:

Zephyr-specific Kconfig behavior for defaults
=============================================

Zephyr uses a Kconfig patch that gives later ``default``\ s precedence over
earlier ``default``\ s. This is a significant change from standard Kconfig
behavior, which is to pick the first ``default`` with a satisfied condition.

Consider the following example:

.. code-block:: none

    config FOO
        string
        default "first" if n
        default "second"
        default "third" if n
        default "fourth"
        default "fifth" if n

In unpatched Kconfig, this will give ``FOO`` the value ``"second"``, which is
the first ``default`` with a satisfied condition.

In Zephyr, this will give ``FOO`` the value ``"fourth"``, which is the last
``default`` with a satisfied condition.

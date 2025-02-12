.. _CMODULES-CHAPTER:

======================
C Modules Installation
======================

This section describes how to install a C module on your VEE Port, depending on your SDK version the first steps may vary.

Fetching the module source files
++++++++++++++++++++++++++++++++

.. tabs::

   .. tab:: SDK 5

      Under the SDK 5, follow these steps:

      - Go to the location of your C module, for example the C module for microUI 14.2.0 is located `here <https://repository.microej.com/modules/com/microej/clibrary/llimpl/microui/14.2.0/>`_.
      - Get the dependency declaration, for the previous example this would be: ``<dependency org="com.microej.clibrary.llimpl" name="microui" rev="14.2.0" />``.
      - Add it inside the block ``<dependencies>`` from the file ``module.ivy`` of your VEE Port ``*-configuration`` project.
      - **If you update a C module:** Remove the ``.properties`` file in the folder ``*-bsp/projects/microej`` corresponding to the desired C module. For example with microui C module, its .properties file is named ``cco_microui.properties``.
      - Rebuild the VEE Port: in the SDK 5 Project Explorer, right click on the VEE Port module ``*-configuration > build module``.

   .. tab:: SDK 6

      Under the SDK 6, follow these steps:

      - Go to the location of your C module, for example the C module for microUI 14.2.0 is located `here <https://repository.microej.com/modules/com/microej/clibrary/llimpl/microui/14.2.0/>`_.
      - Download the archive file with the ``.cco`` extension.
      - Unzip the content of this file.
      - The source files are located in the folder ``bsp/``.
      - Copy the **content** of the ``bsp/`` folder into your VEE Port at the path ``bsp/vee/port/``.

C module configuration and firmware build
+++++++++++++++++++++++++++++++++++++++++

- Update the toolchain build of your BSP (IAR, CMake, etc...) to include any new files if this is the case.
- Configure the C module if required (typically, the configuration is located in files suffixed with ``*_configuration.h``).
- Now you should be able to build your BSP firmware.

..
   | Copyright 2025, MicroEJ Corp. Content in this space is free 
   for read and redistribute. Except if otherwise stated, modification 
   is subject to MicroEJ Corp prior approval.
   | MicroEJ is a trademark of MicroEJ Corp. All other trademarks and 
   copyrights are the property of their respective owners.

.. _section_font_loader:

===========
Font Loader
===========

Principle
=========

The Font Loader is a module of the MicroUI runtime, dedicated to the :ref:`internal font format <section_font_internal_format>`, that loads font data (precomputed bitmaps of glyphs) ready to be displayed.
The font data must be stored as a resource (in RAW format).
Typically, these resources are generated by the :ref:`section_fontgen` and embedded as internal resources or loaded from external memories (:ref:`section_font_loader_memory` loader).

The loader is automatically used for all fonts retrieved through the MicroUI APIs `Font.getAllFonts()`_, `Font.getDefaultFont()`_ and `Font.getFont(String)`_.

.. _section_font_loader_memory:

External Resources
==================

Memory Management
-----------------

The Font Loader is able to load some fonts located outside the CPU addresses' space range.
It uses the External Resource Loader.

When a font is located in such memory, the font characters are loaded one by one from the External Memory.

The Font Loader uses a RAM buffer (External Font Heap) containing only the font character currently being drawn by the application. 
It is unloaded from the RAM when the Graphics Engine's software algorithms no longer need it.

The External Font Heap is stored into a RAM section called ``.bss.microui.display.externalFontsHeap``.
Its size is automatically calculated according to the external fonts used by the firmware.
This size can be checked when enabling the verbose mode when building the application executable:

   .. figure:: images/font-external-font-heap.png
      :alt: External Font Heap size in verbose mode
      :align: center

      External Font Heap size in verbose mode

However, it is possible to change this value by setting the application property ``ej.microui.memory.externalfontsheap.size``.
This option is very useful when building a kernel: the kernel may anticipate the section size required by the features.

.. warning:: When this size is smaller than the size required by an external font, some characters may be not drawn.

Also, the Font Loader copies a very short part of the resource (the font file) in RAM (into CPU address space range): the font header.
This header remains located in RAM as long as the application is using the font.
As soon as the application uses another external font, the new font replaces the old one.

Configuration File
------------------

Like internal resources, the Font Generator uses a :ref:`configuration file <section_fontgen_conffile>` (also called the "list file") for describing fonts that need to be processed.
The list file must be specified in the application launcher (see :ref:`application_options`).
However, all the files in the application classpath with the suffix ``.fontsext.list`` are automatically parsed by the Font Generator tool.

Process
-------

This chapter describes the steps to setup the loading of an external resource from the application:

1. Add the font to the application project resources (typically in the source folder ``src/main/resources`` and in the package ``fonts``).
2. Create / open the configuration file (e.g. ``application.fontsext.list``).
3. Add the relative path of the font and, at least, its output format (e.g. ``/fonts/myFont.fnt::4``, see :ref:`section.ui.Fonts`).
4. Build the application: the Font Generator converts the font in RAW format in the external resources folder (``[application_output_folder]/externalResources``).
5. Deploy the external resources to the external memory (SDCard, flash, etc.) of the device.
6. (optional) Configure the :ref:`section_externalresourceloader` to load from this source.
7. Build the application and run it on the device.
8. The application loads the external resource using `Font.getFont(String)`_.
9. The font loader looks for the font and only reads the font header.
10. (optional) The external resource is closed if the external resource is inside the CPU addresses' space range.
11. The application can use the font.
12. The external resource is never closed: the font characters are retrieved one by one from External Memory on demand (drawString, etc.).

Simulation
----------

The Simulator automatically manages the external resources like internal resources.
All fonts listed in ``*.fontsext.list`` files are copied in the external resources folder, and this folder is added to the Simulator's classpath.

Backward Compatibility
----------------------

As explained :ref:`here<section.tool.fontdesigner.styles>`, the notion of ``Dynamic`` styles and the style ``UNDERLINED`` are not supported anymore by MicroUI 3. However, an external font may have been generated with an older version of the Font Generator; consequently, the generated file can hold the ``Dynamic`` style.
The Font Loader can load these old versions of fonts.
However, there are some runtime limitations:

* The ``Dynamic`` styles are ignored.
* The font is drawn without any dynamic algorithm.
* The font style (the style returned by ``Font.isBold()`` and ``Font.isItalic()``) is the ``Dynamic`` style.
* For instance, when a font holds the style `bold` as dynamic style and the style `italic` as built-in style, the font is considered as `bold` + `italic`; even if the style `bold` is not rendered.

Installation
============

The Font Loader is part of the MicroUI module and Display module.
You must install them in order to be able to use some fonts.


Use
===

The MicroUI font APIs are available in the class
`ej.microui.display.Font`_.

.. _Font.getFont(String): https://repository.microej.com/javadoc/microej_5.x/apis/ej/microui/display/Font.html#getFont-java.lang.String-
.. _Font.getDefaultFont(): https://repository.microej.com/javadoc/microej_5.x/apis/ej/microui/display/Font.html#getDefaultFont--
.. _Font.getAllFonts(): https://repository.microej.com/javadoc/microej_5.x/apis/ej/microui/display/Font.html#getAllFonts--
.. _ej.microui.display.Font: https://repository.microej.com/javadoc/microej_5.x/apis/ej/microui/display/Font.html#

..
   | Copyright 2008-2025, MicroEJ Corp. Content in this space is free 
   for read and redistribute. Except if otherwise stated, modification 
   is subject to MicroEJ Corp prior approval.
   | MicroEJ is a trademark of MicroEJ Corp. All other trademarks and 
   copyrights are the property of their respective owners.

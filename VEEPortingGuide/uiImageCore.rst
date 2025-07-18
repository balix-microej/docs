.. _section_image_renderer:

==============
Image Renderer
==============

Principle
=========

The Image Renderer is a module of the MicroUI runtime that reads and draws the images (see :ref:`section_image_raw`).
It calls Abstraction Layer APIs to draw and transform the images (rotation, scaling, deformation, etc.).
It also includes software algorithms to perform the rendering.

Functional Description
======================

All MicroUI image drawings are redirected to a set of Abstraction Layer APIs.
All Abstraction Layer APIs are implemented by weak functions, which call software algorithms.
The BSP can override this default behavior for each Abstraction Layer API independently.
Furthermore, the BSP can override an Abstraction Layer API for a specific MicroEJ format (for instance ``ARGB8888``) and call the software algorithms for all other formats.

Destination Format
==================

Since MicroUI 3.2, the destination buffer of the drawings can be different from the display back buffer format (see :ref:`section_image_display_raw`).
This destination buffer format can be a :ref:`standard format <section_image_standard_raw>` (ARGB8888, A8, etc.) or a :ref:`custom format <section_image_custom_raw>`.

See :ref:`section_buffered_image` for more information about how to create buffered images with another format than the display format and how to draw in them.

Input Formats
=============

Standard
--------

The Image Renderer is by default able to draw all :ref:`standard formats <section_image_standard_raw>`.
No extra support in the VEE Port is required to draw this kind of image.

The image drawing resembles a :ref:`shape drawing <section_drawings>`.
The drawing is performed by default by the :ref:`section_drawings_soft` and can be overridden to use a third-party library or a GPU.

.. _section_buffered_image_drawer_custom_format:

Custom
------

A :ref:`section_image_custom_raw` image can be:

* an image with a pixel buffer but whose pixel organization is not standard,
* an image with a data buffer: an image encoded with a third-party encoder (proprietary format or not),
* an image with a "command" buffer: instead of performing the drawings on pixels, the image *stores* the drawing actions to replay them later,
* etc.

The VEE Port must extend the Image Renderer to support the drawing of these images.
This extension can consist in:

* decoding the image at runtime to draw it,
* using a compatible GPU to draw it,
* using a command interpreter to perform some :ref:`shape drawings <section_drawings>`,
* etc.

To draw the custom images, the Image Renderer introduces the notion of *custom image drawer*.
This drawer is an engine that has the responsibility to draw the image.
Each custom image format (``0`` to ``7``) has its own image drawer.

Each drawing of a custom image is redirected to the associated image drawer.

.. hint:: A custom image drawer can call the UI Shapes Drawing API to draw its elements in the destination.

The implementation is not the same between the Embedded side and the Simulation.
However, the concepts are the same and are described in dedicated chapters.

.. _section_renderer_cco:

MicroUI C Module
================

Principle
---------

As described above, an :ref:`image drawer <section_buffered_image_drawer_custom_format>` allows drawing the images whose format is *custom*.
The :ref:`MicroUI C module<section_ui_releasenotes_cmodule>` is designed to manage the notion of drawers: it does not *support* the custom formats but allows adding some additional drawers.

This support uses several weak functions and tables to redirect the image drawings.
When custom drawers are not used (when the VEE Port does not need to support *custom* images), this support can be removed to reduce the memory footprint (by removing the indirection tables) and improve the performances (by reducing the number of runtime function calls).

.. _section_buffered_image_drawer_standard:

Standard Formats Only (Default)
-------------------------------

The default implementation can only draw images with a :ref:`standard format <section_image_standard_raw>`.
In other words, the application cannot draw a custom image.
This is the most frequent use case, the only one available with MicroUI before version 3.2.

.. attention:: To select this implementation (to disable the custom format support), the define ``UI_FEATURE_IMAGE_CUSTOM_FORMATS`` must be unset.

The image drawing is similar to ``UI_DRAWING_GPU_drawLine`` (see :ref:`section_drawings_cco`), but, theoretically, it should let the image drawer handle the image instead of calling the software drawer directly.
However the MicroUI C Module (and the extended MicroUI modules that handle a GPU) takes advantage of the define ``UI_FEATURE_IMAGE_CUSTOM_FORMATS``: as it is not set, the C Modules bypass the indirection to the image drawer, and as a consequence, the implementation of the weak function only consists in calling the Graphics Engine's software algorithm. 
This tip reduces the footprint and the CPU usage.

An implementation of a third-party GPU can optionally take advantage of the define ``UI_FEATURE_IMAGE_CUSTOM_FORMATS``.
The following diagrams illustrate the drawing of an image with or without taking advantage of the define ``UI_FEATURE_IMAGE_CUSTOM_FORMATS`` (respectively *default* and *optimized* implementation).


.. tabs::

   .. tab:: Default Implementation

      .. graphviz:: :align: center

         digraph {
            ratio="auto"
            splines="true";
            bgcolor="transparent"
            node [style="filled,rounded" fontname="courier new" fontsize="10"]

            { //in/out
               node [shape="ellipse" color="#e5e9eb" fontcolor="black"] mui UID_soft_c UID_gpu_hard stub
            }
            { // h
               node [shape="box" color="#00aec7" fontcolor="white"] LLUI_h UID_h UID_soft_h UID_stub_h UII_h
            }
            { // c
               node [shape="box" color="#ee502e" fontcolor="white"] LLUI_c UID_stub_c UII_c UID_gpu_c UID_gpu_driver
            }
            { // weak
               node [shape="box" style="dashed,rounded" color="#ee502e"] UID_weak_c
            }
            { // choice
               node [shape="diamond" color="#e5e9eb"] UID_cond UID_gpu_cond UII_cond
            }

            // --- SIMPLE FLOW ELEMENTS -- //

            mui [label="[MicroUI]\nPainter.drawImage();"] 
            LLUI_h [label="[LLUI_PAINTER_impl.h]\nLLUI_PAINTER_IMPL_drawImage();"]
            LLUI_c [label="[LLUI_PAINTER_impl.c]\nLLUI_PAINTER_IMPL_drawImage();"]
            UID_h [label="[ui_drawing.h]\nUI_DRAWING_drawImage();"]
            UID_weak_c [label="[ui_drawing.c]\nweak UI_DRAWING_drawImage();"]
            UID_soft_h [label="[ui_drawing_soft.h]\nUI_DRAWING_SOFT_drawImage();"]
            UID_soft_c [label="[Graphics Engine]"]

            // --- GPU FLOW ELEMENTS -- //

            UID_cond [label="algo implemented?"]
            UID_gpu_c [label="[ui_drawing_gpu.c]\nUI_DRAWING_drawImage();"]
            UID_gpu_cond [label="GPU compatible?"]
            UID_gpu_driver [label="[GPU driver]"]
            UID_gpu_hard [label="[GPU]"]

            UID_stub_h [label="[ui_drawing_stub.h]\nUI_DRAWING_STUB_drawImage();"]
            UID_stub_c [label="[ui_drawing_stub.c]\nUI_DRAWING_STUB_drawImage();"]
            stub [label="-"]

            // --- MULTIPLE IMAGES FLOW ELEMENTS -- //

            UII_h [label="[ui_image_drawing.h]\nUI_IMAGE_DRAWING_draw();"]
            UII_c [label="[ui_image_drawing.c]\nUI_IMAGE_DRAWING_draw();"]
            UII_cond [label="standard image?"]

            // --- FLOW -- //

            mui->LLUI_h->LLUI_c->UID_h->UID_cond
            UID_cond->UID_weak_c [label="no" fontname="courier new" fontsize="10"]
            UID_weak_c->UID_soft_h [label="built-in optim" fontname="courier new" fontsize="10"]
            UID_cond->UID_gpu_c [label="yes" fontname="courier new" fontsize="10"]
            UID_gpu_c->UID_gpu_cond
            UID_gpu_cond->UII_h->UII_c->UII_cond [label="no" fontname="courier new" fontsize="10"]
            UID_gpu_cond->UID_gpu_driver [label="yes" fontname="courier new" fontsize="10"]
            UID_gpu_driver->UID_gpu_hard
            UII_cond->UID_soft_h [label="yes" fontname="courier new" fontsize="10"]
            UII_cond->UID_stub_h [label="no" fontname="courier new" fontsize="10"]
            UID_soft_h->UID_soft_c
            UID_stub_h->UID_stub_c->stub
         }

   .. tab:: Optimized Implementation

      .. graphviz:: :align: center

         digraph {
            ratio="auto"
            splines="true";
            bgcolor="transparent"
            node [style="filled,rounded" fontname="courier new" fontsize="10"]

            { //in/out
               node [shape="ellipse" color="#e5e9eb" fontcolor="black"] mui UID_soft_c UID_gpu_hard
            }
            { // h
               node [shape="box" color="#00aec7" fontcolor="white"] LLUI_h UID_h UID_soft_h
            }
            { // c
               node [shape="box" color="#ee502e" fontcolor="white"] LLUI_c UID_gpu_c UID_gpu_driver
            }
            { // weak
               node [shape="box" style="dashed,rounded" color="#ee502e"] UID_weak_c
            }
            { // choice
               node [shape="diamond" color="#e5e9eb"] UID_cond UID_gpu_cond
            }

            // --- SIMPLE FLOW ELEMENTS -- //

            mui [label="[MicroUI]\nPainter.drawImage();"] 
            LLUI_h [label="[LLUI_PAINTER_impl.h]\nLLUI_PAINTER_IMPL_drawImage();"]
            LLUI_c [label="[LLUI_PAINTER_impl.c]\nLLUI_PAINTER_IMPL_drawImage();"]
            UID_h [label="[ui_drawing.h]\nUI_DRAWING_drawImage();"]
            UID_weak_c [label="[ui_drawing.c]\nweak UI_DRAWING_drawImage();"]
            UID_soft_h [label="[ui_drawing_soft.h]\nUI_DRAWING_SOFT_drawImage();"]
            UID_soft_c [label="[Graphics Engine]"]

            // --- GPU FLOW ELEMENTS -- //

            UID_cond [label="algo implemented?"]
            UID_gpu_c [label="[ui_drawing_gpu.c]\nUI_DRAWING_drawImage();"]
            UID_gpu_cond [label="GPU compatible?"]
            UID_gpu_driver [label="[GPU driver]"]
            UID_gpu_hard [label="[GPU]"]

            // --- FLOW -- //

            mui->LLUI_h->LLUI_c->UID_h->UID_cond
            UID_cond->UID_weak_c [label="no" fontname="courier new" fontsize="10"]
            UID_weak_c->UID_soft_h [label="built-in optim" fontname="courier new" fontsize="10"]
            UID_cond->UID_gpu_c [label="yes" fontname="courier new" fontsize="10"]
            UID_gpu_c->UID_gpu_cond
            UID_gpu_cond->UID_soft_h [label="no" fontname="courier new" fontsize="10"]
            UID_gpu_cond->UID_gpu_driver [label="yes" fontname="courier new" fontsize="10"]
            UID_gpu_driver->UID_gpu_hard
            UID_soft_h->UID_soft_c
         }

**LLUI_PAINTER_IMPL_drawImage** (available in MicroUI C Module)

Similar to ``LLUI_PAINTER_IMPL_drawLine``, see :ref:`section_drawings_cco`.

**UI_DRAWING_drawImage**

.. code-block:: c

   // Available in MicroUI C Module
   #define UI_DRAWING_DEFAULT_drawImage UI_DRAWING_drawImage

   // To write in the BSP (optional)
   #define UI_DRAWING_GPU_drawImage UI_DRAWING_drawImage

The function names are set with preprocessor macros.
These name redirections are helpful when the VEE Port features more than one destination format (which is not the case here).

**UI_DRAWING_GPU_drawImage** (to write in the BSP)

Similar to ``UI_DRAWING_GPU_drawLine`` (see :ref:`section_drawings_cco`), but lets the image drawer manage the image instead of calling the software drawer directly (*Default Implementation*) or takes advantage of the define ``UI_FEATURE_IMAGE_CUSTOM_FORMATS`` (*Optimized Implementation*):


.. tabs::

   .. tab:: Default Implementation

      .. code-block:: c

         // Unlike the MicroUI C Module, this function is not "weak".
         DRAWING_Status UI_DRAWING_GPU_drawImage(MICROUI_GraphicsContext* gc, MICROUI_Image* img, jint regionX, jint regionY, jint width, jint height, jint x, jint y, jint alpha) {
            
            DRAWING_Status status;

            if (is_gpu_compatible(xxx)) {
               
               // See chapter "Drawings"
               // [...]
            }
            else {
               // Let the image drawer manages the image (available in the C module)
               status = UI_IMAGE_DRAWING_draw(gc, img, regionX, regionY, width, height, x, y, alpha);
            }
            return status;
         }

   .. tab:: Optimized Implementation

      .. code-block:: c

         // Unlike the MicroUI C Module, this function is not "weak".
         DRAWING_Status UI_DRAWING_GPU_drawImage(MICROUI_GraphicsContext* gc, MICROUI_Image* img, jint regionX, jint regionY, jint width, jint height, jint x, jint y, jint alpha) {
            
            DRAWING_Status status;

            if (is_gpu_compatible(xxx)) {
               
               // See chapter "Drawings"
               // [...]
            }
            else {
         #if !defined(UI_FEATURE_IMAGE_CUSTOM_FORMATS)
               status = UI_DRAWING_SOFT_drawImage(gc, img, regionX, regionY, width, height, x, y, alpha);
         #else
               // Let the image drawer manages the image (available in the C module)
               status = UI_IMAGE_DRAWING_draw(gc, img, regionX, regionY, width, height, x, y, alpha);
         #endif
            }
            return status;
         }



**UI_DRAWING_DEFAULT_drawImage** (available in MicroUI C Module)

.. code-block:: c

   // Use the compiler's 'weak' attribute
   __weak DRAWING_Status UI_DRAWING_DEFAULT_drawImage(MICROUI_GraphicsContext* gc, MICROUI_Image* img, jint regionX, jint regionY, jint width, jint height, jint x, jint y, jint alpha) {
   #if !defined(UI_FEATURE_IMAGE_CUSTOM_FORMATS)
      return UI_DRAWING_SOFT_drawImage(gc, img, regionX, regionY, width, height, x, y, alpha);
   #else
      return UI_IMAGE_DRAWING_draw(gc, img, regionX, regionY, width, height, x, y, alpha);
   #endif
   }

The define ``UI_FEATURE_IMAGE_CUSTOM_FORMATS`` is not set, so the implementation of the weak function only consists in calling the Graphics Engine's software algorithm.

.. _section_buffered_image_drawer_custom:

Custom Format Support 
---------------------

In addition to the :ref:`standard formats <section_image_standard_raw>`, this implementation allows drawing images with a :ref:`custom format <section_image_custom_raw>`.
This advanced use case is available only with MicroUI 3.2 or higher.

.. attention:: To select this implementation, the define ``UI_FEATURE_IMAGE_CUSTOM_FORMATS`` must be set (no specific value).

The MicroUI C module uses some tables to redirect the image management to the expected extension.
There is one table per Image Abstraction Layer API (draw, copy, region, rotate, scale, flip) to embed only the necessary algorithms (a table and its functions are only embedded in the final binary file if and only if the MicroUI drawing method is called).

Each table contains ten elements:

.. code:: c

   static const UI_IMAGE_DRAWING_draw_t UI_IMAGE_DRAWING_draw_custom[] = {
         &UI_DRAWING_STUB_drawImage,
         &UI_DRAWING_SOFT_drawImage,
         &UI_IMAGE_DRAWING_draw_custom0,
         &UI_IMAGE_DRAWING_draw_custom1,
         &UI_IMAGE_DRAWING_draw_custom2,
         &UI_IMAGE_DRAWING_draw_custom3,
         &UI_IMAGE_DRAWING_draw_custom4,
         &UI_IMAGE_DRAWING_draw_custom5,
         &UI_IMAGE_DRAWING_draw_custom6,
         &UI_IMAGE_DRAWING_draw_custom7,
   };

* ``UI_DRAWING_STUB_drawImage`` is the drawing function called when the drawing function is not implemented,
* ``UI_DRAWING_SOFT_drawImage`` is the drawing function that redirects the drawing to the :ref:`section_drawings_soft`,
* ``UI_IMAGE_DRAWING_draw_customX`` (``0`` to ``7``) are the drawing functions for each custom format.

The MicroUI C Module retrieves the table index according to the image format.

The following diagram illustrates the drawing of an image:


.. graphviz:: :align: center

   digraph {
      ratio="auto"
      splines="true";
      bgcolor="transparent"
      node [style="filled,rounded" fontname="courier new" fontsize="10"]

      { //in/out
         node [shape="ellipse" color="#e5e9eb" fontcolor="black"] mui UID_soft_c UID_gpu_hard stub UIIx_impl_d
      }
      { // h
         node [shape="box" color="#00aec7" fontcolor="white"] LLUI_h UID_h UID_soft_h UID_stub_h UII_h UID_h2
      }
      { // c
         node [shape="box" color="#ee502e" fontcolor="white"] LLUI_c UID_gpu_c UID_stub_c UII_c UIIx_c UIIx_impl_c UID_gpu_driver
      }
      { // weak
         node [shape="box" style="dashed,rounded" color="#ee502e"] UID_weak_c UIIx_weak_c
      }
      { // choice
         node [shape="diamond" color="#e5e9eb"] UID_cond UID_gpu_cond UII_cond UIIx_cond
      }

      // --- SIMPLE FLOW ELEMENTS -- //

      mui [label="[MicroUI]\nPainter.drawImage();"] 
      LLUI_h [label="[LLUI_PAINTER_impl.h]\nLLUI_PAINTER_IMPL_drawImage();"]
      LLUI_c [label="[LLUI_PAINTER_impl.c]\nLLUI_PAINTER_IMPL_drawImage();"]
      UID_h [label="[ui_drawing.h]\nUI_DRAWING_drawImage();"]
      UID_weak_c [label="[ui_drawing.c]\nweak UI_DRAWING_drawImage();"]
      UID_soft_h [label="[ui_drawing_soft.h]\nUI_DRAWING_SOFT_drawImage();"]
      UID_soft_c [label="[Graphics Engine]"]

      // --- GPU FLOW ELEMENTS -- //

      UID_cond [label="algo implemented?"]
      UID_gpu_c [label="[ui_drawing_gpu.c]\nUI_DRAWING_drawImage();"]
      UID_gpu_cond [label="GPU compatible?"]
      UID_gpu_driver [label="[GPU driver]"]
      UID_gpu_hard [label="[GPU]"]

      UID_stub_h [label="[ui_drawing_stub.h]\nUI_DRAWING_STUB_drawImage();"]
      UID_stub_c [label="[ui_drawing_stub.c]\nUI_DRAWING_STUB_drawImage();"]
      stub [label="-"]

      // --- MULTIPLE IMAGES FLOW ELEMENTS -- //

      UII_h [label="[ui_image_drawing.h]\nUI_IMAGE_DRAWING_draw();"]
      UII_c [label="[ui_image_drawing.c]\nUI_IMAGE_DRAWING_draw();"]
      UII_cond [label="standard image?"]
      UIIx_c [label="[ui_image_drawing.c]\ntable[x] = UI_IMAGE_DRAWING_draw_customX()"]
      UIIx_weak_c [label="[ui_image_drawing.c]\nweak UI_IMAGE_DRAWING_draw_customX();"]
      UIIx_cond [label="implemented?"]
      UIIx_impl_c [label="[ui_image_x.c]\nUI_IMAGE_DRAWING_draw_customX()"]
      UIIx_impl_d [label="[custom drawing]"]

      UID_h2 [label="[ui_drawing.h]\n@see Simple Flow (chapter Drawings)"]

      // --- FLOW -- //

      mui->LLUI_h->LLUI_c->UID_h->UID_cond
      UID_cond->UID_weak_c [label="no" fontname="courier new" fontsize="10"]
      UID_weak_c->UII_h->UII_c->UII_cond
      UID_cond->UID_gpu_c [label="yes" fontname="courier new" fontsize="10"]
      UID_gpu_c->UID_gpu_cond
      UID_gpu_cond->UII_h [label="no" fontname="courier new" fontsize="10"]
      UID_gpu_cond->UID_gpu_driver [label="yes" fontname="courier new" fontsize="10"]
      UID_gpu_driver->UID_gpu_hard
      UII_cond->UID_soft_h [label="yes" fontname="courier new" fontsize="10"]
      UII_cond->UIIx_c [label="no" fontname="courier new" fontsize="10"]
      UID_soft_h->UID_soft_c
      UIIx_c->UIIx_cond
      UIIx_cond->UIIx_weak_c [label="no" fontname="courier new" fontsize="10"]
      UIIx_weak_c->UID_stub_h->UID_stub_c->stub
      UIIx_cond->UIIx_impl_c [label="yes" fontname="courier new" fontsize="10"]
      UIIx_impl_c->UIIx_impl_d
      UIIx_impl_d->UID_h2 [style=dotted label="optional
      (drawShapes)" fontname="courier new" fontsize="10"]
   }

.. force a new line

|

Take the same example as the *Standard Formats Only* implementation (draw an image):

**UI_DRAWING_DEFAULT_drawImage** (available in MicroUI C Module)

.. code-block:: c

   // use the compiler's 'weak' attribute
   __weak DRAWING_Status UI_DRAWING_DEFAULT_drawImage(MICROUI_GraphicsContext* gc, MICROUI_Image* img, jint regionX, jint regionY, jint width, jint height, jint x, jint y, jint alpha) {
   #if !defined(UI_FEATURE_IMAGE_CUSTOM_FORMATS)
      return UI_DRAWING_SOFT_drawImage(gc, img, regionX, regionY, width, height, x, y, alpha);
   #else
      return UI_IMAGE_DRAWING_draw(gc, img, regionX, regionY, width, height, x, y, alpha);
   #endif
   }

The define ``UI_FEATURE_IMAGE_CUSTOM_FORMATS`` is set so the implementation of the weak function redirects the image drawing to the image drawer manager (``ui_image_drawing.h``).

**UI_IMAGE_DRAWING_draw** (available in MicroUI C Module)

.. code-block:: c

   static const UI_IMAGE_DRAWING_draw_t UI_IMAGE_DRAWING_draw_custom[] = {
      &UI_DRAWING_STUB_drawImage,
      &UI_DRAWING_SOFT_drawImage,
      &UI_IMAGE_DRAWING_draw_custom0,
      &UI_IMAGE_DRAWING_draw_custom1,
      &UI_IMAGE_DRAWING_draw_custom2,
      &UI_IMAGE_DRAWING_draw_custom3,
      &UI_IMAGE_DRAWING_draw_custom4,
      &UI_IMAGE_DRAWING_draw_custom5,
      &UI_IMAGE_DRAWING_draw_custom6,
      &UI_IMAGE_DRAWING_draw_custom7,
   };

   DRAWING_Status UI_IMAGE_DRAWING_draw(MICROUI_GraphicsContext* gc, MICROUI_Image* img, jint regionX, jint regionY, jint width, jint height, jint x, jint y, jint alpha){
      return (*UI_IMAGE_DRAWING_draw_custom[_get_table_index(gc, img)])(gc, img, regionX, regionY, width, height, x, y, alpha);
   }

The implementation in the MicroUI C module redirects the drawing to the expected drawer.
The drawer is retrieved using its format (function ``_get_table_index()``):

* the format is standard but the destination is not the *display* format: index ``0`` is returned,
* the format is standard and the destination is the *display* format: index ``1`` is returned,
* the format is custom: an index from ``2`` to ``9`` is returned.

**UI_IMAGE_DRAWING_draw_custom0** (available in MicroUI C Module)

.. code-block:: c

   // Use the compiler's 'weak' attribute
   __weak DRAWING_Status UI_IMAGE_DRAWING_draw_custom0(MICROUI_GraphicsContext* gc, MICROUI_Image* img, jint regionX, jint regionY, jint width, jint height, jint x, jint y, jint alpha){
      return UI_DRAWING_STUB_drawImage(gc, img, regionX, regionY, width, height, x, y, alpha);
   }

The default implementation of ``UI_IMAGE_DRAWING_draw_custom0`` (same behavior for ``0`` to ``7``) consists in calling the stub implementation.

**UI_DRAWING_STUB_drawImage** (available in MicroUI C Module)

.. code-block:: c

  DRAWING_Status UI_DRAWING_STUB_drawImage(MICROUI_GraphicsContext* gc, MICROUI_Image* img, jint regionX, jint regionY, jint width, jint height, jint x, jint y, jint alpha){
    // Set the drawing log flag "not implemented"
    LLUI_DISPLAY_reportError(gc, DRAWING_LOG_NOT_IMPLEMENTED);
    return DRAWING_DONE;
  }

The implementation only consists in setting the :ref:`Drawing log flag <section.veeport.ui.drawings.drawing_logs>` ``DRAWING_LOG_NOT_IMPLEMENTED`` to notify the application that the drawing has not been performed.

.. _section_image_renderer_sim:

Simulation
==========

Principle
---------

The simulation behavior is similar to the :ref:`section_renderer_cco` for the Embedded side.

The :ref:`Front Panel<section_ui_releasenotes_frontpanel>` defines support for the drawers based on the Java service loader.

Standard Formats Only (Default)
-------------------------------

The default implementation can draw images with a standard format.

.. note:: Contrary to the :ref:`section_renderer_cco`, the simulation does not (and doesn't need to) provide an option to disable the use of custom image. 

The following diagram illustrates the drawing of an image:

.. graphviz:: :align: center

   digraph {
      ratio="auto"
      splines="true";
      bgcolor="transparent"
      node [style="filled,rounded" fontname="courier new" fontsize="10"]

      { //in/out
         node [shape="ellipse" color="#e5e9eb" fontcolor="black"] mui UID_soft_c UID_gpu_hard stub
      }
      { // h
         node [shape="box" color="#00aec7" fontcolor="white"] UID_h UID_soft_h UII_h
      }
      { // c
         node [shape="box" color="#ee502e" fontcolor="white"] LLUI_c UID_gpu_c UID_stub_c
      }
      { // weak
         node [shape="box" style="dashed,rounded" color="#ee502e"] UID_weak_c
      }
      { // choice
         node [shape="diamond" color="#e5e9eb"] UID_cond UID_gpu_cond UII_cond
      }

      // --- SIMPLE FLOW ELEMENTS -- //

      mui [label="[MicroUI]\nPainter.drawImage();"] 
      LLUI_c [label="[FrontPanel]\nLLUIPainter.drawImage();"]
      UID_h [label="[FrontPanel]\ngetUIDrawer().drawImage();"]
      UID_weak_c [label="[FrontPanel]\nDisplayDrawer.drawImage();"]
      UID_soft_h [label="[FrontPanel]\ngetUIDrawerSoftware()\n.drawImage();"]
      UID_soft_c [label="[Graphics Engine]"]

      // --- GPU FLOW ELEMENTS -- //

      UID_cond [label="method overridden?"]
      UID_gpu_c [label="[VEE Port FP]\nDisplayDrawerExtension\n.drawImage();"]
      UID_gpu_cond [label="can draw algo?"]
      UID_gpu_hard [label="[Third-party lib]"]

      UID_stub_c [label="[FrontPanel]\nno op"]
      stub [label="-"]

      // --- MULTIPLE IMAGES FLOW ELEMENTS -- //

      UII_h [label="[FrontPanel]\ngetUIImageDrawer()\n.drawImage();"]
      UII_cond [label="standard image?"]

      // --- FLOW -- //

      mui->LLUI_c->UID_h->UID_weak_c->UID_cond
      UID_cond->UII_h [label="no" fontname="courier new" fontsize="10"]
      UII_h->UII_cond
      UID_cond->UID_gpu_c [label="yes" fontname="courier new" fontsize="10"]
      UID_gpu_c->UID_gpu_cond
      UID_gpu_cond->UII_h [label="no" fontname="courier new" fontsize="10"]
      UID_gpu_cond->UID_gpu_hard [label="yes" fontname="courier new" fontsize="10"]
      UII_cond->UID_soft_h [label="yes" fontname="courier new" fontsize="10"]
      UII_cond->UID_stub_c [label="no" fontname="courier new" fontsize="10"]
      UID_soft_h->UID_soft_c
      UID_stub_c->stub
   }

.. force a new line

|

It is possible to override the image drawers for the standard format in the same way as the custom formats.

.. _section_buffered_image_drawer_custom_fp:

Custom Format Support 
---------------------

It is possible to draw images with a custom format by implementing the ``UIImageDrawing`` interface.
This advanced use case is available only with MicroUI 3.2 or higher.

The ``UIImageDrawing`` interface contains one method for each image drawing primitive (draw, copy, region, rotate, scale, flip).
Only the necessary methods have to be implemented.
Each non-implemented method will result in calling the stub implementation.

The method ``handledFormat()`` must be implemented and returns the image format handled by the drawer.

Once created, the ``UIImageDrawing`` implementation must be registered as a service.

The following diagram illustrates the drawing of an image: 

.. graphviz:: :align: center

   digraph {
      ratio="auto"
      splines="true";
      bgcolor="transparent"
      node [style="filled,rounded" fontname="courier new" fontsize="10"]

      { //in/out
         node [shape="ellipse" color="#e5e9eb" fontcolor="black"] mui UID_soft_c UID_gpu_hard stub UIIx_impl_d
      }
      { // h
         node [shape="box" color="#00aec7" fontcolor="white"] UID_h UID_soft_h UII_h UID_h2
      }
      { // c
         node [shape="box" color="#ee502e" fontcolor="white"] LLUI_c UID_gpu_c UID_stub_c UIIx_impl_c
      }
      { // weak
         node [shape="box" style="dashed,rounded" color="#ee502e"] UID_weak_c
      }
      { // choice
         node [shape="diamond" color="#e5e9eb"] UID_cond UID_gpu_cond UII_cond UIIx_cond
      }

      // --- SIMPLE FLOW ELEMENTS -- //

      mui [label="[MicroUI]\nPainter.drawImage();"] 
      LLUI_c [label="[FrontPanel]\nLLUIPainter.drawImage();"]
      UID_h [label="[FrontPanel]\ngetUIDrawer().drawImage();"]
      UID_weak_c [label="[FrontPanel]\nDisplayDrawer.drawImage();"]
      UID_soft_h [label="[FrontPanel]\ngetUIDrawerSoftware()\n.drawImage();"]
      UID_soft_c [label="[Graphics Engine]"]

      // --- GPU FLOW ELEMENTS -- //

      UID_cond [label="method overridden?"]
      UID_gpu_c [label="[VEE Port FP]\nDisplayDrawerExtension\n.drawImage();"]
      UID_gpu_cond [label="can draw algo?"]
      UID_gpu_hard [label="[Third-party lib]"]

      UID_stub_c [label="[FrontPanel]\nno op"]
      stub [label="-"]

      // --- MULTIPLE IMAGES FLOW ELEMENTS -- //

      UII_h [label="[FrontPanel]\ngetUIImageDrawer()\n.drawImage();"]
      UII_cond [label="standard image?"]
      UIIx_cond [label="available image drawer\nand method implemented?"]
      UIIx_impl_c [label="[VEE Port Fp]\nCustomImageDrawing.draw()"]
      UIIx_impl_d [label="[custom drawing]"]

      UID_h2 [label="[FrontPanel]\ngetUIDrawer().drawImage();\n@see Simple Flow (chapter Drawings)"]

      // --- FLOW -- //

      mui->LLUI_c->UID_h->UID_weak_c->UID_cond
      UID_cond->UII_h [label="no" fontname="courier new" fontsize="10"]
      UII_h->UII_cond
      UID_cond->UID_gpu_c [label="yes" fontname="courier new" fontsize="10"]
      UID_gpu_c->UID_gpu_cond
      UID_gpu_cond->UII_h [label="no" fontname="courier new" fontsize="10"]
      UID_gpu_cond->UID_gpu_hard [label="yes" fontname="courier new" fontsize="10"]
      UII_cond->UID_soft_h [label="yes" fontname="courier new" fontsize="10"]
      UII_cond->UIIx_cond [label="no" fontname="courier new" fontsize="10"]
      UID_soft_h->UID_soft_c
      UIIx_cond->UID_stub_c [label="no" fontname="courier new" fontsize="10"]
      UID_stub_c->stub
      UIIx_cond->UIIx_impl_c [label="yes" fontname="courier new" fontsize="10"]
      UIIx_impl_c->UIIx_impl_d
      UIIx_impl_d->UID_h2 [style=dotted label="optional\n(drawShapes)" fontname="courier new" fontsize="10"]
   }

.. force a new line

|

Let's implement the image drawer for the `CUSTOM_0` format.

.. code:: java

   public class MyCustomImageDrawer implements UIImageDrawing {

      @Override
      public MicroUIImageFormat handledFormat() {
         return MicroUIImageFormat.MICROUI_IMAGE_FORMAT_CUSTOM_0;
      }

      @Override
      public void draw(MicroUIGraphicsContext gc, MicroUIImage img, int regionX, int regionY, int width, int height,
            int x, int y, int alpha) {
         MyCustomImage customImage = (MyCustomImage) img.getImage().getRAWImage();
         customImage.drawOn(gc, regionX, regionY, width, height, x, y, alpha);
      }

   }

Now, this drawer needs to be registered as a service.
This can be achieved by creating a file in the resources of the Front Panel project named ``META-INF/services/ej.microui.display.UIImageDrawing`` and containing the fully qualified name of the previously created image drawer.

.. code-block::

   com.mycompany.MyCustomImageDrawer

It is also possible to declare it programmatically (see where a drawer is registered in the :ref:`drawing custom <section_drawings_sim_custom>` section):

.. code-block:: java

   LLUIDisplay.Instance.registerUIImageDrawer(new MyCustomImageDrawer());

.. _display_pixel_conversion:

Image Pixel Conversion
======================

Overview
--------

The Graphics Engine is built for a dedicated display pixel format (see :ref:`display_pixel_structure`).
For this pixel format, the Graphics Engine must be able to draw images with or without alpha blending and with or without transformation.
In addition, it must be able to read all image formats.

The application may not use all MicroUI image drawing options and may not use all images formats.
It is not possible to detect what the application needs, so no optimization can be performed at application compiletime.
However, for a given application, the VEE Port can be built with a reduced set of pixel support.

All pixel format manipulations (read, write, copy) are using dedicated functions.
It is possible to remove some functions or to use generic functions.
The advantage is to reduce the memory footprint.
The inconvenient is that some features are removed (the application should not use them) or some features are slower (generic functions are slower than the dedicated functions).

Functions
---------

There are five pixel *conversion* modes:

-  Draw an image without transformation and without global alpha blending: copy a pixel from a format to the destination format (display format).
-  Draw an image without transformation and with global alpha blending: copy a pixel with alpha blending from a format to the destination format (display format).
-  Draw an image with transformation and with or without alpha blending: draw an ARGB8888 pixel in destination format (display format).
-  Load a `ResourceImage`_ with an output format: convert an ARGB8888 pixel to the output format.
-  Read a pixel from an image (`Image.readPixel()`_ or to draw an image with transformation or to convert an image): read any pixel format and convert it to ARGB8888.

.. table:: Pixel Conversion

   +------------------------------------------+-------------+-------------+-------------+
   |                                          | Nb input    | Nb output   | Number of   |
   |                                          | formats     | formats     | combinations|
   +==========================================+=============+=============+=============+
   | Draw image without global alpha          |     22      |      1      |     22      |
   +------------------------------------------+-------------+-------------+-------------+
   | Draw image with global alpha             |     22      |      1      |     22      |
   +------------------------------------------+-------------+-------------+-------------+
   | Draw image with transformation           |      2      |      1      |      2      |
   +------------------------------------------+-------------+-------------+-------------+
   | Load a  ``ResourceImage``                |      1      |      6      |      6      |
   +------------------------------------------+-------------+-------------+-------------+
   | Read an image                            |     22      |      1      |     22      |
   +------------------------------------------+-------------+-------------+-------------+

There are ``22x1 + 22x1 + 2x1 + 1x6 + 22x1 = 74`` functions.
Each function takes between 50 and 200 bytes depending on its complexity and the C compiler.

.. _ResourceImage: https://repository.microej.com/javadoc/microej_5.x/apis/ej/microui/display/ResourceImage.html
.. _Image.readPixel(): https://repository.microej.com/javadoc/microej_5.x/apis/ej/microui/display/Image.html#readPixel-int-int-

Linker File
-----------

All pixel functions are listed in a VEE Port linker file.
It is possible to edit this file to remove some features or to share some functions (using generic function).

How to get the file:

#. Build VEE Port as usual.
#. Copy VEE Port file ``[platform]/source/link/display_image_x.lscf`` in the VEE Port configuration project: ``[VEE Port configuration project]/dropins/link/``. Where ``x`` is a number that characterizes the display pixel format (see :ref:`display_pixel_structure`). See next warning.
#. Perform some changes into the copied file (see after).
#. Rebuild the VEE Port: the file in the `dropins` folder is copied in the VEE Port instead of the original one.

.. warning:: When the display format in ``[VEE Port configuration project]/display/display.properties`` changes, the linker file suffix changes too. Perform again all the operations in the new file with the new suffix.

The linker file holds five tables, one for each use case, respectively ``IMAGE_UTILS_TABLE_COPY``, ``IMAGE_UTILS_TABLE_COPY_WITH_ALPHA``, ``IMAGE_UTILS_TABLE_DRAW``, ``IMAGE_UTILS_TABLE_SET`` and ``IMAGE_UTILS_TABLE_READ``.
For each table, a comment describes how to remove an option (when possible) or how to replace an option by a generic function (if available).

Installation
============

The Image Renderer module is part of the MicroUI module and Display module.
Install them to be able to use some images.

Use
===

The MicroUI image APIs are available in the class `ej.microui.display.Image`_.

.. _ej.microui.display.Image: https://repository.microej.com/javadoc/microej_5.x/apis/ej/microui/display/Image.html

..
   | Copyright 2008-2025, MicroEJ Corp. Content in this space is free 
   for read and redistribute. Except if otherwise stated, modification 
   is subject to MicroEJ Corp prior approval.
   | MicroEJ is a trademark of MicroEJ Corp. All other trademarks and 
   copyrights are the property of their respective owners.

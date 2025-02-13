.. _section_app_microui_errors:

Error Messages
==============

When an exception is thrown by the implementation of the MicroUI API, the exception `MicroUIException`_  with the error message ``MicroUI:E=<messageId>`` is issued, where the meaning of ``<messageId>`` is defined in following table:

.. _table_mui-error-msgs:
.. tabularcolumns:: |p{2cm}|p{13cm}|
.. list-table:: MicroUI Error Messages
   :widths: 2 20

   * - **Message ID**
     - **Description**
   * - 1
     - Another `EventGenerator`_ cannot be added into the system pool (max 254).
   * - 0
     - [VEE Port issue] Result of :ref:`MicroUI static initialization step<section_static_init>` seems invalid: MicroUI cannot start. Fix MicroUI static initialization step and rebuild the VEE Port.
   * - -1
     - MicroUI is not started; call MicroUI.start() before using a MicroUI API.
   * - -2
     - [Warning] Event generator specified during :ref:`MicroUI static initialization step<section_static_init>` is not available in the application classpath.
   * - -3
     - Deadlock. Cannot wait for an event in the same thread that runs events. `Display.waitFlushCompleted()`_ must not be called in the MicroUI thread (for example in ``render`` method).
   * - -4
     - Resource's path must be relative to the classpath (start with '/') or resource is not available.
   * - -5
     - The resource data cannot be read for unknown reason.
   * - -6
     - The resource has been closed and cannot be used anymore.
   * - -7
     - Out of memory. Not enough memory to allocate the `Image`_'s buffer. Try to close some useless images and retry opening the new image, or increase the size of the :ref:`MicroUI images heap<images_heap>`.
   * - -8
     - The VEE Port cannot decode this kind of image (the required runtime image decoder is not available in the VEE Port).
   * - -9
     - | This exception is thrown when the FIFO of the internal MicroUI thread is full. In this case, no more event (such as ``requestRender``, input events, etc.) can be added into it.
       | Most of time this error occurs when:
       | -  There is a user thread which performs too many calls to the method ``requestRender`` without waiting for the end of the previous drawing.
       | -  Too many input events are pushed from an input driver to the MicroUI thread (for example some touch events).
   * - -10
     - There is no display on the VEE Port.
   * - -11
     - There is no font (VEE Port and application).
   * - -12
     - The maximum number of event generators in the pool (254) has been reached.

.. _MicroUIException: https://repository.microej.com/javadoc/microej_5.x/apis/ej/microui/MicroUIException.html
.. _EventGenerator: https://repository.microej.com/javadoc/microej_5.x/apis/ej/microui/event/EventGenerator.html
.. _Display.waitFlushCompleted(): https://repository.microej.com/javadoc/microej_5.x/apis/ej/microui/display/Display.html#waitFlushCompleted--
.. _Image: https://repository.microej.com/javadoc/microej_5.x/apis/ej/microui/display/Image.html
..
   | Copyright 2008-2025, MicroEJ Corp. Content in this space is free 
   for read and redistribute. Except if otherwise stated, modification 
   is subject to MicroEJ Corp prior approval.
   | MicroEJ is a trademark of MicroEJ Corp. All other trademarks and 
   copyrights are the property of their respective owners.

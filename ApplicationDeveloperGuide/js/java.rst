
.. _js.java:

Communication Between Java code and JavaScript Code
===================================================

The MicroEJ engine allows to communicate between Java code and JavaScript code: Java API can be used from JavaScript code and vice-versa.

.. _js.java.engine:

JavaScript Engine
-----------------

The JavaScript code is executed in a single-threaded engine, which means only one JavaScript statement is executed at a given time.
Each piece of JavaScript code that must be executed is pushed in a job queue.
It is up to the engine to manage the job queue and execute the jobs.

One consequence of this design is that Java code called from a JavaScript code must not be blocker.
When calling a Java API from a Javascript code, in order to avoid blocking the JavaScript engine, the Java code must return as quick as possible.
Otherwise the JavaScript engine is stuck and cannot execute other JavaScript jobs.
It is especially harmfull when the Java operation takes time, for example for network or IO operations.
In such a case, it is therefore recommended to execute it in a new thread and return immediately.

Another consequence of the JavaScript engine design is that JavaScript code must always be executed by the engine, by the single thread.
Therefore, any call to a JavaScript code from a Java code must create a job and add it to the job queue.  

.. _js.java.java_to_js:

Calling Java from JavaScript
----------------------------

The MicroEJ engine allows to expose Java objects or methods to the JavaScript code by using the engine API and creating the adequate JavaScript object.

Import Java Types from JavaScript
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Java objects can be exposed to JavaScript using the ``JavaImport`` mechanism.
It takes a Java fully qualified name as argument and returns an object that 
gives access to the constructors, static methods and static fields. All the 
classes from the project's classpath can be imported (project's own classes and 
its dependencies).

For instance, the following code imports `java.lang.System`_ and prints a 
string calling `System.out.println()`_:

.. code-block:: javascript

	var System = JavaImport("java.lang.System")
	System.out.println("foo");

Here we instantiate a Java ``File`` object and check that it exists:

.. code-block:: javascript

	var File = JavaImport("java.io.File")
	var myFile = new File("myFile.txt")

	if (myFile.exists()) {
		print("myFile.txt exists")
	} else {
		print("myFile.txt does not exist")
	}

.. warning::

     You cannot instantiate an anonymous class from an interface or an abstract class with the ``new`` keyword and ``JavaImport``. Nevertheless, you can still access to static fields and methods.   

.. _java.lang.System: https://repository.microej.com/javadoc/microej_5.x/apis/java/lang/System.html
.. _System.out.println(): https://repository.microej.com/javadoc/microej_5.x/apis/java/io/PrintStream.html#println--

Implement JavaScript Functions in Java Code
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

We can also implement JavaScript functions in Java code by adding their 
implementation to the global object from Java code. For example, here is the code to 
create a JavaScript function named ``javaPrint`` in the global scope:

.. code-block:: java

    JsRuntime.JS_GLOBAL_OBJECT.put("javaPrint", JsRuntime.createFunction(new JsClosure() {
		@Override
		public Object invoke(Object thisBinding, Object... arguments) {
			System.out.println("Print from Java code: " + arguments[0]);
			return null;
		}
	}), false);

The function is created with a ``com.microej.js.objects.JsObjectFunction`` object created with the API ``JsRuntime.createFunction(JsClosure jsClosure)``,
and injected in the object ``JsRuntime.JS_GLOBAL_OBJECT`` which maps to the JavaScript global scope.

The function ``javaPrint`` can then be used in JS:

.. code-block:: javascript

    javaPrint("foo")

This technique can also be used to share any Java object to JavaScript.
It is achieved by returning the Java object in the ``invoke`` method of the ``JsClosure`` object.
For example, a Java `Date`_ object can be exposed as follows:

.. code-block:: java

	JsRuntime.JS_GLOBAL_OBJECT.put("getCurrentDate", JsRuntime.createFunction(new JsClosure() {
		@Override
		public Object invoke(Object thisBinding, Object... arguments) {
			return Calendar.getInstance().getTime();
		}
	}), false);

When a Java object is exposed in JavaScript, all its public methods can be called, therefore the JavaScript code can then use this `Date`_ object and get the time:

.. code-block:: javascript

	var date = getCurrentDate()
	var time = date.getTime()
	print("Current time: ", time)

for more information on how these called are managed by the MicroEJ JavaScript engine, please go to the :ref:`js.java.ffi` section.

Java objects can also be shared using one of the other Java JS adapter objects.
With this solution, the code of the Java object is executed at engine initialisation, contrary to the previous solution where it is executed only when the JavaScript code is called.
For example, here is the code to expose a Java string named ``javaString`` in the JavaScript global scope:

.. code-block:: java

    JsRuntime.JS_GLOBAL_OBJECT.put("javaString", "Hello World!", false);

The string ``javaString`` can then be used in JS:

.. code-block:: javascript

    var myString = javaString;

The available Java JS adapter objects are:

- ``com.microej.js.objects.JsObject`` : exposes a Java object as a JavaScript object
- ``com.microej.js.objects.JsObjectFunction`` : exposes a Java "process" as a JavaScript function (using a JsClosure object)
- ``com.microej.js.objects.JsObjectString`` : exposes a Java String as a JavaScript String
- ``com.microej.js.objects.JsObjectArray`` : exposes a Java items collection as a JavaScript Array
- ``com.microej.js.objects.JsObjectBoolean`` : exposes a Java Boolean as a JavaScript Boolean
- ``com.microej.js.objects.JsObjectNumber`` : exposes a Java Number as a JavaScript Number

.. _Date: https://repository.microej.com/javadoc/microej_5.x/apis/java/util/Date.html

.. _js.java.js_to_java:

Calling JavaScript from Java
----------------------------

The MicroEJ JavaScript engine API allows to call JavaScript code from Java code.
For example, given the following JavaScript function in a file in ``src/main/js``:

.. code-block:: javascript

    function sum(a, b) {
        print(a + " + " + b + " = " + (a+b));
    }

it can be called from a Java piece of code with:

.. code-block:: java

    JsObjectFunction functionObject = (JsObjectFunction) JsRuntime.JS_GLOBAL_OBJECT.get("sum");
    JsRuntime.ENGINE.addJob(functionObject, JsRuntime.JS_GLOBAL_OBJECT, new Integer(5), new Integer(3));

The first line gets the JavaScript function from the global scope.
The second line adds a job in the JavaScript engine queue to execute the function, in the global scope, with the arguments ``5`` and ``3``.

Passing Values Between JavaScript and Java
------------------------------------------

JavaScript base types are represented by Java objects and not Java base types. 
The following table shows the mapping between types in both languages: 

.. list-table::
    :widths: 20 40

    * - **JavaScript**
      - **Java**
    * - Number
      - `java.lang.Integer`_ or `java.lang.Double`_
    * - Boolean
      - `java.lang.Boolean`_
    * - String
      - `java.lang.String`_
    * - Null
      - ``null`` value
    * - Undefined
      - ``JsRuntime.JS_UNDEFINED_OBJECT`` singleton


In JavaScript, a ``Number`` type is a 64-bits floating-point value. 
Nevertheless, Kifaru may use integer values (`Integer`_ Java type) when 
possible for performance reasons. Otherwhise, `Double`_ type will be used.

.. note::

    Prefer passing `Integer`_ values as argument to a job added to the JavaScript execution queue, or return ``Integer`` values when implementing a ``JsClosure`` instead of `Double`_ when possible.

It is not possible to retrieve the returned value of a JavaScript function from 
Java code. For instance, consider the following JavaScript function:

.. code-block:: javascript

    function sum(a, b) {
        return a + b;
    }

When calling this function from Java code, we have no way to get the result back:

.. code-block:: java

    JsObjectFunction functionObject = (JsObjectFunction) JsRuntime.JS_GLOBAL_OBJECT.get("sum");
    JsRuntime.ENGINE.addJob(functionObject, JsRuntime.JS_GLOBAL_OBJECT, new Integer(5), new Integer(3));

A workaround is to modify the JavaScript function so it takes a callback object 
as argument:

.. code-block:: javascript

    function sum(a, b, callback) {
        callback.returnValue(a + b);
    }

Here is a possible implementation of the callback object:

.. code-block:: java

    public class Callback<T> {

        @Nullable
        private T value;

        private boolean returned;

        /**
         * Gets the value returned by this callback function when ready.
         * <p>
         * A call to this method waits for the value to be ready.
         *
         * @return the value return by the callback
         */
	    @Nullable
        public T getValue() {
            synchronized (this) {
                while (!this.returned) {
                    try {
                        wait();
                    } catch (InterruptedException e) {
                        throw new JsErrorWrapper(""); //$NON-NLS-1$
                    }
                }
            }

             return this.value;
        }

        /**
         * Sets the value to return by this callback function.
         *
         * @param value
         *            the value to return
         */
        public synchronized void returnValue(@Nullable T value) {
            this.value = value;
            this.returned = true;
            notify();
        }
    }

We can now pass the callback to the job. The Java code will wait on the 
``callback.getValue()`` until the result is ready.

.. code-block:: java

    JsObjectFunction functionObject = (JsObjectFunction) JsRuntime.JS_GLOBAL_OBJECT.get("sum");
    Callback<Integer> callback = new Callback<>();
    JsRuntime.ENGINE.addJob(functionObject, JsRuntime.JS_GLOBAL_OBJECT, new Integer(5), new Integer(3), callback);
    Integer returnedValue = callback.getValue();
    System.out.println("Result is " + returnedValue);


.. _java.lang.Integer: https://repository.microej.com/javadoc/microej_5.x/apis/java/lang/Integer.html
.. _Integer: https://repository.microej.com/javadoc/microej_5.x/apis/java/lang/Integer.html
.. _java.lang.Double: https://repository.microej.com/javadoc/microej_5.x/apis/java/lang/Double.html
.. _Double: https://repository.microej.com/javadoc/microej_5.x/apis/java/lang/Double.html
.. _java.lang.Boolean: https://repository.microej.com/javadoc/microej_5.x/apis/java/lang/Boolean.html
.. _java.lang.String: https://repository.microej.com/javadoc/microej_5.x/apis/java/lang/String.html

..
   | Copyright 2008-2025, MicroEJ Corp. Content in this space is free 
   for read and redistribute. Except if otherwise stated, modification 
   is subject to MicroEJ Corp prior approval.
   | MicroEJ is a trademark of MicroEJ Corp. All other trademarks and 
   copyrights are the property of their respective owners.

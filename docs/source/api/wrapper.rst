.. py:module:: STPyV8
    :noindex:

.. testsetup:: *

   from STPyV8 import *

.. _wrapper:

Interoperability
================

STPyV8 was designed so that the end user doesn't need to spend additional efforts for interoperability. This means your Python
code could seamlessy access a property or call a Javascript function and STPyV8 will take care of performing all the dirty work
i.e. :ref:`typeconv`, :ref:`funcall` and :ref:`exctrans`.

.. _typeconv:

Type Conversion
---------------

Python and Javascript have different type system and build-in primitives so STPyV8 is required to transparently perform the type
conversion operations.

Python to Javascript
^^^^^^^^^^^^^^^^^^^^

When converting a Python value to Javascript value, STPyV8 will try the following conversion rules first. The remaining unknown
Python types will be converted to a plain Javascript object.

=============================   ===============     ==================  ================
Python Type                     Python Value        Javascript Type     Javascript Value
=============================   ===============     ==================  ================
:py:class:`NoneType`            None                Object/Null [#f1]_  null
:py:func:`bool`                 True/False          Boolean [#f2]_      true/false
:py:func:`int`                  123                 Number [#f3]_       123
:py:func:`long`                 123                 Number              123
:py:func:`float`                3.14                Number              3.14
:py:func:`str`                  'test'              String [#f4]_       'test'
:py:func:`unicode`              u'test'             String              'test'
:py:class:`datetime.datetime`                       Date [#f5]_
:py:class:`datetime.time`                           Date
built-in function                                   Object/Function
function/method                                     Object/Function
:py:func:`type`                                     Object/Function
=============================   ===============     ==================  ================

.. doctest::

    >>> ctxt = JSContext()
    >>> ctxt.enter()

    >>> typeof = ctxt.eval("(function type(value) { return typeof value; })")
    >>> protoof = ctxt.eval("(function protoof(value) { return Object.prototype.toString.apply(value); })")

    >>> protoof(None)
    '[object Null]'
    >>> protoof(True)
    '[object Boolean]'
    >>> typeof(True)
    'boolean'
    >>> typeof(123)
    'number'
    >>> typeof(3.14)
    'number'
    >>> typeof('test')
    'string'
    >>> typeof(u'test')
    'string'

    >>> from datetime import *
    >>> protoof(datetime.now())
    '[object Date]'
    >>> protoof(date.today())
    '[object Date]'
    >>> protoof(time())
    '[object Date]'

    >>> protoof(abs)
    '[object Function]'
    >>> def test():
    ...     pass
    >>> protoof(test)
    '[object Function]'
    >>> protoof(int)
    '[object Function]'

.. note::

    All the Python *functions*, *methods* and *types* will be converted to a Javascript function object, because the Python *type* could be used as a constructor and create a new instance.

Javascript to Python
^^^^^^^^^^^^^^^^^^^^

When converting from Javascript to Python, STPyV8 will try the following conversion rules

===============     ================    =============================   ============
Javascript Type     Javascript Value    Python Type                     Python Value
===============     ================    =============================   ============
Null                null                :py:class:`NoneType`            None
Undefined           undefined           :py:class:`NoneType`            None
Boolean             true/false          :py:func:`bool`                 True/False
String              'test'              :py:func:`str`                  'test'
Number/Int32        123                 :py:func:`int`                  123
Number              3.14                :py:func:`float`                3.14
Date                                    :py:class:`datetime.datetime`
Array [#f6]_                            :py:class:`JSArray`
Function                                :py:class:`JSFunction`
Object                                  :py:class:`JSObject`
===============     ================    =============================   ============

.. note::

    ECMAScript standard defines only one Number type for both integer and float numbers but Google V8 defined the Integer/Int32/Uint32 class to improve the overall performances. STPyV8 makes use of the same internal V8 type system to perform the same job.

.. doctest::

    >>> ctxt = JSContext()
    >>> ctxt.enter()

    >>> type(ctxt.eval("null"))
    <type 'NoneType'>
    >>> type(ctxt.eval("undefined"))
    <type 'NoneType'>
    >>> type(ctxt.eval("true"))
    <type 'bool'>
    >>> type(ctxt.eval("'test'"))
    <type 'str'>
    >>> type(ctxt.eval("123"))
    <type 'int'>
    >>> type(ctxt.eval("3.14"))
    <type 'float'>
    >>> type(ctxt.eval("new Date()"))
    <type 'datetime.datetime'>
    >>> type(ctxt.eval("[1, 2, 3]"))
    <class '_STPyV8.JSArray'>
    >>> type(ctxt.eval("(function() {})"))
    <class '_STPyV8.JSFunction'>
    >>> type(ctxt.eval("new Object()"))
    <class '_STPyV8.JSObject'>

Sequences and Arrays
^^^^^^^^^^^^^^^^^^^^

.. sidebar:: sequence

    An iterable which supports efficient element access using integer indices via the :py:meth:`object.__getitem__` special method and defines a :py:func:`len` method that returns the length of the sequence. Some built-in sequence types are :py:func:`list`, :py:func:`str`, :py:func:`tuple`, and :py:func:`unicode`. 

Since Python haven't built-in Array type but defines a sequence concept, you could access a object as a sequence if it implement the sequence interface.

STPyV8 provides the :py:class:`JSArray` class which wraps the Javascript Array object acting as a Python sequence. You could access and modify it just like a normal Python list.

.. note::

    The Javascript Array object supports the Sparse Array [#f7]_ because it was implemented as an Associative Array [#f8]_, just like the :py:class:`dict` class in the Python. So adding a new item in the :py:class:`JSArray` with an index value larger than the length guarantees the padding item will always be None.

.. doctest::

    >>> ctxt = JSContext()
    >>> ctxt.enter()

    >>> array = ctxt.eval('[1, 2, 3]')

    >>> array[1]            # via :py:meth:`JSArray.__getitem__`
    2
    >>> len(array)          # via :py:meth:`JSArray.__len__`
    3

    >>> 2 in array          # via :py:meth:`JSArray.__contains__`
    True
    >>> del array[1]        # via :py:meth:`JSArray.__delitem__`
    >>> 2 in array
    False

    >>> array[5] = 3        # via :py:meth:`JSArray.__setitem__`
    >>> [i for i in array]  # via :py:meth:`JSArray.__iter__`
    [1, None, 3, None, None, 3]

On the other hand, Javascript code could access all the Python sequence types as a array like object. 

.. doctest::

    >>> ctxt = JSContext()
    >>> ctxt.enter()

    >>> ctxt.locals.array = [1, 2, 3]

    >>> ctxt.eval('array[1]')
    2

    >>> ctxt.eval('delete array[1]')
    True
    >>> ctxt.locals.array
    [1, 3]

    >>> ctxt.eval('array[0] = 4')
    4
    >>> ctxt.locals.array
    [4, 3]

    >>> ctxt.eval('var sum=0; for (i in array) sum+= array[i]; sum')
    7

    >>> ctxt.eval('0 in array') # check whether the index exists
    True
    >>> ctxt.eval('3 in array') # array contains the value 3 but not the index 3
    False

    >>> ctxt.locals.len = len
    >>> ctxt.eval('len(array)')
    2

.. note::

    The Python sequence doesn't support the Javascript Array properties or methods, such as *length* etc. You could directly use the Python :py:func:`len` function in the Javascript code.

If you want to pass a real Javascript Array, you could directly create a :py:class:`JSArray` instance passing a Python :py:func:`list` as the parameter to the :py:meth:`JSArray.__init__` constructor.

.. doctest::

    >>> ctxt = JSContext()
    >>> ctxt.enter()

    >>> ctxt.locals.array = JSArray([1, 2, 3])

    >>> ctxt.eval("Object.prototype.toString.apply(array)")
    '[object Array]'
    >>> ctxt.eval("array.length")
    3

Mapping and Property
^^^^^^^^^^^^^^^^^^^^

.. sidebar:: mapping

    A container object (such as dict) which supports arbitrary key lookups using the special method :py:meth:`object.__getitem__`.

Like the sequence types, Python also defines mapping types, such as :py:class:`dict` etc.

You could access the Python mapping with a named index or as a property and STPyV8 will access the value based on its type.

.. doctest::

    >>> ctxt = JSContext()
    >>> ctxt.enter()

    >>> ctxt.locals.dict = { 'a':1, 'b':2 }

    >>> ctxt.eval("dict['a']")  # named index
    1
    >>> ctxt.eval('dict.a')     # property
    1

    >>> ctxt.eval("'a' in dict")
    True

    >>> ctxt.eval("dict['c'] = 3")
    3
    >>> ctxt.locals.dict
    {'a': 1, 'c': 3, 'b': 2}

All the Javascript objects could be accessed from the Python side as a mapping. You could get the keys with :py:meth:`JSObject.keys`, or check whether a key is exists with :py:meth:`JSObject.__contains__`.

.. doctest::

    >>> ctxt = JSContext()
    >>> ctxt.enter()

    >>> ctxt.eval("var obj = {a:1, b:2};")

    >>> ctxt.locals.obj['a']        # via :py:meth:`JSObject.__getitem__`
    1
    >>> ctxt.locals.obj.a           # via :py:meth:`JSObject.__getattr__`
    1

    >>> 'a' in ctxt.locals.obj      # via :py:meth:`JSObject.__contains__`
    True

    >>> ctxt.locals.obj['c'] = 3    # via :py:meth:`JSObject.__setitem__`
    >>> dict([(k, ctxt.locals.obj[k]) for k in ctxt.locals.obj.keys()])
    {'a': 1, 'c': 3, 'b': 2}

The Python new-style object [#f9]_ allows to define a property with a getter, a setter and a deleter. STPyV8 will propertly handle the property if built with the SUPPORT_PROPERTY enabled in the Config.h file (enabled by default). In such case the getter, the setter and/or the deleter will be called when the Javascript code access the property

.. testcode::

    class Global(JSClass):
        def __init__(self, name):
            self._name = name
        def getname(self):
            return self._name
        def setname(self, name):
            self._name = name
        def delname(self):
            self._name = 'deleted'
        name = property(getname, setname, delname)

    with JSContext(Global('test')) as ctxt:
        print(ctxt.eval("name"))                 # test
        print(ctxt.eval("this.name = 'flier';")) # flier
        print(ctxt.eval("name"))                 # flier
        print(ctxt.eval("delete name"))          # True

.. testoutput::
   :hide:

   test
   flier
   flier
   True

.. _funcall:

Function and Constructor
------------------------

.. _exctrans:

Exception Translation
---------------------

Based on the aforementioned design principles, STPyV8 will take care of converting the exceptions between Python and Javascript so that direct use of **try...except** statements allows to handle the Javascript exception in the Python code, and vice versa.

.. testcode::

    with JSContext() as ctxt:
        try:
            ctxt.eval("throw Error('test');")
        except JSError as e:
            print(e)            # JSError: Error: test (  @ 1 : 6 )  -> throw Error('test');
            print(e.name)
            print(e.message)

.. testoutput::
   :hide:

   JSError: Error: test (  @ 1 : 6 )  -> throw Error('test');
   Error
   test

If the Javascript code throws a well known exception, it will be translated to an equivalent Python exception. Other
Javascript exceptions will be wrapped as a :py:class:`JSError` instance. You could access the :py:class:`JSError`
properties for more detail.

====================    ========================
Javascript Exception    Python Exception
====================    ========================
RangeError              :py:exc:`IndexError`
ReferenceError          :py:exc:`ReferenceError`
SyntaxError             :py:exc:`SyntaxError`
TypeError               :py:exc:`TypeError`
Error                   :py:class:`JSError`
====================    ========================

From the Javascript side, you could also use **try...catch** statements to catch Python exceptions.

.. testcode::

    class Global(JSClass):
        def raiseIndexError(self):
            return [1, 2, 3][5]

    with JSContext(Global()) as ctxt:
        ctxt.eval("try { this.raiseIndexError(); } catch (e) { msg = e; }")

        print(ctxt.locals.msg)   # RangeError: list index out of range

.. testoutput::
   :hide:

   RangeError: list index out of range
   
JSClass
-------

.. autoclass:: JSClass
   :members:
   :inherited-members:

JSObject
--------

.. autoclass:: JSObject
   :members:
   :inherited-members:

   .. automethod:: __getattr__(name) -> object

        Called when an attribute lookup has not found the attribute in the usual places (i.e. it is not an instance attribute nor is it found in the class tree for self). name is the attribute name. This method should return the (computed) attribute value or raise an AttributeError exception.

        .. seealso:: :py:meth:`object.__getattr__`

   .. automethod:: __setattr__(name, value) -> None

        Called when an attribute assignment is attempted. This is called instead of the normal mechanism (i.e. store the value in the instance dictionary). name is the attribute name, value is the value to be assigned to it.

        .. seealso:: :py:meth:`object.__setattr__`

   .. automethod:: __delattr__(name) -> None

        Like __setattr__() but for attribute deletion instead of assignment. This should only be implemented if del obj.name is meaningful for the object.

        .. seealso:: :py:meth:`object.__delattr__`

   .. automethod:: __hash__() -> int

        Called by built-in function hash() and for operations on members of hashed collections including set, frozenset, and dict. __hash__() should return an integer. The only required property is that objects which compare equal have the same hash value; it is advised to somehow mix together (e.g. using exclusive or) the hash values for the components of the object that also play a part in comparison of objects.

        .. seealso:: :py:meth:`object.__hash__`

   .. automethod:: __getitem__(key) -> object

        Called to implement evaluation of self[key]. For sequence types, the accepted keys should be integers and slice objects. Note that the special interpretation of negative indexes (if the class wishes to emulate a sequence type) is up to the __getitem__() method. If key is of an inappropriate type, TypeError may be raised; if of a value outside the set of indexes for the sequence (after any special interpretation of negative values), IndexError should be raised. For mapping types, if key is missing (not in the container), KeyError should be raised.

        .. seealso:: :py:meth:`object.__getitem__`
   
   .. automethod:: __setitem__(key, value) -> None
   
        Called to implement assignment to self[key]. Same note as for __getitem__(). This should only be implemented for mappings if the objects support changes to the values for keys, or if new keys can be added, or for sequences if elements can be replaced. The same exceptions should be raised for improper key values as for the __getitem__() method.

        .. seealso:: :py:meth:`object.__setitem__`

   .. automethod:: __delitem__(key) -> None

        Called to implement deletion of self[key]. Same note as for __getitem__(). This should only be implemented for mappings if the objects support removal of keys, or for sequences if elements can be removed from the sequence. The same exceptions should be raised for improper key values as for the __getitem__() method.

        .. seealso:: :py:meth:`object.__delitem__`

   .. automethod:: __contains__(key) -> bool

        Called to implement membership test operators. Should return true if item is in self, false otherwise. For mapping objects, this should consider the keys of the mapping rather than the values or the key-item pairs.

        .. seealso:: :py:meth:`object.__contains__`

   .. automethod:: __bool__() -> bool

        Called to implement truth value testing and the built-in operation bool(); should return False or True, or their integer equivalents 0 or 1. When this method is not defined, __len__() is called, if it is defined, and the object is considered true if its result is nonzero. If a class defines neither __len__() nor __bool__(), all its instances are considered true.

        .. seealso:: :py:meth:`object.__bool__`

   .. automethod:: __eq__(other) -> bool

        .. seealso:: :py:meth:`object.__eq__`

   .. automethod:: __ne__(other) -> bool

        .. seealso:: :py:meth:`object.__ne__`

   .. automethod:: __int__() -> int

        .. seealso:: :py:meth:`object.__int__`

   .. automethod:: __float__() -> float

        .. seealso:: :py:meth:`object.__float__`

   .. automethod:: __str__() -> str

        .. seealso:: :py:meth:`object.__str__`


JSArray
-------

.. autoclass:: JSArray
   :members:
   :inherited-members:

   .. automethod:: __init__(size or items) -> JSArray object

      :param int size: The array size
      :param list items: The item list

   .. automethod:: __len__() -> int

       Called to implement the built-in function len(). Should return the length of the object, an integer >= 0. Also, an object that doesn’t define a __bool__() method and whose __len__() method returns zero is considered to be false in a Boolean context.

       .. seealso:: :py:meth:`object.__len__`

   .. automethod:: __getitem__(key) -> object

        Called to implement evaluation of self[key]. For sequence types, the accepted keys should be integers and slice objects. Note that the special interpretation of negative indexes (if the class wishes to emulate a sequence type) is up to the __getitem__() method. If key is of an inappropriate type, TypeError may be raised; if of a value outside the set of indexes for the sequence (after any special interpretation of negative values), IndexError should be raised. For mapping types, if key is missing (not in the container), KeyError should be raised.

        .. seealso:: :py:meth:`object.__getitem__`

   .. automethod:: __setitem__(key, value) -> None

        Called to implement assignment to self[key]. Same note as for __getitem__(). This should only be implemented for mappings if the objects support changes to the values for keys, or if new keys can be added, or for sequences if elements can be replaced. The same exceptions should be raised for improper key values as for the __getitem__() method.

        .. seealso:: :py:meth:`object.__setitem__`

   .. automethod:: __delitem__(key) -> None

        Called to implement deletion of self[key]. Same note as for __getitem__(). This should only be implemented for mappings if the objects support removal of keys, or for sequences if elements can be removed from the sequence. The same exceptions should be raised for improper key values as for the __getitem__() method.

        .. seealso:: :py:meth:`object.__delitem__`
        
   .. automethod:: __iter__() -> iterator

       This method is called when an iterator is required for a container. This method should return a new iterator object that can iterate over all the objects in the container. For mappings, it should iterate over the keys of the container, and should also be made available as the method iterkeys().

       .. seealso:: :py:meth:`object.__iter__`

   .. automethod:: __contains__(item) -> bool

       Called to implement membership test operators. Should return true if item is in self, false otherwise. For mapping objects, this should consider the keys of the mapping rather than the values or the key-item pairs.

       .. seealso:: :py:meth:`object.__contains__`
       
JSFunction
----------

.. autoclass:: JSFunction
   :members:
   :inherited-members:

   .. automethod:: __call__(*args, **kwds) -> object

       Called when the instance is “called” as a function; if this method is defined, x(arg1, arg2, ...) is a shorthand for x.__call__(arg1, arg2, ...).

      :param list args: the argument list
      :param dict kwds: the argument dictionary whose keys are strings

       .. seealso:: :py:meth:`object.__call__`

   .. automethod:: apply(self, args=[], kwds={}) -> object

      The function is called with *args* as the argument list; the number of arguments is the length of the tuple. If the optional *kwds* argument is present, it must be a dictionary whose keys are strings. It specifies keyword arguments to be added to the end of the argument list.

      :param self: The :py:class:`JSObject` instance as self
      :type self: :py:class:`JSObject`
      :param list args: the argument list
      :param dict kwds: the argument dictionary whose keys are strings

JSError
-------

.. autoclass:: JSError
   :members:
   :inherited-members:

   .. automethod:: __str__() -> str

      .. seealso:: :py:meth:`object.__str__`

   .. automethod:: __getattribute__(attr) -> object

      .. seealso:: :py:meth:`object.__getattribute__`

   .. py:attribute:: name -> str

      The exception name.

   .. py:attribute:: message -> str

      The exception message.

   .. py:attribute:: scriptName -> str

      The script name which throw the exception.

   .. py:attribute:: lineNum -> int

      The line number of error statement.

   .. py:attribute:: startPos -> int

      The start position of error statement in the script.

   .. py:attribute:: endPos -> int

      The end position of error statement in the script.

   .. py:attribute:: startCol -> int

      The start column of error statement in the script.

   .. py:attribute:: endCol -> int

      The end column of error statement in the script.

   .. py:attribute:: sourceLine -> str

      The source line of error statement.

   .. py:attribute:: stackTrace -> str

      The stack trace of error statement.

   .. py:method:: print_tb(out=sys.stdout -> file object) -> None

      Print the stack trace of error statement.

.. toctree::
   :maxdepth: 2

.. rubric:: Footnotes

.. [#f1] **Null Type**

         The type Null has exactly one value, called null.

.. [#f2] **Boolean Type**

         The type Boolean represents a logical entity and consists of exactly two unique values. One is called true and the other is called false.

.. [#f3] **Number Type**

        The type **Number** is a set of values representing numbers. In ECMAScript, the set of values represents the double-precision 64-bit format IEEE 754 values including the special “Not-a-Number” (NaN) values, positive infinity, and negative infinity.

.. [#f4] **String Type**

        The type String is the set of all string values. A string value is a finite ordered sequence of zero or more 16-bit unsigned integer values.

.. [#f5] **Date Object**

        A Date object contains a number indicating a particular instant in time to within a millisecond. The number may also be NaN, indicating that the Date object does not represent a specific instant of time.

.. [#f6] **Array Object**

        Array objects give special treatment to a certain class of property names. A property name P (in the form of a string value) is an array index if and only if ToString(ToUint32(P)) is equal to P and ToUint32(P) is not equal to 232−1.

.. [#f7] `Sparse Array <http://en.wikipedia.org/wiki/Sparse_array>`_

        A sparse array is an array in which most of the elements have the same value (known as the default value—usually 0 or null). The occurence of zero elements in a large array is both computational and storage inconvenient. An array in which there is large number of zero elements is referred to as being sparse.

.. [#f8] `Associative Array <http://en.wikipedia.org/wiki/Associative_array>`_

        An associative array (also associative container, map, mapping, dictionary, finite map, table, and in query processing, an index or index file) is an abstract data type composed of a collection of unique keys and a collection of values, where each key is associated with one value (or set of values). The operation of finding the value associated with a key is called a lookup or indexing, and this is the most important operation supported by an associative array. The relationship between a key and its value is sometimes called a mapping or binding. For example, if the value associated with the key "joel" is 1, we say that our array maps "joel" to 1. Associative arrays are very closely related to the mathematical concept of a function with a finite domain. As a consequence, a common and important use of associative arrays is in memoization.

.. [#f9] `new-style class <http://docs.python.org/reference/datamodel.html#new-style-and-classic-classes>`_
        Any class which inherits from :py:class:`object`. This includes all built-in types like :py:func:`list` and :py:class:`dict`. Only new-style classes can use Python’s newer, versatile features like :py:meth:`object.__slots__`, descriptors, properties, and :py:meth:`object.__getattribute__`.
        
        New-style classes were introduced in Python 2.2 to unify classes and types. A new-style class is neither more nor less than a user-defined type. If x is an instance of a new-style class, then type(x) is typically the same as x.__class__ (although this is not guaranteed - a new-style class instance is permitted to override the value returned for x.__class__).
        
        The major motivation for introducing new-style classes is to provide a unified object model with a full meta-model. It also has a number of practical benefits, like the ability to subclass most built-in types, or the introduction of “descriptors”, which enable computed properties.

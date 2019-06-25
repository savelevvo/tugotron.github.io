---
layout: post
title: "How Python import works"
comments: true
description: "This article is not an original description of import process. Best parts of descriptions from different sources are combined here, so it might be considered as a summary."
keywords: "python, import"
---

## Introduction

This article is not an original description of import process. Best parts of descriptions from different sources are combined here, so it might be considered as a summary.

It’s gathered here to give the reader a deeper understanding of processes taking place during import. It’s not going to teach basic concepts or best practices of language usage. It’s just meant to discover what’s behind such common statement as import. It assumes you are familiar with basic principles of python such as namespaces, functions and classes. And it’s relevant for Python version 3.

When you’re using Python, you’re always in a module. There is no way to write and run Python code outside of a module, even in shell - it uses `__main__` module. That’s why it’s better to understand such a fundamental concept.

<br>

## Namespaces and modules

Namespace is a dictionary. Following functions return a dict that contains names and references to the values: `globals()`, `locals()`

<script src="https://gist.github.com/tugotron/e0aa8dc7d3f465ed9da5440881752d7d.js"></script>

Module is a collection of Python definitions and statements that you have access to from a script executed at the top level. After the module is imported, it becomes a part of the namespace which means it’s a dictionary that contains symbols and whatever they correspond to. It appears in `sys.modules` and looks like this:

<script src="https://gist.github.com/tugotron/115c35fa98b333d2bd14a110a7f01619.js"></script>

Module has it's own namespace that can be seen in `__dict__` property

<script src="https://gist.github.com/tugotron/5702b32b4398675e3af3919f21043d35.js"></script>
<br>

## Different ways to initiate import

The `import` statement is the most common way of invoking the import machinery, but it is not the only way. Functions such as `importlib.import_module()` and built-in `__import__()` can also be used. When an import statement is executed, the standard built-in `__import__()` function is called. Other mechanisms for invoking the import system (such as `importlib.import_module()`) may use their own solutions to implement import semantics.

<script src="https://gist.github.com/tugotron/62d7049f5eebc03522037c6829146963.js"></script>

Normally, the top-level package (the name up till the first dot) is returned, not the module named by `name` argument. However, when a non-empty `fromlist` argument is given, the module named by `name` is returned.

Examples of different `import` statements and what they results after resembled into bytecode

<script src="https://gist.github.com/tugotron/b2f1d9c90c221c7188ce36e1c825a80f.js"></script>
<br>

## Steps of import process

When we run a statement like `import fractions` what is Python doing? 

First, the `sys.modules` dict is checked to see if the module has already been imported. If so it returns the module object.

Otherwise, Python should find module file. Python’s import protocol is invoked to find it and load. This search protocol consists of two conceptual objects: finders and loaders. 

Finder - an object that tries to find the loader for a module that is being imported. Finders do not actually load modules. If they can find the named module, they return a module specifications object  `__spec__` (since Python 3.4) that contain loaders, which the import machinery then uses when loading the module. Otherwise, they return `None`.

An object that both finds and loads a module is called importer. It returns themself when it finds that it can load the requested module. Python includes a number of default finders and importers. 

<script src="https://gist.github.com/tugotron/70953f8aa57eb827d7578e0bd82dbd21.js"></script>

In more detailed way it would look like:
1. Find module file using finders
2. Module code has to be retrieved (loaded) using loaders
3. Empty module typed object is created
4. Reference to the module is added to the system cache (`sys.modules`)
5. Compile into byte-code (it’s only needed if `.pyc` file isn’t older than `.py` file)
6. Execute to create objects that are defined in it and set up module’s namespace (`module.__dict__` is `module.globals()`)

The `sys` module has some properties that define where Python is going to look for modules (either built-in or standard library as well). Here are the most important ones:

<script src="https://gist.github.com/tugotron/4d660d7ab416b4fc22204417953589c8.js"></script>

`sys.meta_path` by default has a list of three finders: 
finder for built-in modules
finder for frozen modules
path based finder - a list of locations that usually comes from `sys.path`
These finders are queried in order to see if anyone of them know how to handle the named module.

`sys.path` contains (in order of search Python performs):
The directory where the application is located (current directory)
What’s in variable `PYTHONPATH` (if it’s defined)
Standard library folders
What’s in any `.pht` files (text files, each line of which contains a path)
Python interpreter combines all the values above into one list and puts it into `sys.path` variable.

If search doesn't return a result after this step - `sys.meta_path` processing reaches the end of its list without returning a specifications object, import fails and `ModuleNotFoundError` exception is raised.

But if module’s specifications object is obtained, new module object (`types.ModuleType`) will be created and reference to the object will be added to `sys.modules`. After that module’s code will be executed.

Execution understands that at the import stage Python performs one iteration to analyse syntax of `.py` module from the top to bottom and generates executable byte-code. When Python meets function or class in a module being imported for the first time:
function: interpreter compiles body of the function and binds function object with the global name. Body of the function is not executed.
class: interpreter executes body of the class, even of nested classes. Defines properties and methods of the class. Then the class object itself is getting built.

Simplified example of what Python does to import a module:

<script src="https://gist.github.com/tugotron/aedd0de2704f9791521172a42786f7f7.js"></script>
<br>

## Comparison of import statements

Let’s consider import process once again comparing different import statements.

<style type="text/css">
.tg  {border-collapse:collapse;border-spacing:0;border-color:#ccc;}
.tg td{font-family:Arial, sans-serif;font-size:14px;padding:10px 5px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;border-color:#ccc;color:#333;background-color:#fff;}
.tg th{font-family:Arial, sans-serif;font-size:14px;font-weight:normal;padding:10px 5px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;border-color:#ccc;color:#333;background-color:#f0f0f0;}
.tg .tg-fzdr{border-color:#c0c0c0;text-align:center;vertical-align:top}
.tg .tg-7seq{background-color:#f9f9f9;border-color:#c0c0c0;text-align:left;vertical-align:top}
.tg .tg-2elc{background-color:#f9f9f9;border-color:#c0c0c0;text-align:center;vertical-align:top}
.tg .tg-wo29{border-color:#c0c0c0;text-align:left;vertical-align:top}
</style>

<table class="tg">
  <tr>
    <th class="tg-fzdr" colspan="4">File: module1.py</th>
  </tr>
  <tr>
    <td class="tg-7seq">import math</td>
    <td class="tg-7seq">import math as r_math</td>
    <td class="tg-7seq">from math import sqrt</td>
    <td class="tg-7seq">from math import sqrt as r_sqrt</td>
  </tr>
  <tr>
    <td class="tg-fzdr" colspan="4">Check if "math" is in sys.modules</td>
  </tr>
  <tr>
    <td class="tg-2elc" colspan="4">if yes - return the reference (sys.modules['math']), if not - load it and insert the reference</td>
  </tr>
  <tr>
    <td class="tg-wo29">Add symbol math to module1’s global namespace</td>
    <td class="tg-wo29">Add symbol r_math to module1’s global namespace</td>
    <td class="tg-wo29">Add symbol sqrt to module1’s global namespace</td>
    <td class="tg-wo29">Add symbol r_sqrt to module1’s global namespace</td>
  </tr>
  <tr>
    <td class="tg-2elc" colspan="4">If the symbol already exists - update reference</td>
  </tr>
  <tr>
    <td class="tg-wo29">In [1]: globals()['math']<br>
    Out[1]: &lt;module 'math' from '/Users/.../python3.6/lib-dynload/math.cpython-36m-darwin.so'&gt;
  </td>
    <td class="tg-wo29">In [2]: globals()['r_math']<br>
    Out[2]: &lt;module 'math' from '/Users/.../python3.6/lib-dynload/math.cpython-36m-darwin.so'&gt;
  </td>
    <td class="tg-wo29">In [3]: globals()['sqrt']<br>
    Out[3]: &lt;function math.sqrt&gt;
  </td>
    <td class="tg-wo29">In [4]: globals()['r_sqrt'] <br>
    Out[4]: &lt;function math.sqrt&gt;
  </td>
  </tr>
  <tr>
    <td class="tg-7seq"></td>
    <td class="tg-2elc" colspan="3">"math" is not in globals()</td>
  </tr>
</table>
<br>
<table class="tg">
  <tr>
    <th class="tg-fzdr" width="25%"></th>
    <th class="tg-wo29">import math</th>
    <th class="tg-wo29">from math import sqrt</th>
  </tr>
  <tr>
    <td class="tg-wo29">import</td>
    <td class="tg-fzdr" colspan="2">same amount of work</td>
  </tr>
  <tr>
    <td class="tg-wo29">call</td>
    <td class="tg-wo29">math.sqrt(2)<br>Needs to find “sqrt” symbol in “math” namespace</td>
    <td class="tg-wo29">sqrt(2)<br>“sqrt” already is in namespace and it directly points to the function</td>
  </tr>
  <tr>
    <td class="tg-fzdr"></td>
    <td class="tg-fzdr" colspan="2">Average dict lookup complexity is <a href="https://wiki.python.org/moin/TimeComplexity" target="blank">O(1)</a>.<br>So, there’s no significant difference!</td>
  </tr>
</table>

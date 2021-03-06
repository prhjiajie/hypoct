Tutorial
========

We now present a tutorial on using the Python interface to hypoct. From here on, we will therefore assume that hypoct has been properly installed along with its Python wrapper; if this is not the case, please go back to :doc:`install`. The choice to cover only the Python interface is merely for convenience. For help regarding the Fortran or C interfaces, please consult the corresponding source code and driver programs.

Overview
--------

The Python interface is located in the directory ``python``, which contains the directory ``hypoct``, organizing the main Python package, and ``hypoct_python.so``, the F2PY-ed Fortran library. The file ``hypoct_python.so`` contains all wrapped routines and is imported by :mod:`hypoct`, which creates a somewhat more convenient (but still pretty bare-bones) object-oriented interface around it. For details on the Python modules, please see the :doc:`api`; for details on data formats, please refer to the Fortran source code.

We will now step through the process of running a program calling hypoct from Python, following the Python driver program as a guide.

Initializing
------------

The first step is to import :mod:`hypoct` by issuing the command::

>>> import hypoct

at the Python prompt. This should work if you are in the ``python`` directory; otherwise, you may have to first type something like:

>>> import sys
>>> sys.path.append(/path/to/hypoct/python/)

in order to tell Python where to look.

Now let's generate some data. As an example, we consider points distributed uniformly on the unit circle::

>>> import numpy as np
>>> n = 100
>>> theta = np.linspace(0, 2*np.pi, n+1)[:n]
>>> x = np.array([np.cos(theta), np.sin(theta)], order='F')

Note the use of the flag ``order='F'`` in :func:`numpy.array`. This instantiates the array in Fortran-contiguous order and is important for avoiding data copying when passing to the backend.

Building the tree
-----------------

Building a tree can be as easy as typing::

>>> tree = hypoct.Tree(x)

Of course, this uses only the default options, e.g., sorting with a maximum occupany parameter of one, treating the data as zero-dimensional points, and no control over the extent of the root. To specify any of these, we can use keyword arguments as explained in :meth:`hypoct.Tree`. We also outline these briefly below.

Adaptivity setting
..................

To switch the build mode from adaptive to uniform subdivision (i.e., all leaves are divided if any one of them violates the occupany condition), enter::

>>> tree = hypoct.Tree(x, adap='u')

The default is ``adap='a'``.

Tree depth
..........

To control the level of subdivision, we can set the maximum leaf occupancy using the ``occ`` keyword. For example, to subdivide until all leaves contain no more than five points, we can type::

>>> tree = hypoct.Tree(x, occ=5)

The default is ``occ=1``.

It is also possible to set the maximum tree depth explicitly using the ``lvlmax`` keyword, e.g.::

>>> tree = hypoct.Tree(x, lvlmax=3)

The root is denoted as level zero. The default is ``lvlmax=-1``, which specifies no maximum depth. Both ``occ`` and ``lvlmax`` can be employed together, with ``lvlmax`` setting a hard limit on the tree.

Element setting
...............

To treat the points as elements, each with a size, first create an array containing the size of each point, then build the tree using the ``elem`` and ``siz`` keywords. For instance, to consider the points as elements each of size 0.1, write::

>>> tree = hypoct.Tree(x, elem='e', siz=0.1)

where we have used the shorthand that if ``siz`` is a single number, then it is automatically expanded into an array of the appropriate size. Similarly, to build a tree on "sparse elements", write::

>>> tree =  hypoct.Tree(x, elem='s', siz=0.1)

The defaults are ``elem='p'``, corresponding to points, and ``siz=0``.

Root extent
...........

The extent of the root node can be specified using the ``ext`` keyword, e.g.,

>>> tree = hypoct.Tree(x, ext=[10, 0])

This tells the code to set the length of the root along the first dimension to 10; its length along the second dimension is calculated from the data (since the corresponding entry is nonpositive). This is often useful if there is some external parameter governing the problem geometry, for example, periodicity conditions. Like ``siz``, ``ext`` can also be given as a single number, in which case it is automatically expanded as appropriate. The default is ``ext=0``.

Remarks
.......

All options can be combined with each other. The output is stored as a :class:`hypoct.Tree` instance, which is a thin wrapper for the arrays output from Fortran. On our machine, running::

>>> tree = hypoct.Tree(x)
>>> tree.lvlx

gives::

  array([[  0,   1,   5,  17,  45,  97, 177, 193],
         [  6,   0,   3,   3,   3,   3,   3,   3]], dtype=int32)

which indicates that the tree has 6 levels (beyond the root at level 0) with 193 nodes in total. See the Fortran source code for details.

Generating auxiliary data
-------------------------

The base tree output is stored in a rather spartan manner; it contains only the bare minimum necessary to reconstruct the data for the entire tree. This is not always convenient and it is sometimes useful to have the data in a more easily accessible form. For instance, the base tree representation contains only parent and child identifier information that only really allows you to traverse a tree from the bottom up. To traverse a tree from the top down, we have to generate child pointers, which we can do via::

>>> tree.generate_child_data()

We can also generate geometry information (center and extent) for each node by using::

>>> tree.generate_geometry_data()

These commands create the arrays ``tree.chldp``, and ``tree.l``, and ``tree.ctr``, respectively.

Finding neighbors
-----------------

To find the neighbors of each node, type::

>>> tree.find_neighbors()

which creates the neighbor index and pointer arrays ``tree.nbori`` and ``tree.nborp``, respectively. The method also accepts the keyword ``per`` indicating whether the root is periodic in a given dimension. For example, to impose that the root is periodic in the first but not the second dimension, set::

>>> tree.find_neighbors(per=[True, False])

It is worth emphasizing that the size of the unit cell cannot be directly controlled here; for this, use the ``ext`` keyword in :meth:`hypoct.Tree`. As with the ``siz`` and ``ext`` keywords for :meth:`hypoct.Tree`, we can also use shorthand by writing just, e.g.::

>>> tree.find_neighbors(per=True)

for double periodicity. The default is ``per=False``.

The method :meth:`hypoct.Tree.find_neighbors` requires that the child data from :meth:`hypoct.Tree.generate_child_data` have already been generated; if this is not the case, then this is done automatically.

.. note::
   Recall that neighbors are defined differently for points vs. elements as described briefly in :doc:`intro`.

Getting interaction lists
-------------------------

Interaction lists are often used in fast multipole-type algorithms to systematically cover the far field. To get interaction lists for all nodes, type::

>>> tree.get_interaction_lists()

This command requires that the neighbor data from :meth:`hypoct.Tree.find_neighbors` have already been generated; if this is not the case, then this is done automatically using default settings. Outputs include the index and pointer arrays ``tree.ilsti`` and ``tree.ilstp``, respectively.

Searching the tree
------------------

It is often also useful to be able to search the tree for a given set of points. This can be done via::

>>> trav = tree.search(x)

where ``x`` is the set of points to search for. The output ``trav`` is an array that records the tree traversal history for each point: the node containing the point ``x[:,i]`` at level ``j`` has index ``trav[i,j]``; if no such node exists, then ``trav[i,j] = 0``. By default, the tree is traversed fully from top to bottom. To limit the maximum tree depth searched, use the keyword ``mlvl``.

If we have a tree on elements, then we can also attach a size to each point using the keyword ``siz``. Each node in the tree traversal array, if it exists, must fully contain the point based on its size. The default is ``siz=0``.

This command requires that child and geometry data have already been generated; if this is not the case, then this is done automatically.

Putting it all together
-----------------------

A complete example program for building a tree and generating all auxiliary data is given as follows::

  import hypoct, numpy as np

  # initialize points
  n = 100
  theta = np.linspace(0, 2*np.pi, n+1)[:n]
  x = np.array([np.cos(theta), np.sin(theta)], order='F')

  # build tree
  tree = hypoct.Tree(x, occ=4)
  tree.generate_child_data()
  tree.generate_geometry_data()
  tree.find_neighbors()
  tree.get_interaction_lists()

This is a slightly modified and abridged version of the driver program ``examples/hypoct_driver.py``.

Viewing trees in 1D and 2D
--------------------------

Trees in 1D and 2D can be viewed graphically using the :class:`hypoct.tools.TreeViewer` class. To use the viewer, type::

>>> from hypoct.tools import TreeViewer
>>> view = TreeViewer(tree)
>>> view.draw_interactive()

This brings up an interactive session where each node in the tree is highlighted in turn, displaying its geometry, contained points, and neighbor and interaction list information, if available. Press ``Enter`` to step through the tree. All plot options can be controlled using :mod:`matplotlib`-style keywords.

.. note::
   The :class:`hypoct.tools.TreeViewer` class was written originally for 2D trees. It was extended to 1D trees by trivially lifting into 2D.
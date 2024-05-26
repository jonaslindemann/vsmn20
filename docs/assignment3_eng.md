# Worksheet 3

!!! note "Important"

    When **...** appears in the program examples, this indicates that there is no code that you yourselves must add. Variables and data structures are only examples. Depending on the type of problem you may need other data structures than those described in the code examples.

## General

In this worksheet contains the following elements:

  1. The classes implemented in sheet 2 are adapted to implementa geometric model that describes the calculation model parametrically.
  2. load and save functions must be adapted to handle the geometric model.
  2. grid generation is implemented using **GMSH** and the results visualized.
  3. Visualization of the model to be made with **calfem.vis_mpl** functions.

## Geometric model

In this worksheet, we will update our classes to manage a geometric model that is the basis for the mesh generation in our **ModelSolver**-class. Instead of defining the model in terms of nodes and elements, we define it instead with parameters such as width and height.

In this worksheet, we use some additional Calfem modules. Add the following modules in the module where your classes are defined:

``` py hl_lines="4-6"
import json, math, sys

import calfem.core as cfc
import calfem.geometry as cfg
import calfem.mesh as cfm
import calfem.vis_mpl as cfv
import calfem.utils as cfu

import numpy as np
import tabulate as tb
``` 

## Updating the ModelParams-class

**ModelParams**-class must be updated to handle a parametric model described using parameters instead of defining the problem at the element level. For the groundwater problem, parameters **w** **t** , **d**  and **h** are used to describe the geometry and **EP**  **kx** and **ky** to describe elements properties. Variables for loads and boundary conditions can be maintained. The difference now is that instead of storing the degrees of freedom and value we now store the boundary condition marker and value.

All variables such as **coord** and **edof** describing the input of element level to be moved to the **ModelSolver** - class because they will be created when the elements are generated in the solver.

The geometry will be defined with the classes and functions of  **calfem.geometry**. To ensure that the **ModelSolver** method **execute()** will get an up-to-date geometry definition we add a method **geometry()** in the **ModelParams**-class, which will return a **cfg.Geometry**-instance. An example of how this method might look like the following example:

``` py
class ModelParams(object):
    """Class defining our model parameters"""
    ...
    
    def geometry(self):
        """Skapa en geometri instans baserat på definierade parametrar"""
        
        # --- Create a geometry instance to store the geometry
        #     description
        
        g = cfg.Geometry()
        
        # --- Create shorter variable references to class attributes
        
        w = self.w
        h = self.h
        t = self.t
        d = self.d
        
        # --- Points in the geometry are created with the .point() method
        
        g.point([0, 0])
        g.point([w, 0])
        g.point([w, h])
        
        ...
        
        # --- Lines and splies are created with the .spline method.
        
        g.spline([0, 1])            
        g.spline([1, 2])           
        g.spline([2, 3], marker=...) # <-- Use marker to define
                                     #     lines that will have  
                                     #     boundary conditions 
                                     #     and loads.
        
        ...
        
        # --- Surface defining what defines the geometry
        
        g.surface([0,1, ... ,6,7])

        # --- Return the generated geometry.
        
        return g
```

## Updating the ModelSolver-class

In the **ModelSolver**-class, we must add a call to the mesh generator in the **calfem.mesh**-module, **GmshMeshGenerator**. This will give us the element coordinates, topology, and variables that can be used for the connection between geometry and element mesh. An example of how this might look in the **execute()**-method is shown below:

``` py
class ModelSolver(object):
    """Class implementing the solver for our model."""

    ...
                
    def execute(self):
        """Perform finite element computation."""
        
        # --- Create references (shortcuts) to input data variables
        
        version = self.model_params.version
        ep = self.model_params.ep

        ...
        
        
        # --- Call the mode
        
        geometry = self.model_params.geometry()        
        
        # --- Mesh generation
        
        el_type = 3        # <-- four node element flw2i4e
        dofs_per_node = 1  # <-- scalar problem
        
        mesh = cfm.GmshMesh(geometry)
        mesh.el_size_factor = 0.5     # <-- max area for elements
        mesh.el_type = el_type
        mesh.dofs_per_node = dofs_per_node
        mesh.return_boundary_elements = True
        
        coords, edof, dofs, bdofs, element_markers, boundary_elements = mesh.create()
```
            
Element generation and assemblation do not need to be changed from worksheet 2. However, one must update the handling of loads and boundary conditions, as we are now working load markers instead of degrees of freedom. Use the functions **apply_bc(...)** and **apply_force_total(...)**/**apply_tracion_linear_element(...)** available in the **calfem.utils** module.

Solving equations and calculating element forces do not need to be changed. However, we need to store the generated variables **coord**, **edof**, **geometry** and other output variables needed to visualize the results.

!!! note "Tip"

    It may also be useful to define a variable in the **ModelParams** - class to set the maximum size of the generated elements such as **el_size_factor**, which can then be assigned to **cfm.GmshMesh** - class attribute **el_size_factor**.

## Using the parametric model

The idea of the parametric description of the problem is that a user easily be able to specify their area of concern in the code without having to go into the module for the model. This is shown in the following code:

``` py
# -*- coding: utf-8 -*-

import flowmodel as fm

if __name__ == "__main__":
    
    model_params = fm.ModelParams()

    model_params.w = 100.0
    model_params.h = 10.0
    model_params.d = 5.0
    model_params.t = 0.5
    model_params.kx = 20.0
    model_params.ky = 20.0
    
    ...
```

In this way, one can also easily study the effect of, an increasing depth of the tongue in the ground and find out how this affects the flow, as shown below:

``` py
# -*- coding: utf-8 -*-

import flowmodel as fm
import numpy as np

if __name__ == "__main__":
    
    dRange = np.linspace(3.0, 7.0, 10).tolist()
    
    for d in dRange:
    
        print("-------------------------------------------")    
        print("Simulating d = ", d)
    
        model_params = fm.ModelParams()
    
        model_params.w = 100.0
        model_params.h = 10.0
        model_params.d = d
        model_params.t = 0.5
        model_params.kx = 20.0
        model_params.ky = 20.0
        
        model_results = fm.ModelResults()
    
        solver = fm.ModelSolver(model_params, model_results)
        solver.execute()
        
        print("Max flow = ", np.max(model_result.maxFlow))        
```

## ModelReport class

In the **ModelReport**-class, we need to add the ability to print the geometry description.

## Visualisation with a new class ModelVisualisation

In worksheet 2 the only output from the calculations where the printout from the **ModelResult**-class. To interpret the results in text format can be difficult. We must therefore use routines from the CALFEM visualization module **calfem.vis**. The visualization will be implemented in the new class **ModeVisualization**. This class has the same input parameters as the **ModelResult**-class with references to instances of **ModelParams** and **ModelResult**.

There are a number of visualization features in Calfem. In this sheet, the following visualizations are implemented:

 * Geometry - draw_geometry(...)
 * Generated network - draw_mesh(...)
 * Deformed networks - draw_mesh(...) (for example voltage)
 * Element Values ​​- draw_element_values​​(...)
 * Node values ​​- draw_nodal_values​(...)
 
Documentation for these procedures, see the user manual for [mesh generation routines](https://calfem-for-python.readthedocs.io/en/latest/calfem_mesh_guide.html).
      
The following code shows how the class can be implemented with a method for visualising the model geometry.

``` py
class ModelVisualisation(object):
    def __init__(self, model_params, model_result):
        self.model_params = model_params
        self.model_result = model_result
        
    def show(self):
        
        geometry = self.model_result.geometry
        a = self.model_result.a
        max_flow = self.model_result.max_flow
        coords = self.model_result.coords
        edof = self.model_result.edof
        dofs_per_node = self.model_result.dofs_per_node
        el_type = self.model_result.el_type
        
        cfv.figure() 
        cfv.draw_geometry(geometry, title="Geometry")
        
        ...
                    
    def wait(self):
        """This method make sure the windows are kept updated."""

        cfv.show_and_wait()
```
            
The **ModelVisualiation**-class is then added to the main program in the code shown:

``` py
# -*- coding: utf-8 -*-

import flowmodel as fm

if __name__ == "__main__":
    
    model_params = fm.ModelParams()

    model_result = fm.OutputData()

    solver = fm.Solver(model_params, model_result)
    solver.execute()

    report = fm.Report(model_params, model_result)
    print(report)
    
    vis = fm.ModelVisualisation(model_params, model_result)
    vis.show()
    vis.wait()        
```

**vis.wait()** must be called last in the main program, since this function does not return until the final visualization window closes.

## Submission and reporting

It is to be done in this worksheet are:

 * Change **ModelPArams**-class to describe the problem parametrically according to the examples described at the end of this sheet. Create a method **geometry()** that returns a **cfg.Geometry**-instance with geometry defined based on the parameter description.
 * Update the **ModelSolver**-class so that this creates an element mesh using **cfm.GmshMeshGenerator**-class. Save maximum flows / maxspänningar (von Mises) and store these in the **ModelResult**-class.
 * Complete the implementation of the **ModelVisualization**-class to handle visualisation of geometry, element mesh, element flows and node values.
 * Makes a parameter study wherein one of the parameters is varied and maximum flow/max stress is plotted relative to the selected parameter. Create a new main program for parameter study.

The submission shall consist of a zip file (or other archive format) consisting of:

 * All the Python files. (.py files)
 * An example of a saved file JSON.
 * Printing from program execution.
 * Printing of the parameter study.

## Example problems

### Grundvattenströmning
 
![case1](images/gw.svg)

h = 10.0, w = 100.0, d = 5.0, t = 0.5

k_x = k_y = 20 m/dag

### Tvådimensionell värmeledning
 
![case2](images/temp.svg)

h = 0.1, w = 0.1, a = 0.01, b = 0.01, x = 0.01, y = 0.01

lambda_x = lambda_y = 1.7 W/m C

### Plan skiva
 
![case3](images/stress.svg)

h = 0.1, w = 0.3, a = 0.05, b = 0.025

E = 2.08e10, ν = 0.2, t = 0.15

q = 100 kN / m

## Solution to problems

=== "Ground water problem"

    ```
    -------------------------------------------------------------
    -------------- Model input ----------------------------------
    -------------------------------------------------------------

    Model parameters:

    +---------------+---------+
    | Parameter     |   Value |
    |---------------+---------|
    | w [m]         |     100 |
    | h [m]         |      10 |
    | d [m]         |       5 |
    | t [m]         |     0.5 |
    | thickness [m] |       1 |
    | kx [m/day]    |      20 |
    | ky [m/day]    |      20 |
    +---------------+---------+

    Model boundary conditions:

    +----------+--------------------+
    |   Marker |   Piezometric head |
    |----------+--------------------|
    |       10 |                 10 |
    |       20 |                  0 |
    +----------+--------------------+

    -------------------------------------------------------------
    -------------- Results --------------------------------------
    -------------------------------------------------------------

    Model info:

    +---------------------+------+
    | Max element size:   |    1 |
    | Dofs per node:      |    1 |
    | Element type:       |    3 |
    | Number of dofs:     | 1395 |
    | Number of elements: | 1277 |
    | Number of nodes:    | 1395 |
    +---------------------+------+

    Summary of results:

    +------------------------+---------+---------+
    | Max piezometric head:  |  10.000 | [m]     |
    | Min piezometric head:  |   0.000 | [m]     |
    | Max boundary flow:     |  10.413 | [m/day] |
    | Min boundary flow:     | -10.388 | [m/day] |
    | Sum boundary flow:     |  -0.000 | [m/day] |
    | Max resulting gw flow: |  40.134 | [m/day] |
    | Min resulting gw flow: |   0.001 | [m/day] |
    +------------------------+---------+---------+

    Results per node:

    +------+-------+-----------+------------+--------+-----------+
    |    N |   dof |   x_coord |    y_coord |   P.h. |      q BC |
    |      |       |       [m] |        [m] |    [m] |   [m/day] |
    |------+-------+-----------+------------+--------+-----------|
    |    1 |     1 |       0.0 |        0.0 |  9.996 |     0.000 |
    |    2 |     2 |     100.0 |        0.0 |  0.004 |     0.000 |
    |    3 |     3 |     100.0 |       10.0 |  0.000 |    -0.007 |
    |    4 |     4 |      50.2 |       10.0 |  0.000 |    -5.593 |
    |    5 |     5 |      50.2 |        5.0 |  4.154 |     0.000 |
    |    6 |     6 |      49.8 |        5.0 |  5.851 |     0.000 |
    |    7 |     7 |      49.8 |       10.0 | 10.000 |     5.648 |
    |    8 |     8 |       0.0 |       10.0 | 10.000 |     0.007 |
    |    9 |     9 |       1.0 |        0.0 |  9.996 |     0.000 |
    |   10 |    10 |       2.0 |        0.0 |  9.996 |    -0.000 |
    |   11 |    11 |       3.0 |        0.0 |  9.996 |     0.000 |
    |   12 |    12 |       4.0 |        0.0 |  9.995 |     0.000 |
    |   13 |    13 |       5.0 |        0.0 |  9.995 |     0.000 |
    |   14 |    14 |       6.0 |        0.0 |  9.994 |     0.000 |
    |   15 |    15 |       7.0 |        0.0 |  9.993 |     0.000 |
    |   16 |    16 |       8.0 |        0.0 |  9.992 |     0.000 |
    |   17 |    17 |       9.0 |        0.0 |  9.991 |     0.000 |
    |   18 |    18 |      10.0 |        0.0 |  9.990 |    -0.000 |
    |   19 |    19 |      11.0 |        0.0 |  9.988 |     0.000 |
    |   20 |    20 |      12.0 |        0.0 |  9.986 |    -0.000 |
    |   21 |    21 |      13.0 |        0.0 |  9.984 |    -0.000 |
    |   22 |    22 |      14.0 |        0.0 |  9.981 |     0.000 |
    |   23 |    23 |      15.0 |        0.0 |  9.979 |     0.000 |
    |   24 |    24 |      16.0 |        0.0 |  9.974 |     0.000 |
    |   25 |    25 |      17.0 |        0.0 |  9.971 |     0.000 |
    |   26 |    26 |      18.0 |        0.0 |  9.965 |     0.000 |
    |   27 |    27 |      19.0 |        0.0 |  9.960 |     0.000 |
    |   28 |    28 |      20.0 |        0.0 |  9.952 |     0.000 |
    |   29 |    29 |      21.0 |        0.0 |  9.946 |     0.000 |
    |   30 |    30 |      22.0 |        0.0 |  9.934 |    -0.000 |
    |   31 |    31 |      23.0 |        0.0 |  9.926 |     0.000 |
    |   32 |    32 |      24.0 |        0.0 |  9.910 |    -0.000 |
    |   33 |    33 |      25.0 |        0.0 |  9.898 |     0.000 |
    |   34 |    34 |      26.0 |        0.0 |  9.876 |    -0.000 |
    |   35 |    35 |      27.0 |        0.0 |  9.860 |    -0.000 |
    |   36 |    36 |      28.0 |        0.0 |  9.830 |     0.000 |
    |   37 |    37 |      29.0 |        0.0 |  9.808 |     0.000 |
    |   38 |    38 |      30.0 |        0.0 |  9.768 |     0.000 |
    |   39 |    39 |      31.0 |        0.0 |  9.737 |    -0.000 |
    |   40 |    40 |      32.0 |        0.0 |  9.682 |     0.000 |
    |   41 |    41 |      33.0 |        0.0 |  9.639 |     0.000 |
    |   42 |    42 |      34.0 |        0.0 |  9.565 |     0.000 |
    |   43 |    43 |      35.0 |        0.0 |  9.504 |    -0.000 |
    |   44 |    44 |      36.0 |        0.0 |  9.404 |    -0.000 |
    |   45 |    45 |      37.0 |        0.0 |  9.320 |    -0.000 |
    |   46 |    46 |      38.0 |        0.0 |  9.184 |     0.000 |
    |   47 |    47 |      39.0 |        0.0 |  9.066 |     0.000 |
    |   48 |    48 |      40.0 |        0.0 |  8.883 |    -0.000 |
    |   49 |    49 |      41.0 |        0.0 |  8.719 |    -0.000 |
    |   50 |    50 |      42.0 |        0.0 |  8.471 |    -0.000 |
    |   51 |    51 |      43.0 |        0.0 |  8.243 |     0.000 |
    |   52 |    52 |      44.0 |        0.0 |  7.909 |     0.000 |
    |   53 |    53 |      45.0 |        0.0 |  7.592 |    -0.000 |
    |   54 |    54 |      46.0 |        0.0 |  7.151 |     0.000 |
    |   55 |    55 |      47.0 |        0.0 |  6.718 |    -0.000 |
    |   56 |    56 |      48.0 |        0.0 |  6.168 |    -0.000 |
    |   57 |    57 |      49.0 |        0.0 |  5.618 |     0.000 |
    |   58 |    58 |      50.0 |        0.0 |  4.999 |     0.000 |
    |   59 |    59 |      51.0 |        0.0 |  4.397 |    -0.000 |
    |   60 |    60 |      52.0 |        0.0 |  3.834 |     0.000 |
    |   61 |    61 |      53.0 |        0.0 |  3.270 |     0.000 |
    |   62 |    62 |      54.0 |        0.0 |  2.845 |     0.000 |
    |   63 |    63 |      55.0 |        0.0 |  2.407 |    -0.000 |
    |   64 |    64 |      56.0 |        0.0 |  2.101 |     0.000 |
    |   65 |    65 |      57.0 |        0.0 |  1.753 |     0.000 |
    |   66 |    66 |      58.0 |        0.0 |  1.524 |     0.000 |
    |   67 |    67 |      59.0 |        0.0 |  1.279 |    -0.000 |
    |   68 |    68 |      60.0 |        0.0 |  1.115 |     0.000 |
    |   69 |    69 |      61.0 |        0.0 |  0.935 |     0.000 |
    |   70 |    70 |      62.0 |        0.0 |  0.816 |     0.000 |
    |   71 |    71 |      63.0 |        0.0 |  0.684 |    -0.000 |
    |   72 |    72 |      64.0 |        0.0 |  0.594 |     0.000 |
    |   73 |    73 |      65.0 |        0.0 |  0.498 |     0.000 |
    |   74 |    74 |      66.0 |        0.0 |  0.433 |    -0.000 |
    |   75 |    75 |      67.0 |        0.0 |  0.363 |    -0.000 |
    |   76 |    76 |      68.0 |        0.0 |  0.317 |     0.000 |
    |   77 |    77 |      69.0 |        0.0 |  0.265 |    -0.000 |
    |   78 |    78 |      70.0 |        0.0 |  0.232 |     0.000 |
    |   79 |    79 |      71.0 |        0.0 |  0.194 |    -0.000 |
    |   80 |    80 |      72.0 |        0.0 |  0.169 |    -0.000 |
    |   81 |    81 |      73.0 |        0.0 |  0.141 |    -0.000 |
    |   82 |    82 |      74.0 |        0.0 |  0.123 |    -0.000 |
    |   83 |    83 |      75.0 |        0.0 |  0.103 |     0.000 |
    |   84 |    84 |      76.0 |        0.0 |  0.090 |    -0.000 |
    |   85 |    85 |      77.0 |        0.0 |  0.075 |    -0.000 |
    |   86 |    86 |      78.0 |        0.0 |  0.066 |    -0.000 |
    |   87 |    87 |      79.0 |        0.0 |  0.055 |     0.000 |
    |   88 |    88 |      80.0 |        0.0 |  0.048 |     0.000 |
    |   89 |    89 |      81.0 |        0.0 |  0.040 |     0.000 |
    |   90 |    90 |      82.0 |        0.0 |  0.035 |     0.000 |
    |   91 |    91 |      83.0 |        0.0 |  0.029 |     0.000 |
    |   92 |    92 |      84.0 |        0.0 |  0.026 |     0.000 |
    |   93 |    93 |      85.0 |        0.0 |  0.022 |    -0.000 |
    |   94 |    94 |      86.0 |        0.0 |  0.019 |     0.000 |
    |   95 |    95 |      87.0 |        0.0 |  0.016 |    -0.000 |
    |   96 |    96 |      88.0 |        0.0 |  0.014 |     0.000 |
    |   97 |    97 |      89.0 |        0.0 |  0.012 |    -0.000 |
    |   98 |    98 |      90.0 |        0.0 |  0.010 |     0.000 |
    |   99 |    99 |      91.0 |        0.0 |  0.009 |    -0.000 |
    |  100 |   100 |      92.0 |        0.0 |  0.008 |    -0.000 |
    |  101 |   101 |      93.0 |        0.0 |  0.007 |     0.000 |
    |  102 |   102 |      94.0 |        0.0 |  0.006 |    -0.000 |
    |  103 |   103 |      95.0 |        0.0 |  0.005 |    -0.000 |
    |  104 |   104 |      96.0 |        0.0 |  0.005 |     0.000 |
    |  105 |   105 |      97.0 |        0.0 |  0.004 |    -0.000 |
    |  106 |   106 |      98.0 |        0.0 |  0.004 |    -0.000 |
    |  107 |   107 |      99.0 |        0.0 |  0.004 |    -0.000 |
    |  108 |   108 |     100.0 |        1.0 |  0.004 |     0.000 |
    |  109 |   109 |     100.0 |        2.0 |  0.004 |    -0.000 |
    |  110 |   110 |     100.0 |        3.0 |  0.004 |     0.000 |
    |  111 |   111 |     100.0 |        4.0 |  0.003 |     0.000 |
    |  112 |   112 |     100.0 |        5.0 |  0.003 |    -0.000 |
    |  113 |   113 |     100.0 |        6.0 |  0.002 |     0.000 |
    |  114 |   114 |     100.0 |        7.0 |  0.002 |     0.000 |
    |  115 |   115 |     100.0 |        8.0 |  0.001 |     0.000 |
    |  116 |   116 |     100.0 |        9.0 |  0.001 |     0.000 |
    |  117 |   117 |      99.0 |       10.0 |  0.000 |    -0.013 |
    |  118 |   118 |      98.0 |       10.0 |  0.000 |    -0.014 |
    |  119 |   119 |      97.0 |       10.0 |  0.000 |    -0.014 |
    |  120 |   120 |      96.0 |       10.0 |  0.000 |    -0.016 |
    |  121 |   121 |      95.0 |       10.0 |  0.000 |    -0.017 |
    |  122 |   122 |      94.0 |       10.0 |  0.000 |    -0.019 |
    |  123 |   123 |      93.0 |       10.0 |  0.000 |    -0.021 |
    |  124 |   124 |      92.0 |       10.0 |  0.000 |    -0.025 |
    |  125 |   125 |      91.0 |       10.0 |  0.000 |    -0.028 |
    |  126 |   126 |      90.0 |       10.0 |  0.000 |    -0.032 |
    |  127 |   127 |      89.1 |       10.0 |  0.000 |    -0.036 |
    |  128 |   128 |      88.1 |       10.0 |  0.000 |    -0.045 |
    |  129 |   129 |      87.1 |       10.0 |  0.000 |    -0.050 |
    |  130 |   130 |      86.1 |       10.0 |  0.000 |    -0.058 |
    |  131 |   131 |      85.1 |       10.0 |  0.000 |    -0.067 |
    |  132 |   132 |      84.1 |       10.0 |  0.000 |    -0.080 |
    |  133 |   133 |      83.1 |       10.0 |  0.000 |    -0.091 |
    |  134 |   134 |      82.1 |       10.0 |  0.000 |    -0.110 |
    |  135 |   135 |      81.1 |       10.0 |  0.000 |    -0.125 |
    |  136 |   136 |      80.1 |       10.0 |  0.000 |    -0.149 |
    |  137 |   137 |      79.1 |       10.0 |  0.000 |    -0.166 |
    |  138 |   138 |      78.1 |       10.0 |  0.000 |    -0.207 |
    |  139 |   139 |      77.1 |       10.0 |  0.000 |    -0.236 |
    |  140 |   140 |      76.1 |       10.0 |  0.000 |    -0.278 |
    |  141 |   141 |      75.1 |       10.0 |  0.000 |    -0.321 |
    |  142 |   142 |      74.1 |       10.0 |  0.000 |    -0.368 |
    |  143 |   143 |      73.1 |       10.0 |  0.000 |    -0.434 |
    |  144 |   144 |      72.1 |       10.0 |  0.000 |    -0.514 |
    |  145 |   145 |      71.1 |       10.0 |  0.000 |    -0.586 |
    |  146 |   146 |      70.1 |       10.0 |  0.000 |    -0.724 |
    |  147 |   147 |      69.2 |       10.0 |  0.000 |    -0.823 |
    |  148 |   148 |      68.2 |       10.0 |  0.000 |    -0.970 |
    |  149 |   149 |      67.2 |       10.0 |  0.000 |    -1.120 |
    |  150 |   150 |      66.2 |       10.0 |  0.000 |    -1.340 |
    |  151 |   151 |      65.2 |       10.0 |  0.000 |    -1.512 |
    |  152 |   152 |      64.2 |       10.0 |  0.000 |    -1.812 |
    |  153 |   153 |      63.2 |       10.0 |  0.000 |    -2.067 |
    |  154 |   154 |      62.2 |       10.0 |  0.000 |    -2.518 |
    |  155 |   155 |      61.2 |       10.0 |  0.000 |    -2.853 |
    |  156 |   156 |      60.2 |       10.0 |  0.000 |    -3.344 |
    |  157 |   157 |      59.2 |       10.0 |  0.000 |    -3.871 |
    |  158 |   158 |      58.2 |       10.0 |  0.000 |    -4.648 |
    |  159 |   159 |      57.2 |       10.0 |  0.000 |    -5.211 |
    |  160 |   160 |      56.2 |       10.0 |  0.000 |    -6.232 |
    |  161 |   161 |      55.2 |       10.0 |  0.000 |    -7.012 |
    |  162 |   162 |      54.2 |       10.0 |  0.000 |    -8.217 |
    |  163 |   163 |      53.2 |       10.0 |  0.000 |    -8.813 |
    |  164 |   164 |      52.2 |       10.0 |  0.000 |   -10.296 |
    |  165 |   165 |      51.2 |       10.0 |  0.000 |   -10.388 |
    |  166 |   166 |      50.2 |        9.2 |  0.446 |    -0.000 |
    |  167 |   167 |      50.2 |        8.3 |  0.970 |     0.000 |
    |  168 |   168 |      50.2 |        7.5 |  1.400 |     0.000 |
    |  169 |   169 |      50.2 |        6.7 |  2.085 |    -0.000 |
    |  170 |   170 |      50.2 |        5.8 |  2.639 |    -0.000 |
    |  171 |   171 |      50.0 |        5.0 |  5.021 |    -0.000 |
    |  172 |   172 |      49.7 |        5.8 |  7.394 |    -0.000 |
    |  173 |   173 |      49.8 |        6.7 |  7.889 |     0.000 |
    |  174 |   174 |      49.8 |        7.5 |  8.604 |    -0.000 |
    |  175 |   175 |      49.7 |        8.3 |  9.016 |    -0.000 |
    |  176 |   176 |      49.8 |        9.2 |  9.551 |     0.000 |
    |  177 |   177 |      48.8 |       10.0 | 10.000 |    10.413 |
    |  178 |   178 |      47.8 |       10.0 | 10.000 |    10.235 |
    |  179 |   179 |      46.8 |       10.0 | 10.000 |     8.849 |
    |  180 |   180 |      45.8 |       10.0 | 10.000 |     8.197 |
    |  181 |   181 |      44.8 |       10.0 | 10.000 |     6.962 |
    |  182 |   182 |      43.8 |       10.0 | 10.000 |     6.259 |
    |  183 |   183 |      42.8 |       10.0 | 10.000 |     5.246 |
    |  184 |   184 |      41.8 |       10.0 | 10.000 |     4.609 |
    |  185 |   185 |      40.8 |       10.0 | 10.000 |     3.846 |
    |  186 |   186 |      39.8 |       10.0 | 10.000 |     3.400 |
    |  187 |   187 |      38.8 |       10.0 | 10.000 |     2.844 |
    |  188 |   188 |      37.8 |       10.0 | 10.000 |     2.490 |
    |  189 |   189 |      36.8 |       10.0 | 10.000 |     2.073 |
    |  190 |   190 |      35.8 |       10.0 | 10.000 |     1.821 |
    |  191 |   191 |      34.8 |       10.0 | 10.000 |     1.518 |
    |  192 |   192 |      33.8 |       10.0 | 10.000 |     1.334 |
    |  193 |   193 |      32.8 |       10.0 | 10.000 |     1.116 |
    |  194 |   194 |      31.8 |       10.0 | 10.000 |     0.974 |
    |  195 |   195 |      30.8 |       10.0 | 10.000 |     0.817 |
    |  196 |   196 |      29.8 |       10.0 | 10.000 |     0.708 |
    |  197 |   197 |      28.9 |       10.0 | 10.000 |     0.588 |
    |  198 |   198 |      27.9 |       10.0 | 10.000 |     0.519 |
    |  199 |   199 |      26.9 |       10.0 | 10.000 |     0.431 |
    |  200 |   200 |      25.9 |       10.0 | 10.000 |     0.383 |
    |  201 |   201 |      24.9 |       10.0 | 10.000 |     0.316 |
    |  202 |   202 |      23.9 |       10.0 | 10.000 |     0.280 |
    |  203 |   203 |      22.9 |       10.0 | 10.000 |     0.232 |
    |  204 |   204 |      21.9 |       10.0 | 10.000 |     0.204 |
    |  205 |   205 |      20.9 |       10.0 | 10.000 |     0.170 |
    |  206 |   206 |      19.9 |       10.0 | 10.000 |     0.151 |
    |  207 |   207 |      18.9 |       10.0 | 10.000 |     0.123 |
    |  208 |   208 |      17.9 |       10.0 | 10.000 |     0.109 |
    |  209 |   209 |      16.9 |       10.0 | 10.000 |     0.089 |
    |  210 |   210 |      15.9 |       10.0 | 10.000 |     0.079 |
    |  211 |   211 |      14.9 |       10.0 | 10.000 |     0.067 |
    |  212 |   212 |      13.9 |       10.0 | 10.000 |     0.059 |
    |  213 |   213 |      12.9 |       10.0 | 10.000 |     0.049 |
    |  214 |   214 |      11.9 |       10.0 | 10.000 |     0.044 |
    |  215 |   215 |      10.9 |       10.0 | 10.000 |     0.036 |
    |  216 |   216 |       9.9 |       10.0 | 10.000 |     0.033 |
    |  217 |   217 |       9.0 |       10.0 | 10.000 |     0.027 |
    |  218 |   218 |       8.0 |       10.0 | 10.000 |     0.025 |
    |  219 |   219 |       7.0 |       10.0 | 10.000 |     0.021 |
    |  220 |   220 |       6.0 |       10.0 | 10.000 |     0.019 |
    |  221 |   221 |       5.0 |       10.0 | 10.000 |     0.017 |
    |  222 |   222 |       4.0 |       10.0 | 10.000 |     0.016 |
    |  223 |   223 |       3.0 |       10.0 | 10.000 |     0.014 |
    |  224 |   224 |       2.0 |       10.0 | 10.000 |     0.014 |
    |  225 |   225 |       1.0 |       10.0 | 10.000 |     0.013 |
    |  226 |   226 |       0.0 |        9.0 |  9.999 |    -0.000 |
    |  227 |   227 |       0.0 |        8.0 |  9.999 |    -0.000 |
    |  228 |   228 |       0.0 |        7.0 |  9.998 |     0.000 |
    |  229 |   229 |       0.0 |        6.0 |  9.998 |     0.000 |
    |  230 |   230 |       0.0 |        5.0 |  9.997 |    -0.000 |
    |  231 |   231 |       0.0 |        4.0 |  9.997 |     0.000 |
    |  232 |   232 |       0.0 |        3.0 |  9.996 |     0.000 |
    |  233 |   233 |       0.0 |        2.0 |  9.996 |     0.000 |
    |  234 |   234 |       0.0 |        1.0 |  9.996 |     0.000 |
    |  235 |   235 |      50.8 |        5.4 |  3.243 |    -0.000 |
    |  236 |   236 |      49.3 |        5.4 |  6.623 |     0.000 |
    |  237 |   237 |      55.9 |        9.2 |  0.262 |    -0.000 |
    |  238 |   238 |      44.2 |        9.1 |  9.725 |     0.000 |
    |  239 |   239 |      58.3 |        0.8 |  1.417 |     0.000 |
    |  240 |   240 |      41.2 |        0.9 |  8.673 |     0.000 |
    |  241 |   241 |       5.8 |        9.1 |  9.999 |     0.000 |
    |  242 |   242 |      94.5 |        9.2 |  0.001 |    -0.000 |
    |  243 |   243 |       8.8 |        9.2 |  9.999 |    -0.000 |
    |  244 |   244 |      91.5 |        9.2 |  0.001 |    -0.000 |
    |  245 |   245 |      11.3 |        8.9 |  9.998 |     0.000 |
    |  246 |   246 |      88.5 |        9.0 |  0.002 |     0.000 |
    |  247 |   247 |      14.1 |        9.1 |  9.997 |     0.000 |
    |  248 |   248 |      85.1 |        9.1 |  0.003 |    -0.000 |
    |  249 |   249 |      17.6 |        9.3 |  9.996 |     0.000 |
    |  250 |   250 |      82.7 |        9.2 |  0.004 |     0.000 |
    |  251 |   251 |      79.3 |        9.2 |  0.007 |     0.000 |
    |  252 |   252 |      20.2 |        9.2 |  9.994 |     0.000 |
    |  253 |   253 |      76.9 |        9.2 |  0.010 |    -0.000 |
    |  254 |   254 |      23.2 |        9.1 |  9.989 |    -0.000 |
    |  255 |   255 |      73.8 |        9.0 |  0.019 |     0.000 |
    |  256 |   256 |      26.2 |        9.1 |  9.982 |    -0.000 |
    |  257 |   257 |      70.8 |        9.1 |  0.029 |    -0.000 |
    |  258 |   258 |      29.2 |        9.1 |  9.973 |    -0.000 |
    |  259 |   259 |      68.0 |        9.2 |  0.038 |    -0.000 |
    |  260 |   260 |      32.1 |        9.2 |  9.961 |     0.000 |
    |  261 |   261 |      64.9 |        9.1 |  0.072 |     0.000 |
    |  262 |   262 |      35.1 |        9.1 |  9.930 |    -0.000 |
    |  263 |   263 |      62.0 |        9.1 |  0.110 |     0.000 |
    |  264 |   264 |      38.1 |        9.1 |  9.888 |     0.000 |
    |  265 |   265 |      54.5 |        0.9 |  2.591 |     0.000 |
    |  266 |   266 |      45.1 |        0.9 |  7.555 |     0.000 |
    |  267 |   267 |      63.3 |        0.9 |  0.656 |     0.000 |
    |  268 |   268 |      36.1 |        0.9 |  9.413 |     0.000 |
    |  269 |   269 |      66.5 |        0.9 |  0.390 |    -0.000 |
    |  270 |   270 |      33.1 |        0.9 |  9.625 |     0.000 |
    |  271 |   271 |      69.5 |        0.9 |  0.249 |     0.000 |
    |  272 |   272 |      30.1 |        0.9 |  9.774 |    -0.000 |
    |  273 |   273 |      72.5 |        0.9 |  0.152 |    -0.000 |
    |  274 |   274 |      27.1 |        0.9 |  9.854 |     0.000 |
    |  275 |   275 |      24.1 |        0.9 |  9.912 |     0.000 |
    |  276 |   276 |      75.5 |        0.9 |  0.097 |    -0.000 |
    |  277 |   277 |      21.2 |        0.9 |  9.942 |     0.000 |
    |  278 |   278 |      78.3 |        0.9 |  0.060 |    -0.000 |
    |  279 |   279 |      81.4 |        1.2 |  0.038 |    -0.000 |
    |  280 |   280 |      18.5 |        0.8 |  9.962 |    -0.000 |
    |  281 |   281 |      15.9 |        0.9 |  9.976 |    -0.000 |
    |  282 |   282 |      84.5 |        0.9 |  0.023 |     0.000 |
    |  283 |   283 |      12.8 |        0.9 |  9.984 |    -0.000 |
    |  284 |   284 |      87.5 |        0.9 |  0.015 |    -0.000 |
    |  285 |   285 |       9.8 |        0.9 |  9.990 |    -0.000 |
    |  286 |   286 |      90.5 |        0.9 |  0.009 |    -0.000 |
    |  287 |   287 |       6.8 |        0.9 |  9.993 |    -0.000 |
    |  288 |   288 |      93.4 |        0.9 |  0.007 |    -0.000 |
    |  289 |   289 |      58.8 |        9.1 |  0.185 |     0.000 |
    |  290 |   290 |      41.2 |        9.1 |  9.811 |    -0.000 |
    |  291 |   291 |      49.8 |        4.7 |  5.384 |    -0.000 |
    |  292 |   292 |      99.1 |        4.3 |  0.003 |    -0.000 |
    |  293 |   293 |       0.9 |        4.9 |  9.997 |    -0.000 |
    |  294 |   294 |      51.0 |        6.8 |  1.808 |     0.000 |
    |  295 |   295 |      49.0 |        7.3 |  8.364 |    -0.000 |
    |  296 |   296 |      51.5 |        1.0 |  4.070 |    -0.000 |
    |  297 |   297 |      48.8 |        0.8 |  5.756 |     0.000 |
    |  298 |   298 |      96.3 |        0.9 |  0.005 |    -0.000 |
    |  299 |   299 |       3.9 |        0.8 |  9.995 |     0.000 |
    |  300 |   300 |      43.1 |        0.9 |  8.191 |     0.000 |
    |  301 |   301 |      56.2 |        1.2 |  1.950 |    -0.000 |
    |  302 |   302 |      46.2 |        9.2 |  9.655 |    -0.000 |
    |  303 |   303 |      53.9 |        9.0 |  0.422 |     0.000 |
    |  304 |   304 |      39.2 |        0.9 |  9.030 |     0.000 |
    |  305 |   305 |      60.2 |        0.8 |  1.051 |     0.000 |
    |  306 |   306 |       0.8 |        6.5 |  9.998 |     0.000 |
    |  307 |   307 |      99.1 |        6.2 |  0.002 |     0.000 |
    |  308 |   308 |       3.8 |        9.1 |  9.999 |     0.000 |
    |  309 |   309 |      96.4 |        9.2 |  0.001 |     0.000 |
    |  310 |   310 |      49.5 |        4.7 |  5.957 |    -0.000 |
    |  311 |   311 |      49.6 |        4.3 |  5.513 |     0.000 |
    |  312 |   312 |      49.2 |        4.4 |  5.958 |     0.000 |
    |  313 |   313 |      49.5 |        3.9 |  5.532 |     0.000 |
    |  314 |   314 |      49.0 |        4.0 |  6.041 |     0.000 |
    |  315 |   315 |      49.8 |        3.9 |  5.188 |    -0.000 |
    |  316 |   316 |      49.4 |        3.5 |  5.554 |     0.000 |
    |  317 |   317 |      50.0 |        3.6 |  4.996 |     0.000 |
    |  318 |   318 |      49.9 |        3.1 |  5.137 |    -0.000 |
    |  319 |   319 |      48.8 |        4.6 |  6.514 |     0.000 |
    |  320 |   320 |      48.5 |        4.1 |  6.410 |     0.000 |
    |  321 |   321 |      48.0 |        4.4 |  6.919 |     0.000 |
    |  322 |   322 |      47.5 |        4.8 |  7.259 |    -0.000 |
    |  323 |   323 |      47.1 |        4.2 |  7.307 |    -0.000 |
    |  324 |   324 |      46.4 |        4.7 |  7.721 |    -0.000 |
    |  325 |   325 |      46.1 |        3.9 |  7.640 |     0.000 |
    |  326 |   326 |      45.8 |        4.7 |  7.999 |     0.000 |
    |  327 |   327 |      45.5 |        4.2 |  7.887 |     0.000 |
    |  328 |   328 |      45.0 |        4.6 |  8.194 |     0.000 |
    |  329 |   329 |      44.7 |        4.1 |  8.118 |     0.000 |
    |  330 |   330 |      44.3 |        4.8 |  8.391 |    -0.000 |
    |  331 |   331 |      44.2 |        4.1 |  8.317 |    -0.000 |
    |  332 |   332 |      43.5 |        5.0 |  8.651 |    -0.000 |
    |  333 |   333 |      43.3 |        4.2 |  8.502 |    -0.000 |
    |  334 |   334 |      42.6 |        5.1 |  8.819 |    -0.000 |
    |  335 |   335 |      42.4 |        4.3 |  8.742 |    -0.000 |
    |  336 |   336 |      41.7 |        5.2 |  9.021 |     0.000 |
    |  337 |   337 |      41.5 |        4.3 |  8.900 |    -0.000 |
    |  338 |   338 |      40.8 |        5.4 |  9.154 |     0.000 |
    |  339 |   339 |      40.5 |        4.5 |  9.103 |     0.000 |
    |  340 |   340 |      39.0 |        4.6 |  9.283 |     0.000 |
    |  341 |   341 |      38.5 |        5.8 |  9.467 |    -0.000 |
    |  342 |   342 |      37.6 |        4.7 |  9.449 |    -0.000 |
    |  343 |   343 |      37.6 |        5.6 |  9.512 |    -0.000 |
    |  344 |   344 |      36.7 |        4.8 |  9.513 |    -0.000 |
    |  345 |   345 |      36.6 |        5.5 |  9.588 |     0.000 |
    |  346 |   346 |      36.1 |        4.8 |  9.573 |    -0.000 |
    |  347 |   347 |      35.8 |        5.3 |  9.613 |    -0.000 |
    |  348 |   348 |      35.1 |        4.9 |  9.637 |     0.000 |
    |  349 |   349 |      34.5 |        5.4 |  9.686 |     0.000 |
    |  350 |   350 |      34.2 |        4.9 |  9.689 |    -0.000 |
    |  351 |   351 |      33.8 |        5.7 |  9.746 |    -0.000 |
    |  352 |   352 |      33.6 |        5.0 |  9.710 |    -0.000 |
    |  353 |   353 |      32.9 |        5.9 |  9.780 |     0.000 |
    |  354 |   354 |      32.7 |        5.1 |  9.757 |    -0.000 |
    |  355 |   355 |      31.9 |        6.0 |  9.820 |     0.000 |
    |  356 |   356 |      31.8 |        5.1 |  9.787 |     0.000 |
    |  357 |   357 |      44.7 |        5.5 |  8.534 |     0.000 |
    |  358 |   358 |      50.6 |        3.3 |  4.527 |    -0.000 |
    |  359 |   359 |      51.0 |        3.0 |  4.209 |     0.000 |
    |  360 |   360 |      51.4 |        3.4 |  3.788 |     0.000 |
    |  361 |   361 |      51.7 |        3.1 |  3.729 |    -0.000 |
    |  362 |   362 |      52.0 |        3.8 |  3.337 |     0.000 |
    |  363 |   363 |      52.2 |        3.0 |  3.432 |     0.000 |
    |  364 |   364 |      52.7 |        3.7 |  2.947 |    -0.000 |
    |  365 |   365 |      42.8 |        5.8 |  8.970 |     0.000 |
    |  366 |   366 |      49.4 |        3.0 |  5.468 |    -0.000 |
    |  367 |   367 |      30.9 |        6.1 |  9.844 |    -0.000 |
    |  368 |   368 |      30.8 |        5.2 |  9.826 |    -0.000 |
    |  369 |   369 |      29.9 |        6.2 |  9.876 |     0.000 |
    |  370 |   370 |      29.8 |        5.3 |  9.849 |     0.000 |
    |  371 |   371 |      28.8 |        6.5 |  9.901 |    -0.000 |
    |  372 |   372 |      28.7 |        5.4 |  9.879 |     0.000 |
    |  373 |   373 |      28.8 |        4.5 |  9.853 |     0.000 |
    |  374 |   374 |      27.8 |        4.5 |  9.880 |    -0.000 |
    |  375 |   375 |      27.7 |        5.5 |  9.895 |    -0.000 |
    |  376 |   376 |      26.8 |        4.5 |  9.894 |    -0.000 |
    |  377 |   377 |      26.6 |        5.5 |  9.914 |     0.000 |
    |  378 |   378 |      25.8 |        4.4 |  9.912 |     0.000 |
    |  379 |   379 |      25.4 |        5.4 |  9.926 |    -0.000 |
    |  380 |   380 |      24.7 |        4.9 |  9.931 |    -0.000 |
    |  381 |   381 |      24.4 |        5.9 |  9.944 |     0.000 |
    |  382 |   382 |      23.9 |        5.2 |  9.939 |     0.000 |
    |  383 |   383 |      23.4 |        5.8 |  9.952 |    -0.000 |
    |  384 |   384 |      22.8 |        5.1 |  9.949 |     0.000 |
    |  385 |   385 |      22.7 |        5.9 |  9.955 |     0.000 |
    |  386 |   386 |      21.9 |        5.1 |  9.955 |    -0.000 |
    |  387 |   387 |      21.8 |        5.9 |  9.963 |     0.000 |
    |  388 |   388 |      20.9 |        5.1 |  9.963 |    -0.000 |
    |  389 |   389 |      20.9 |        6.0 |  9.967 |     0.000 |
    |  390 |   390 |      19.9 |        5.2 |  9.967 |     0.000 |
    |  391 |   391 |      20.1 |        4.3 |  9.963 |     0.000 |
    |  392 |   392 |      19.1 |        4.2 |  9.967 |     0.000 |
    |  393 |   393 |      18.9 |        5.2 |  9.973 |     0.000 |
    |  394 |   394 |      18.2 |        4.0 |  9.972 |     0.000 |
    |  395 |   395 |      17.6 |        5.0 |  9.977 |     0.000 |
    |  396 |   396 |      16.8 |        4.2 |  9.978 |    -0.000 |
    |  397 |   397 |      16.4 |        5.4 |  9.983 |     0.000 |
    |  398 |   398 |      16.0 |        4.6 |  9.980 |     0.000 |
    |  399 |   399 |      15.6 |        5.5 |  9.984 |     0.000 |
    |  400 |   400 |      15.3 |        5.0 |  9.984 |     0.000 |
    |  401 |   401 |      14.5 |        5.3 |  9.986 |     0.000 |
    |  402 |   402 |      14.0 |        4.6 |  9.986 |    -0.000 |
    |  403 |   403 |      13.6 |        5.6 |  9.989 |    -0.000 |
    |  404 |   404 |      13.2 |        4.8 |  9.988 |     0.000 |
    |  405 |   405 |      12.7 |        5.7 |  9.990 |    -0.000 |
    |  406 |   406 |      12.3 |        4.9 |  9.990 |     0.000 |
    |  407 |   407 |      11.7 |        5.9 |  9.992 |     0.000 |
    |  408 |   408 |      11.4 |        5.0 |  9.991 |    -0.000 |
    |  409 |   409 |      10.7 |        6.0 |  9.993 |    -0.000 |
    |  410 |   410 |      10.4 |        5.2 |  9.993 |    -0.000 |
    |  411 |   411 |       9.7 |        6.1 |  9.995 |     0.000 |
    |  412 |   412 |       9.4 |        5.4 |  9.994 |    -0.000 |
    |  413 |   413 |       8.9 |        6.4 |  9.995 |     0.000 |
    |  414 |   414 |       8.5 |        5.7 |  9.995 |     0.000 |
    |  415 |   415 |       7.5 |        6.3 |  9.996 |     0.000 |
    |  416 |   416 |       7.3 |        5.5 |  9.996 |    -0.000 |
    |  417 |   417 |       7.9 |        4.8 |  9.994 |    -0.000 |
    |  418 |   418 |       6.8 |        4.6 |  9.995 |     0.000 |
    |  419 |   419 |       6.6 |        5.5 |  9.996 |    -0.000 |
    |  420 |   420 |       5.8 |        4.6 |  9.995 |     0.000 |
    |  421 |   421 |       5.6 |        5.5 |  9.996 |    -0.000 |
    |  422 |   422 |       4.8 |        4.6 |  9.996 |    -0.000 |
    |  423 |   423 |      23.0 |        4.2 |  9.939 |     0.000 |
    |  424 |   424 |      47.7 |        3.8 |  6.824 |     0.000 |
    |  425 |   425 |      52.3 |        4.6 |  2.829 |     0.000 |
    |  426 |   426 |      53.0 |        4.2 |  2.701 |     0.000 |
    |  427 |   427 |      53.8 |        3.7 |  2.441 |     0.000 |
    |  428 |   428 |      54.0 |        4.7 |  2.114 |    -0.000 |
    |  429 |   429 |      53.2 |        5.0 |  2.241 |     0.000 |
    |  430 |   430 |      54.3 |        5.6 |  1.699 |    -0.000 |
    |  431 |   431 |      54.9 |        4.4 |  1.872 |     0.000 |
    |  432 |   432 |      55.1 |        5.3 |  1.628 |     0.000 |
    |  433 |   433 |      55.6 |        4.6 |  1.676 |    -0.000 |
    |  434 |   434 |      55.8 |        5.1 |  1.464 |     0.000 |
    |  435 |   435 |      56.7 |        4.7 |  1.389 |     0.000 |
    |  436 |   436 |      56.9 |        5.7 |  1.124 |     0.000 |
    |  437 |   437 |      57.4 |        4.8 |  1.188 |    -0.000 |
    |  438 |   438 |      57.7 |        5.4 |  1.068 |     0.000 |
    |  439 |   439 |      58.4 |        4.9 |  1.011 |     0.000 |
    |  440 |   440 |      59.0 |        5.4 |  0.863 |    -0.000 |
    |  441 |   441 |      59.4 |        4.9 |  0.853 |     0.000 |
    |  442 |   442 |      59.7 |        5.8 |  0.699 |    -0.000 |
    |  443 |   443 |      60.1 |        5.1 |  0.774 |     0.000 |
    |  444 |   444 |      60.6 |        6.1 |  0.592 |     0.000 |
    |  445 |   445 |      61.0 |        5.3 |  0.634 |     0.000 |
    |  446 |   446 |      61.3 |        6.4 |  0.474 |     0.000 |
    |  447 |   447 |      61.8 |        5.7 |  0.527 |     0.000 |
    |  448 |   448 |      61.2 |        4.4 |  0.704 |     0.000 |
    |  449 |   449 |      62.5 |        4.7 |  0.548 |     0.000 |
    |  450 |   450 |      63.2 |        5.7 |  0.423 |     0.000 |
    |  451 |   451 |      63.6 |        4.3 |  0.496 |    -0.000 |
    |  452 |   452 |      64.2 |        5.2 |  0.386 |    -0.000 |
    |  453 |   453 |      64.9 |        4.4 |  0.400 |     0.000 |
    |  454 |   454 |      65.1 |        5.4 |  0.329 |    -0.000 |
    |  455 |   455 |      65.6 |        4.9 |  0.329 |     0.000 |
    |  456 |   456 |      66.3 |        5.3 |  0.279 |     0.000 |
    |  457 |   457 |      66.8 |        4.6 |  0.279 |     0.000 |
    |  458 |   458 |      67.1 |        5.7 |  0.224 |    -0.000 |
    |  459 |   459 |      67.6 |        4.9 |  0.243 |    -0.000 |
    |  460 |   460 |      68.1 |        5.9 |  0.189 |    -0.000 |
    |  461 |   461 |      68.5 |        5.1 |  0.201 |    -0.000 |
    |  462 |   462 |      69.0 |        6.0 |  0.154 |    -0.000 |
    |  463 |   463 |      69.5 |        5.2 |  0.173 |     0.000 |
    |  464 |   464 |      53.4 |        6.0 |  1.843 |    -0.000 |
    |  465 |   465 |      70.0 |        6.3 |  0.128 |     0.000 |
    |  466 |   466 |      70.5 |        5.4 |  0.139 |     0.000 |
    |  467 |   467 |      70.8 |        6.7 |  0.099 |    -0.000 |
    |  468 |   468 |      71.7 |        5.8 |  0.109 |     0.000 |
    |  469 |   469 |      70.7 |        4.5 |  0.159 |    -0.000 |
    |  470 |   470 |      71.8 |        4.6 |  0.129 |    -0.000 |
    |  471 |   471 |      73.0 |        5.7 |  0.090 |     0.000 |
    |  472 |   472 |      73.1 |        4.6 |  0.106 |     0.000 |
    |  473 |   473 |      73.7 |        5.7 |  0.082 |    -0.000 |
    |  474 |   474 |      74.0 |        5.2 |  0.082 |     0.000 |
    |  475 |   475 |      74.7 |        5.8 |  0.068 |     0.000 |
    |  476 |   476 |      75.3 |        5.4 |  0.065 |     0.000 |
    |  477 |   477 |      75.6 |        5.9 |  0.058 |    -0.000 |
    |  478 |   478 |      76.1 |        5.2 |  0.061 |     0.000 |
    |  479 |   479 |      76.2 |        6.0 |  0.051 |    -0.000 |
    |  480 |   480 |      77.1 |        5.2 |  0.051 |     0.000 |
    |  481 |   481 |      77.1 |        6.1 |  0.044 |    -0.000 |
    |  482 |   482 |      78.1 |        5.3 |  0.044 |    -0.000 |
    |  483 |   483 |      76.9 |        4.3 |  0.062 |     0.000 |
    |  484 |   484 |      31.8 |        4.3 |  9.766 |     0.000 |
    |  485 |   485 |      67.8 |        4.2 |  0.254 |     0.000 |
    |  486 |   486 |      73.9 |        4.1 |  0.098 |     0.000 |
    |  487 |   487 |      11.2 |        4.2 |  9.991 |    -0.000 |
    |  488 |   488 |      78.3 |        6.4 |  0.033 |    -0.000 |
    |  489 |   489 |      79.2 |        5.4 |  0.036 |     0.000 |
    |  490 |   490 |      79.5 |        6.3 |  0.029 |     0.000 |
    |  491 |   491 |      80.2 |        5.4 |  0.031 |     0.000 |
    |  492 |   492 |      79.8 |        4.5 |  0.037 |     0.000 |
    |  493 |   493 |      80.8 |        4.6 |  0.032 |     0.000 |
    |  494 |   494 |      81.2 |        5.5 |  0.025 |     0.000 |
    |  495 |   495 |      81.7 |        4.5 |  0.027 |     0.000 |
    |  496 |   496 |      82.2 |        5.5 |  0.022 |    -0.000 |
    |  497 |   497 |      82.6 |        4.3 |  0.025 |    -0.000 |
    |  498 |   498 |      83.4 |        5.3 |  0.018 |     0.000 |
    |  499 |   499 |      84.0 |        4.5 |  0.020 |    -0.000 |
    |  500 |   500 |      84.7 |        5.7 |  0.014 |     0.000 |
    |  501 |   501 |      84.8 |        4.9 |  0.016 |    -0.000 |
    |  502 |   502 |      85.7 |        5.9 |  0.012 |     0.000 |
    |  503 |   503 |      85.8 |        5.1 |  0.014 |    -0.000 |
    |  504 |   504 |      86.9 |        5.3 |  0.011 |     0.000 |
    |  505 |   505 |      85.9 |        4.3 |  0.015 |     0.000 |
    |  506 |   506 |      86.9 |        4.4 |  0.013 |     0.000 |
    |  507 |   507 |      88.0 |        5.4 |  0.009 |    -0.000 |
    |  508 |   508 |      87.9 |        4.5 |  0.010 |     0.000 |
    |  509 |   509 |      89.2 |        5.5 |  0.007 |     0.000 |
    |  510 |   510 |      89.3 |        4.6 |  0.009 |     0.000 |
    |  511 |   511 |      90.3 |        5.6 |  0.006 |    -0.000 |
    |  512 |   512 |      90.5 |        4.8 |  0.007 |    -0.000 |
    |  513 |   513 |      91.5 |        4.9 |  0.006 |    -0.000 |
    |  514 |   514 |      91.4 |        5.8 |  0.005 |     0.000 |
    |  515 |   515 |      92.5 |        5.0 |  0.005 |    -0.000 |
    |  516 |   516 |      92.4 |        5.8 |  0.005 |     0.000 |
    |  517 |   517 |      93.5 |        5.1 |  0.005 |     0.000 |
    |  518 |   518 |      93.4 |        5.9 |  0.004 |    -0.000 |
    |  519 |   519 |      94.5 |        5.2 |  0.004 |    -0.000 |
    |  520 |   520 |      94.4 |        6.0 |  0.003 |     0.000 |
    |  521 |   521 |      95.4 |        5.3 |  0.004 |    -0.000 |
    |  522 |   522 |      94.4 |        4.3 |  0.005 |     0.000 |
    |  523 |   523 |      91.5 |        4.1 |  0.006 |     0.000 |
    |  524 |   524 |       4.6 |        5.5 |  9.997 |     0.000 |
    |  525 |   525 |      26.0 |        3.6 |  9.896 |     0.000 |
    |  526 |   526 |      95.4 |        6.1 |  0.003 |     0.000 |
    |  527 |   527 |      95.4 |        4.4 |  0.004 |    -0.000 |
    |  528 |   528 |       5.1 |        3.7 |  9.995 |    -0.000 |
    |  529 |   529 |       4.2 |        3.6 |  9.996 |     0.000 |
    |  530 |   530 |      55.4 |        3.7 |  1.917 |    -0.000 |
    |  531 |   531 |      88.7 |        3.5 |  0.010 |    -0.000 |
    |  532 |   532 |      83.4 |        3.9 |  0.023 |    -0.000 |
    |  533 |   533 |      34.5 |        4.4 |  9.640 |    -0.000 |
    |  534 |   534 |      99.1 |        2.5 |  0.004 |    -0.000 |
    |  535 |   535 |       1.0 |        2.7 |  9.996 |    -0.000 |
    |  536 |   536 |      57.1 |        6.5 |  0.915 |    -0.000 |
    |  537 |   537 |      57.2 |        3.9 |  1.392 |     0.000 |
    |  538 |   538 |      52.6 |        5.8 |  2.075 |     0.000 |
    |  539 |   539 |      13.7 |        4.0 |  9.985 |     0.000 |
    |  540 |   540 |      65.2 |        3.9 |  0.390 |     0.000 |
    |  541 |   541 |      40.0 |        5.8 |  9.322 |     0.000 |
    |  542 |   542 |      55.3 |        6.1 |  1.316 |     0.000 |
    |  543 |   543 |      63.6 |        6.2 |  0.347 |    -0.000 |
    |  544 |   544 |      90.2 |        6.5 |  0.005 |     0.000 |
    |  545 |   545 |      50.2 |        2.8 |  4.891 |     0.000 |
    |  546 |   546 |      48.0 |        5.3 |  7.440 |    -0.000 |
    |  547 |   547 |      53.0 |        6.7 |  1.536 |     0.000 |
    |  548 |   548 |      24.8 |        6.5 |  9.946 |     0.000 |
    |  549 |   549 |      38.2 |        3.7 |  9.307 |     0.000 |
    |  550 |   550 |      18.8 |        6.1 |  9.977 |     0.000 |
    |  551 |   551 |      80.3 |        3.6 |  0.038 |     0.000 |
    |  552 |   552 |      79.4 |        3.6 |  0.045 |     0.000 |
    |  553 |   553 |      16.3 |        3.6 |  9.977 |    -0.000 |
    |  554 |   554 |      16.5 |        6.3 |  9.985 |     0.000 |
    |  555 |   555 |      81.5 |        6.4 |  0.020 |     0.000 |
    |  556 |   556 |      62.4 |        3.4 |  0.657 |     0.000 |
    |  557 |   557 |      22.0 |        6.7 |  9.967 |     0.000 |
    |  558 |   558 |      37.4 |        6.5 |  9.618 |    -0.000 |
    |  559 |   559 |      75.3 |        6.4 |  0.054 |    -0.000 |
    |  560 |   560 |      36.5 |        4.1 |  9.493 |     0.000 |
    |  561 |   561 |      93.3 |        6.8 |  0.003 |     0.000 |
    |  562 |   562 |      40.4 |        3.5 |  8.990 |    -0.000 |
    |  563 |   563 |      66.6 |        6.5 |  0.207 |     0.000 |
    |  564 |   564 |      87.9 |        6.5 |  0.007 |    -0.000 |
    |  565 |   565 |      95.3 |        3.5 |  0.005 |     0.000 |
    |  566 |   566 |      94.3 |        3.5 |  0.005 |    -0.000 |
    |  567 |   567 |      96.2 |        3.6 |  0.004 |     0.000 |
    |  568 |   568 |      71.7 |        3.6 |  0.151 |     0.000 |
    |  569 |   569 |      70.6 |        3.6 |  0.175 |     0.000 |
    |  570 |   570 |      45.8 |        3.4 |  7.597 |    -0.000 |
    |  571 |   571 |      13.1 |        6.6 |  9.992 |     0.000 |
    |  572 |   572 |      19.4 |        3.2 |  9.963 |    -0.000 |
    |  573 |   573 |      20.4 |        3.4 |  9.956 |     0.000 |
    |  574 |   574 |      42.4 |        3.4 |  8.607 |    -0.000 |
    |  575 |   575 |      28.0 |        3.6 |  9.858 |    -0.000 |
    |  576 |   576 |      29.0 |        3.6 |  9.839 |    -0.000 |
    |  577 |   577 |      60.3 |        7.0 |  0.479 |     0.000 |
    |  578 |   578 |      44.2 |        3.4 |  8.143 |     0.000 |
    |  579 |   579 |       4.6 |        6.4 |  9.997 |     0.000 |
    |  580 |   580 |       3.6 |        6.4 |  9.997 |     0.000 |
    |  581 |   581 |      73.1 |        6.5 |  0.074 |    -0.000 |
    |  582 |   582 |      26.5 |        6.4 |  9.927 |     0.000 |
    |  583 |   583 |       7.1 |        3.7 |  9.994 |    -0.000 |
    |  584 |   584 |       8.2 |        3.7 |  9.994 |     0.000 |
    |  585 |   585 |      34.1 |        6.5 |  9.772 |     0.000 |
    |  586 |   586 |      84.6 |        6.6 |  0.012 |    -0.000 |
    |  587 |   587 |      77.8 |        7.2 |  0.029 |     0.000 |
    |  588 |   588 |      86.7 |        3.5 |  0.014 |    -0.000 |
    |  589 |   589 |      85.7 |        3.4 |  0.017 |     0.000 |
    |  590 |   590 |      69.4 |        7.1 |  0.110 |    -0.000 |
    |  591 |   591 |      10.0 |        6.9 |  9.995 |    -0.000 |
    |  592 |   592 |      80.0 |        2.7 |  0.044 |    -0.000 |
    |  593 |   593 |       3.7 |        5.5 |  9.997 |     0.000 |
    |  594 |   594 |      31.2 |        6.9 |  9.873 |    -0.000 |
    |  595 |   595 |      96.3 |        4.5 |  0.004 |     0.000 |
    |  596 |   596 |      80.8 |        2.6 |  0.038 |     0.000 |
    |  597 |   597 |      95.0 |        2.7 |  0.005 |     0.000 |
    |  598 |   598 |      50.9 |        4.0 |  4.076 |     0.000 |
    |  599 |   599 |      20.0 |        2.6 |  9.956 |    -0.000 |
    |  600 |   600 |       4.5 |        2.7 |  9.995 |    -0.000 |
    |  601 |   601 |      71.3 |        2.7 |  0.167 |    -0.000 |
    |  602 |   602 |      48.1 |        9.2 |  9.623 |    -0.000 |
    |  603 |   603 |      52.0 |        9.2 |  0.399 |    -0.000 |
    |  604 |   604 |      52.0 |        2.4 |  3.674 |    -0.000 |
    |  605 |   605 |      52.6 |        2.1 |  3.382 |     0.000 |
    |  606 |   606 |      95.3 |        7.0 |  0.002 |    -0.000 |
    |  607 |   607 |      96.3 |        7.1 |  0.002 |    -0.000 |
    |  608 |   608 |      99.1 |        8.1 |  0.001 |     0.000 |
    |  609 |   609 |       0.9 |        8.1 |  9.999 |    -0.000 |
    |  610 |   610 |      28.1 |        2.7 |  9.849 |    -0.000 |
    |  611 |   611 |      29.1 |        2.7 |  9.817 |    -0.000 |
    |  612 |   612 |       7.4 |        2.7 |  9.994 |     0.000 |
    |  613 |   613 |       8.3 |        6.9 |  9.996 |     0.000 |
    |  614 |   614 |       7.3 |        7.0 |  9.997 |    -0.000 |
    |  615 |   615 |      19.0 |        2.3 |  9.961 |     0.000 |
    |  616 |   616 |       8.4 |        2.7 |  9.992 |     0.000 |
    |  617 |   617 |      97.2 |        3.7 |  0.004 |    -0.000 |
    |  618 |   618 |      70.3 |        2.7 |  0.202 |     0.000 |
    |  619 |   619 |       4.6 |        7.3 |  9.998 |     0.000 |
    |  620 |   620 |       3.5 |        7.3 |  9.998 |    -0.000 |
    |  621 |   621 |      52.4 |        6.5 |  1.876 |     0.000 |
    |  622 |   622 |      52.7 |        7.1 |  1.454 |     0.000 |
    |  623 |   623 |      53.4 |        7.2 |  1.274 |     0.000 |
    |  624 |   624 |      52.1 |        5.8 |  2.373 |     0.000 |
    |  625 |   625 |      29.4 |        7.4 |  9.917 |     0.000 |
    |  626 |   626 |       5.6 |        7.3 |  9.998 |     0.000 |
    |  627 |   627 |      28.2 |        7.3 |  9.929 |     0.000 |
    |  628 |   628 |      61.3 |        3.5 |  0.764 |     0.000 |
    |  629 |   629 |      62.0 |        2.7 |  0.734 |    -0.000 |
    |  630 |   630 |      63.1 |        2.7 |  0.610 |    -0.000 |
    |  631 |   631 |      96.3 |        6.2 |  0.003 |     0.000 |
    |  632 |   632 |      97.2 |        7.2 |  0.002 |    -0.000 |
    |  633 |   633 |      72.4 |        2.7 |  0.146 |     0.000 |
    |  634 |   634 |      27.1 |        2.7 |  9.866 |    -0.000 |
    |  635 |   635 |       6.4 |        2.8 |  9.994 |     0.000 |
    |  636 |   636 |       3.3 |        3.5 |  9.996 |     0.000 |
    |  637 |   637 |      97.0 |        2.8 |  0.004 |    -0.000 |
    |  638 |   638 |      86.4 |        2.6 |  0.016 |     0.000 |
    |  639 |   639 |      87.4 |        2.6 |  0.014 |     0.000 |
    |  640 |   640 |      79.0 |        2.7 |  0.050 |    -0.000 |
    |  641 |   641 |      85.4 |        2.6 |  0.019 |    -0.000 |
    |  642 |   642 |      61.2 |        2.6 |  0.843 |    -0.000 |
    |  643 |   643 |      29.1 |        1.8 |  9.812 |     0.000 |
    |  644 |   644 |      70.9 |        1.8 |  0.193 |    -0.000 |
    |  645 |   645 |      94.0 |        2.6 |  0.006 |    -0.000 |
    |  646 |   646 |      20.6 |        2.6 |  9.953 |    -0.000 |
    |  647 |   647 |       7.6 |        1.8 |  9.993 |    -0.000 |
    |  648 |   648 |       1.9 |        0.8 |  9.996 |     0.000 |
    |  649 |   649 |      98.3 |        1.0 |  0.004 |    -0.000 |
    |  650 |   650 |      53.1 |        1.6 |  3.170 |    -0.000 |
    |  651 |   651 |      53.7 |        2.2 |  2.799 |    -0.000 |
    |  652 |   652 |       2.4 |        3.3 |  9.996 |     0.000 |
    |  653 |   653 |      84.8 |        3.4 |  0.019 |     0.000 |
    |  654 |   654 |      84.4 |        2.7 |  0.022 |     0.000 |
    |  655 |   655 |      21.3 |        3.4 |  9.951 |     0.000 |
    |  656 |   656 |      21.4 |        2.6 |  9.945 |    -0.000 |
    |  657 |   657 |      78.5 |        3.5 |  0.051 |    -0.000 |
    |  658 |   658 |      78.1 |        2.6 |  0.059 |     0.000 |
    |  659 |   659 |       9.2 |        3.6 |  9.992 |    -0.000 |
    |  660 |   660 |       9.4 |        2.7 |  9.992 |     0.000 |
    |  661 |   661 |      29.9 |        3.5 |  9.804 |    -0.000 |
    |  662 |   662 |      30.1 |        2.7 |  9.793 |    -0.000 |
    |  663 |   663 |      69.6 |        3.5 |  0.211 |     0.000 |
    |  664 |   664 |      69.3 |        2.6 |  0.231 |     0.000 |
    |  665 |   665 |      93.3 |        3.4 |  0.006 |    -0.000 |
    |  666 |   666 |      93.1 |        2.6 |  0.006 |    -0.000 |
    |  667 |   667 |      52.9 |        7.6 |  1.153 |     0.000 |
    |  668 |   668 |      52.3 |        7.7 |  1.223 |    -0.000 |
    |  669 |   669 |      85.9 |        1.7 |  0.018 |     0.000 |
    |  670 |   670 |      95.1 |        7.7 |  0.002 |     0.000 |
    |  671 |   671 |      79.6 |        1.8 |  0.048 |    -0.000 |
    |  672 |   672 |      94.1 |        7.7 |  0.002 |    -0.000 |
    |  673 |   673 |      94.7 |        1.8 |  0.005 |    -0.000 |
    |  674 |   674 |      20.5 |        1.8 |  9.950 |     0.000 |
    |  675 |   675 |      97.3 |        6.3 |  0.002 |     0.000 |
    |  676 |   676 |      98.2 |        7.2 |  0.002 |     0.000 |
    |  677 |   677 |       2.7 |        2.5 |  9.996 |     0.000 |
    |  678 |   678 |      61.2 |        1.8 |  0.865 |     0.000 |
    |  679 |   679 |      51.7 |        6.2 |  2.101 |    -0.000 |
    |  680 |   680 |      48.8 |        3.5 |  5.974 |     0.000 |
    |  681 |   681 |      97.1 |        8.2 |  0.001 |    -0.000 |
    |  682 |   682 |       7.7 |        7.5 |  9.997 |     0.000 |
    |  683 |   683 |       6.9 |        8.1 |  9.998 |    -0.000 |
    |  684 |   684 |       9.0 |        7.4 |  9.996 |     0.000 |
    |  685 |   685 |      97.3 |        4.5 |  0.003 |     0.000 |
    |  686 |   686 |      95.7 |        1.8 |  0.005 |    -0.000 |
    |  687 |   687 |      53.9 |        7.9 |  0.896 |     0.000 |
    |  688 |   688 |      54.6 |        7.4 |  1.000 |     0.000 |
    |  689 |   689 |      54.8 |        8.4 |  0.602 |     0.000 |
    |  690 |   690 |      55.5 |        7.5 |  0.814 |    -0.000 |
    |  691 |   691 |      28.6 |        8.0 |  9.943 |    -0.000 |
    |  692 |   692 |      29.8 |        7.8 |  9.924 |    -0.000 |
    |  693 |   693 |      27.4 |        8.1 |  9.957 |    -0.000 |
    |  694 |   694 |       2.1 |        4.2 |  9.996 |    -0.000 |
    |  695 |   695 |      51.5 |        5.6 |  2.737 |     0.000 |
    |  696 |   696 |      52.0 |        5.4 |  2.554 |    -0.000 |
    |  697 |   697 |      98.1 |        2.9 |  0.004 |    -0.000 |
    |  698 |   698 |      97.6 |        1.8 |  0.004 |    -0.000 |
    |  699 |   699 |      54.2 |        1.5 |  2.698 |     0.000 |
    |  700 |   700 |      54.3 |        2.8 |  2.481 |     0.000 |
    |  701 |   701 |      55.0 |        2.0 |  2.293 |     0.000 |
    |  702 |   702 |      55.5 |        2.8 |  2.070 |    -0.000 |
    |  703 |   703 |      55.8 |        1.9 |  2.062 |     0.000 |
    |  704 |   704 |      56.1 |        2.5 |  1.867 |    -0.000 |
    |  705 |   705 |      56.9 |        2.1 |  1.722 |     0.000 |
    |  706 |   706 |      57.3 |        2.9 |  1.511 |     0.000 |
    |  707 |   707 |      57.7 |        1.8 |  1.511 |    -0.000 |
    |  708 |   708 |      58.1 |        2.6 |  1.391 |     0.000 |
    |  709 |   709 |      58.9 |        2.4 |  1.222 |     0.000 |
    |  710 |   710 |      59.2 |        3.0 |  1.118 |     0.000 |
    |  711 |   711 |      59.4 |        2.1 |  1.150 |    -0.000 |
    |  712 |   712 |      58.3 |        3.5 |  1.207 |    -0.000 |
    |  713 |   713 |      52.5 |        5.2 |  2.515 |    -0.000 |
    |  714 |   714 |      51.0 |        6.2 |  2.458 |     0.000 |
    |  715 |   715 |      55.1 |        1.0 |  2.375 |    -0.000 |
    |  716 |   716 |      51.7 |        1.8 |  3.934 |     0.000 |
    |  717 |   717 |      52.6 |        0.9 |  3.459 |    -0.000 |
    |  718 |   718 |       7.7 |        8.6 |  9.998 |    -0.000 |
    |  719 |   719 |      47.2 |        9.2 |  9.612 |     0.000 |
    |  720 |   720 |      46.6 |        8.4 |  9.242 |     0.000 |
    |  721 |   721 |      45.6 |        8.3 |  9.348 |     0.000 |
    |  722 |   722 |      45.9 |        7.5 |  8.952 |     0.000 |
    |  723 |   723 |      47.0 |        7.4 |  8.803 |     0.000 |
    |  724 |   724 |      46.7 |        6.8 |  8.487 |     0.000 |
    |  725 |   725 |      47.7 |        6.7 |  8.259 |    -0.000 |
    |  726 |   726 |      47.1 |        6.4 |  8.266 |     0.000 |
    |  727 |   727 |      48.0 |        6.2 |  7.950 |    -0.000 |
    |  728 |   728 |      47.9 |        7.7 |  8.753 |     0.000 |
    |  729 |   729 |      44.6 |        8.3 |  9.407 |     0.000 |
    |  730 |   730 |      44.8 |        7.4 |  9.110 |     0.000 |
    |  731 |   731 |      45.1 |        6.4 |  8.667 |    -0.000 |
    |  732 |   732 |      43.9 |        6.6 |  8.979 |    -0.000 |
    |  733 |   733 |      43.8 |        7.4 |  9.200 |    -0.000 |
    |  734 |   734 |      45.9 |        5.6 |  8.257 |     0.000 |
    |  735 |   735 |      52.9 |        9.2 |  0.414 |     0.000 |
    |  736 |   736 |      51.0 |        7.6 |  1.404 |    -0.000 |
    |  737 |   737 |      48.7 |        7.9 |  8.887 |    -0.000 |
    |  738 |   738 |      49.1 |        6.6 |  8.032 |     0.000 |
    |  739 |   739 |      49.6 |        0.8 |  5.238 |    -0.000 |
    |  740 |   740 |      49.3 |        1.5 |  5.434 |     0.000 |
    |  741 |   741 |      48.6 |        1.6 |  5.883 |    -0.000 |
    |  742 |   742 |      48.7 |        2.3 |  5.911 |    -0.000 |
    |  743 |   743 |      47.9 |        2.3 |  6.427 |     0.000 |
    |  744 |   744 |      47.8 |        1.6 |  6.341 |    -0.000 |
    |  745 |   745 |      47.1 |        2.5 |  6.836 |     0.000 |
    |  746 |   746 |      46.9 |        1.7 |  6.833 |    -0.000 |
    |  747 |   747 |      49.9 |        1.6 |  5.079 |     0.000 |
    |  748 |   748 |      50.6 |        0.9 |  4.656 |     0.000 |
    |  749 |   749 |      50.9 |        1.6 |  4.447 |    -0.000 |
    |  750 |   750 |      49.6 |        2.3 |  5.332 |     0.000 |
    |  751 |   751 |      55.0 |        9.1 |  0.317 |    -0.000 |
    |  752 |   752 |      44.1 |        0.9 |  7.930 |    -0.000 |
    |  753 |   753 |      45.1 |        1.7 |  7.640 |    -0.000 |
    |  754 |   754 |      47.5 |        8.4 |  9.242 |     0.000 |
    |  755 |   755 |       1.8 |        2.5 |  9.996 |     0.000 |
    |  756 |   756 |      50.4 |        4.1 |  4.593 |    -0.000 |
    |  757 |   757 |      50.8 |        4.5 |  3.911 |    -0.000 |
    |  758 |   758 |      51.4 |        4.4 |  3.433 |     0.000 |
    |  759 |   759 |      44.2 |        1.7 |  7.929 |    -0.000 |
    |  760 |   760 |      43.2 |        1.7 |  8.247 |    -0.000 |
    |  761 |   761 |      43.3 |        2.6 |  8.283 |     0.000 |
    |  762 |   762 |      42.1 |        0.9 |  8.480 |     0.000 |
    |  763 |   763 |      42.2 |        1.7 |  8.468 |    -0.000 |
    |  764 |   764 |      41.3 |        1.7 |  8.713 |    -0.000 |
    |  765 |   765 |      41.3 |        2.6 |  8.734 |     0.000 |
    |  766 |   766 |      44.2 |        2.6 |  8.052 |     0.000 |
    |  767 |   767 |      45.1 |        2.6 |  7.710 |     0.000 |
    |  768 |   768 |      43.2 |        9.1 |  9.753 |     0.000 |
    |  769 |   769 |      42.2 |        9.1 |  9.792 |    -0.000 |
    |  770 |   770 |      41.6 |        8.2 |  9.607 |     0.000 |
    |  771 |   771 |      40.5 |        7.9 |  9.614 |    -0.000 |
    |  772 |   772 |      41.2 |        7.1 |  9.422 |    -0.000 |
    |  773 |   773 |      42.0 |        7.4 |  9.385 |     0.000 |
    |  774 |   774 |      41.6 |        6.6 |  9.281 |     0.000 |
    |  775 |   775 |      39.9 |        7.0 |  9.507 |     0.000 |
    |  776 |   776 |      39.2 |        8.1 |  9.723 |    -0.000 |
    |  777 |   777 |      39.1 |        7.3 |  9.600 |     0.000 |
    |  778 |   778 |      42.6 |        8.3 |  9.549 |    -0.000 |
    |  779 |   779 |      56.8 |        9.2 |  0.239 |    -0.000 |
    |  780 |   780 |      56.5 |        8.3 |  0.478 |     0.000 |
    |  781 |   781 |      57.5 |        8.3 |  0.437 |     0.000 |
    |  782 |   782 |      57.3 |        7.4 |  0.653 |     0.000 |
    |  783 |   783 |      58.2 |        7.4 |  0.591 |     0.000 |
    |  784 |   784 |      56.2 |        6.7 |  0.977 |    -0.000 |
    |  785 |   785 |      58.2 |        6.3 |  0.795 |    -0.000 |
    |  786 |   786 |      59.2 |        7.4 |  0.492 |     0.000 |
    |  787 |   787 |      57.8 |        9.1 |  0.202 |     0.000 |
    |  788 |   788 |      45.2 |        9.2 |  9.678 |    -0.000 |
    |  789 |   789 |      38.2 |        8.2 |  9.766 |    -0.000 |
    |  790 |   790 |      38.3 |        7.4 |  9.668 |     0.000 |
    |  791 |   791 |      38.9 |        6.7 |  9.543 |     0.000 |
    |  792 |   792 |      57.3 |        1.0 |  1.677 |    -0.000 |
    |  793 |   793 |      40.2 |        0.9 |  8.890 |     0.000 |
    |  794 |   794 |      40.3 |        1.8 |  8.877 |    -0.000 |
    |  795 |   795 |      39.3 |        1.8 |  9.061 |    -0.000 |
    |  796 |   796 |      39.3 |        2.6 |  9.078 |     0.000 |
    |  797 |   797 |      38.3 |        2.7 |  9.240 |    -0.000 |
    |  798 |   798 |      38.3 |        1.7 |  9.183 |    -0.000 |
    |  799 |   799 |      37.3 |        2.5 |  9.331 |    -0.000 |
    |  800 |   800 |      37.2 |        1.7 |  9.321 |     0.000 |
    |  801 |   801 |      36.2 |        2.5 |  9.444 |    -0.000 |
    |  802 |   802 |      51.0 |        8.4 |  0.862 |     0.000 |
    |  803 |   803 |      50.0 |        4.2 |  5.037 |     0.000 |
    |  804 |   804 |      37.3 |        8.2 |  9.807 |     0.000 |
    |  805 |   805 |      37.1 |        9.1 |  9.903 |     0.000 |
    |  806 |   806 |      36.3 |        8.2 |  9.831 |    -0.000 |
    |  807 |   807 |       6.9 |        9.1 |  9.999 |    -0.000 |
    |  808 |   808 |      52.3 |        7.1 |  1.502 |     0.000 |
    |  809 |   809 |      60.3 |        3.5 |  0.909 |     0.000 |
    |  810 |   810 |      59.2 |        0.8 |  1.246 |    -0.000 |
    |  811 |   811 |      39.1 |        9.1 |  9.858 |    -0.000 |
    |  812 |   812 |      36.1 |        9.1 |  9.920 |     0.000 |
    |  813 |   813 |      35.2 |        8.3 |  9.861 |     0.000 |
    |  814 |   814 |      34.2 |        8.3 |  9.881 |     0.000 |
    |  815 |   815 |      36.3 |        7.3 |  9.752 |    -0.000 |
    |  816 |   816 |      35.3 |        7.3 |  9.783 |     0.000 |
    |  817 |   817 |      46.9 |        0.8 |  6.735 |    -0.000 |
    |  818 |   818 |      43.5 |        8.3 |  9.502 |     0.000 |
    |  819 |   819 |       6.6 |        7.2 |  9.997 |     0.000 |
    |  820 |   820 |      38.1 |        0.9 |  9.192 |     0.000 |
    |  821 |   821 |       0.8 |        7.2 |  9.998 |    -0.000 |
    |  822 |   822 |       1.7 |        8.2 |  9.999 |     0.000 |
    |  823 |   823 |       1.8 |        9.1 |  9.999 |     0.000 |
    |  824 |   824 |       2.7 |        8.2 |  9.999 |     0.000 |
    |  825 |   825 |      99.1 |        7.1 |  0.002 |    -0.000 |
    |  826 |   826 |       4.6 |        8.2 |  9.999 |     0.000 |
    |  827 |   827 |       1.7 |        7.2 |  9.998 |     0.000 |
    |  828 |   828 |       1.5 |        6.2 |  9.998 |     0.000 |
    |  829 |   829 |       2.6 |        6.3 |  9.998 |     0.000 |
    |  830 |   830 |      94.3 |        6.9 |  0.003 |    -0.000 |
    |  831 |   831 |      93.1 |        7.6 |  0.002 |    -0.000 |
    |  832 |   832 |      92.1 |        7.5 |  0.003 |     0.000 |
    |  833 |   833 |      92.8 |        8.4 |  0.002 |    -0.000 |
    |  834 |   834 |      92.3 |        6.7 |  0.004 |     0.000 |
    |  835 |   835 |      91.1 |        7.4 |  0.003 |     0.000 |
    |  836 |   836 |      91.2 |        6.6 |  0.004 |     0.000 |
    |  837 |   837 |      90.1 |        7.3 |  0.004 |    -0.000 |
    |  838 |   838 |      89.1 |        7.3 |  0.005 |     0.000 |
    |  839 |   839 |      89.8 |        8.2 |  0.003 |     0.000 |
    |  840 |   840 |      88.8 |        8.1 |  0.004 |    -0.000 |
    |  841 |   841 |      90.8 |        8.3 |  0.002 |     0.000 |
    |  842 |   842 |      88.4 |        7.2 |  0.006 |     0.000 |
    |  843 |   843 |      88.0 |        7.8 |  0.005 |     0.000 |
    |  844 |   844 |      87.2 |        8.7 |  0.003 |     0.000 |
    |  845 |   845 |      86.6 |        7.9 |  0.006 |     0.000 |
    |  846 |   846 |      87.2 |        7.3 |  0.007 |     0.000 |
    |  847 |   847 |      86.2 |        7.4 |  0.007 |     0.000 |
    |  848 |   848 |      85.8 |        8.2 |  0.006 |    -0.000 |
    |  849 |   849 |      85.6 |        7.4 |  0.008 |     0.000 |
    |  850 |   850 |      84.9 |        8.3 |  0.006 |     0.000 |
    |  851 |   851 |      86.7 |        6.1 |  0.010 |     0.000 |
    |  852 |   852 |      17.7 |        6.2 |  9.982 |     0.000 |
    |  853 |   853 |      17.8 |        7.4 |  9.987 |    -0.000 |
    |  854 |   854 |      19.0 |        7.1 |  9.982 |    -0.000 |
    |  855 |   855 |      19.4 |        7.9 |  9.986 |     0.000 |
    |  856 |   856 |      18.6 |        8.3 |  9.990 |    -0.000 |
    |  857 |   857 |      17.2 |        8.4 |  9.993 |     0.000 |
    |  858 |   858 |      20.0 |        6.9 |  9.977 |    -0.000 |
    |  859 |   859 |      20.3 |        7.7 |  9.983 |    -0.000 |
    |  860 |   860 |      16.5 |        7.3 |  9.989 |    -0.000 |
    |  861 |   861 |      15.3 |        7.3 |  9.990 |     0.000 |
    |  862 |   862 |      16.3 |        8.2 |  9.993 |     0.000 |
    |  863 |   863 |      15.3 |        8.2 |  9.994 |    -0.000 |
    |  864 |   864 |      14.3 |        7.3 |  9.992 |    -0.000 |
    |  865 |   865 |      16.1 |        9.1 |  9.997 |     0.000 |
    |  866 |   866 |      74.0 |        6.3 |  0.067 |    -0.000 |
    |  867 |   867 |      74.7 |        7.2 |  0.048 |     0.000 |
    |  868 |   868 |      75.9 |        7.5 |  0.035 |    -0.000 |
    |  869 |   869 |      74.8 |        8.3 |  0.029 |     0.000 |
    |  870 |   870 |      75.8 |        8.3 |  0.024 |     0.000 |
    |  871 |   871 |      76.8 |        7.6 |  0.029 |    -0.000 |
    |  872 |   872 |      73.4 |        7.6 |  0.050 |     0.000 |
    |  873 |   873 |      72.6 |        7.4 |  0.062 |     0.000 |
    |  874 |   874 |      72.8 |        8.4 |  0.037 |    -0.000 |
    |  875 |   875 |      72.1 |        7.9 |  0.054 |     0.000 |
    |  876 |   876 |      72.2 |        9.1 |  0.022 |    -0.000 |
    |  877 |   877 |      71.5 |        7.3 |  0.075 |    -0.000 |
    |  878 |   878 |      70.8 |        7.9 |  0.065 |     0.000 |
    |  879 |   879 |      25.5 |        6.4 |  9.940 |     0.000 |
    |  880 |   880 |      25.5 |        7.3 |  9.952 |    -0.000 |
    |  881 |   881 |      24.3 |        7.1 |  9.960 |    -0.000 |
    |  882 |   882 |      24.3 |        8.2 |  9.973 |     0.000 |
    |  883 |   883 |      25.4 |        8.2 |  9.970 |    -0.000 |
    |  884 |   884 |      23.7 |        6.3 |  9.953 |    -0.000 |
    |  885 |   885 |      23.2 |        7.4 |  9.969 |    -0.000 |
    |  886 |   886 |      22.9 |        6.6 |  9.963 |    -0.000 |
    |  887 |   887 |      22.2 |        7.5 |  9.975 |     0.000 |
    |  888 |   888 |      21.2 |        7.6 |  9.979 |     0.000 |
    |  889 |   889 |      21.3 |        8.4 |  9.986 |     0.000 |
    |  890 |   890 |       0.8 |        5.6 |  9.997 |     0.000 |
    |  891 |   891 |       1.8 |        5.1 |  9.997 |    -0.000 |
    |  892 |   892 |       1.2 |        3.8 |  9.997 |    -0.000 |
    |  893 |   893 |       5.6 |        6.4 |  9.997 |    -0.000 |
    |  894 |   894 |       9.3 |        6.9 |  9.996 |    -0.000 |
    |  895 |   895 |       9.8 |        7.7 |  9.996 |    -0.000 |
    |  896 |   896 |      10.7 |        7.9 |  9.996 |    -0.000 |
    |  897 |   897 |      10.5 |        8.7 |  9.998 |    -0.000 |
    |  898 |   898 |       9.6 |        8.5 |  9.998 |     0.000 |
    |  899 |   899 |      89.1 |        6.4 |  0.006 |    -0.000 |
    |  900 |   900 |       5.7 |        8.2 |  9.998 |     0.000 |
    |  901 |   901 |       4.8 |        9.1 |  9.999 |     0.000 |
    |  902 |   902 |      27.5 |        7.3 |  9.935 |     0.000 |
    |  903 |   903 |      26.4 |        8.2 |  9.962 |    -0.000 |
    |  904 |   904 |      27.2 |        9.1 |  9.978 |    -0.000 |
    |  905 |   905 |      26.5 |        7.3 |  9.946 |    -0.000 |
    |  906 |   906 |      82.6 |        6.5 |  0.017 |     0.000 |
    |  907 |   907 |      81.8 |        7.3 |  0.014 |    -0.000 |
    |  908 |   908 |      82.8 |        7.4 |  0.012 |    -0.000 |
    |  909 |   909 |      80.9 |        7.2 |  0.018 |    -0.000 |
    |  910 |   910 |      81.0 |        8.1 |  0.011 |     0.000 |
    |  911 |   911 |      80.4 |        8.1 |  0.014 |    -0.000 |
    |  912 |   912 |      81.1 |        9.1 |  0.006 |    -0.000 |
    |  913 |   913 |      80.0 |        7.1 |  0.020 |    -0.000 |
    |  914 |   914 |      79.3 |        8.1 |  0.016 |    -0.000 |
    |  915 |   915 |      78.8 |        8.8 |  0.011 |    -0.000 |
    |  916 |   916 |      83.6 |        6.5 |  0.014 |     0.000 |
    |  917 |   917 |      83.8 |        7.4 |  0.010 |    -0.000 |
    |  918 |   918 |      83.0 |        8.5 |  0.007 |    -0.000 |
    |  919 |   919 |      78.9 |        7.1 |  0.025 |    -0.000 |
    |  920 |   920 |      12.1 |        6.7 |  9.993 |     0.000 |
    |  921 |   921 |      12.4 |        7.5 |  9.995 |    -0.000 |
    |  922 |   922 |      13.3 |        7.4 |  9.993 |     0.000 |
    |  923 |   923 |      13.3 |        8.3 |  9.996 |    -0.000 |
    |  924 |   924 |      12.4 |        8.3 |  9.996 |     0.000 |
    |  925 |   925 |      14.0 |        6.4 |  9.990 |    -0.000 |
    |  926 |   926 |      11.1 |        6.9 |  9.995 |    -0.000 |
    |  927 |   927 |      11.7 |        7.5 |  9.995 |    -0.000 |
    |  928 |   928 |      21.0 |        6.8 |  9.974 |    -0.000 |
    |  929 |   929 |      77.0 |        6.9 |  0.035 |     0.000 |
    |  930 |   930 |      77.5 |        7.7 |  0.025 |     0.000 |
    |  931 |   931 |      70.2 |        7.4 |  0.089 |     0.000 |
    |  932 |   932 |      69.7 |        7.9 |  0.077 |     0.000 |
    |  933 |   933 |      69.1 |        7.8 |  0.092 |     0.000 |
    |  934 |   934 |      68.9 |        8.5 |  0.062 |    -0.000 |
    |  935 |   935 |      68.5 |        6.9 |  0.138 |    -0.000 |
    |  936 |   936 |      68.2 |        7.6 |  0.109 |     0.000 |
    |  937 |   937 |      67.6 |        6.7 |  0.165 |    -0.000 |
    |  938 |   938 |      67.2 |        7.5 |  0.137 |    -0.000 |
    |  939 |   939 |      66.2 |        7.4 |  0.166 |     0.000 |
    |  940 |   940 |      67.0 |        8.3 |  0.093 |    -0.000 |
    |  941 |   941 |      66.0 |        8.3 |  0.118 |     0.000 |
    |  942 |   942 |      65.1 |        7.2 |  0.211 |    -0.000 |
    |  943 |   943 |      64.9 |        8.2 |  0.141 |    -0.000 |
    |  944 |   944 |      63.9 |        9.1 |  0.079 |    -0.000 |
    |  945 |   945 |      63.7 |        8.1 |  0.184 |    -0.000 |
    |  946 |   946 |      62.9 |        8.5 |  0.159 |     0.000 |
    |  947 |   947 |      62.7 |        7.4 |  0.279 |    -0.000 |
    |  948 |   948 |      32.2 |        6.8 |  9.843 |     0.000 |
    |  949 |   949 |      31.3 |        7.7 |  9.898 |    -0.000 |
    |  950 |   950 |      32.3 |        7.6 |  9.882 |     0.000 |
    |  951 |   951 |      30.4 |        7.8 |  9.917 |     0.000 |
    |  952 |   952 |      31.3 |        8.5 |  9.934 |    -0.000 |
    |  953 |   953 |      30.4 |        8.5 |  9.941 |     0.000 |
    |  954 |   954 |      30.2 |        7.0 |  9.892 |     0.000 |
    |  955 |   955 |      65.5 |        6.1 |  0.266 |     0.000 |
    |  956 |   956 |      99.1 |        5.2 |  0.003 |    -0.000 |
    |  957 |   957 |      96.1 |        7.9 |  0.002 |     0.000 |
    |  958 |   958 |      79.3 |        0.9 |  0.053 |    -0.000 |
    |  959 |   959 |      80.3 |        1.0 |  0.045 |     0.000 |
    |  960 |   960 |      81.1 |        1.9 |  0.038 |    -0.000 |
    |  961 |   961 |      82.5 |        1.0 |  0.031 |    -0.000 |
    |  962 |   962 |      83.1 |        1.9 |  0.028 |     0.000 |
    |  963 |   963 |      82.2 |        2.2 |  0.031 |    -0.000 |
    |  964 |   964 |      83.6 |        2.8 |  0.024 |     0.000 |
    |  965 |   965 |      83.5 |        0.9 |  0.028 |     0.000 |
    |  966 |   966 |      62.2 |        0.9 |  0.765 |     0.000 |
    |  967 |   967 |      63.5 |        1.8 |  0.602 |     0.000 |
    |  968 |   968 |      64.4 |        0.9 |  0.544 |    -0.000 |
    |  969 |   969 |      64.7 |        2.0 |  0.502 |     0.000 |
    |  970 |   970 |      64.2 |        2.8 |  0.513 |     0.000 |
    |  971 |   971 |      65.9 |        1.8 |  0.412 |    -0.000 |
    |  972 |   972 |      66.4 |        2.8 |  0.368 |     0.000 |
    |  973 |   973 |      67.0 |        1.8 |  0.360 |    -0.000 |
    |  974 |   974 |      67.4 |        2.6 |  0.313 |    -0.000 |
    |  975 |   975 |      67.9 |        1.7 |  0.302 |    -0.000 |
    |  976 |   976 |      66.9 |        3.6 |  0.313 |    -0.000 |
    |  977 |   977 |      67.5 |        0.9 |  0.341 |     0.000 |
    |  978 |   978 |      68.5 |        0.9 |  0.285 |     0.000 |
    |  979 |   979 |      65.4 |        0.9 |  0.468 |     0.000 |
    |  980 |   980 |      73.5 |        0.9 |  0.133 |    -0.000 |
    |  981 |   981 |      73.0 |        1.8 |  0.140 |    -0.000 |
    |  982 |   982 |      73.9 |        1.7 |  0.117 |    -0.000 |
    |  983 |   983 |      73.4 |        2.6 |  0.123 |     0.000 |
    |  984 |   984 |      74.3 |        2.6 |  0.108 |     0.000 |
    |  985 |   985 |      71.9 |        1.8 |  0.161 |     0.000 |
    |  986 |   986 |      74.9 |        1.7 |  0.103 |     0.000 |
    |  987 |   987 |      75.3 |        2.6 |  0.091 |     0.000 |
    |  988 |   988 |      74.6 |        3.5 |  0.094 |    -0.000 |
    |  989 |   989 |      75.6 |        3.5 |  0.082 |    -0.000 |
    |  990 |   990 |      76.2 |        2.6 |  0.080 |     0.000 |
    |  991 |   991 |      89.5 |        0.9 |  0.011 |     0.000 |
    |  992 |   992 |      89.9 |        1.7 |  0.010 |    -0.000 |
    |  993 |   993 |      90.9 |        1.7 |  0.009 |    -0.000 |
    |  994 |   994 |      90.3 |        2.6 |  0.009 |     0.000 |
    |  995 |   995 |      88.9 |        1.7 |  0.012 |     0.000 |
    |  996 |   996 |      91.2 |        2.5 |  0.008 |    -0.000 |
    |  997 |   997 |      91.8 |        1.7 |  0.008 |     0.000 |
    |  998 |   998 |      90.5 |        3.4 |  0.008 |     0.000 |
    |  999 |   999 |      89.6 |        3.4 |  0.009 |    -0.000 |
    | 1000 |  1000 |      89.3 |        2.6 |  0.010 |    -0.000 |
    | 1001 |  1001 |      88.5 |        0.9 |  0.013 |     0.000 |
    | 1002 |  1002 |      87.9 |        1.7 |  0.013 |     0.000 |
    | 1003 |  1003 |      92.4 |        0.9 |  0.007 |     0.000 |
    | 1004 |  1004 |      71.5 |        0.9 |  0.182 |    -0.000 |
    | 1005 |  1005 |      70.5 |        0.9 |  0.208 |     0.000 |
    | 1006 |  1006 |      77.4 |        0.9 |  0.072 |     0.000 |
    | 1007 |  1007 |      76.4 |        0.9 |  0.082 |     0.000 |
    | 1008 |  1008 |      76.8 |        1.7 |  0.077 |     0.000 |
    | 1009 |  1009 |      32.1 |        0.9 |  9.690 |     0.000 |
    | 1010 |  1010 |      33.1 |        1.7 |  9.644 |     0.000 |
    | 1011 |  1011 |      34.2 |        1.7 |  9.569 |     0.000 |
    | 1012 |  1012 |      33.1 |        2.6 |  9.654 |    -0.000 |
    | 1013 |  1013 |      32.1 |        2.6 |  9.714 |     0.000 |
    | 1014 |  1014 |      34.1 |        2.6 |  9.605 |     0.000 |
    | 1015 |  1015 |      35.2 |        1.7 |  9.508 |    -0.000 |
    | 1016 |  1016 |      32.9 |        3.4 |  9.693 |    -0.000 |
    | 1017 |  1017 |      35.2 |        2.6 |  9.519 |    -0.000 |
    | 1018 |  1018 |      32.1 |        1.7 |  9.688 |     0.000 |
    | 1019 |  1019 |      31.0 |        2.6 |  9.749 |     0.000 |
    | 1020 |  1020 |      31.1 |        1.7 |  9.743 |     0.000 |
    | 1021 |  1021 |      31.1 |        0.9 |  9.728 |    -0.000 |
    | 1022 |  1022 |      35.1 |        0.9 |  9.486 |     0.000 |
    | 1023 |  1023 |      14.8 |        0.9 |  9.979 |    -0.000 |
    | 1024 |  1024 |      14.7 |        1.7 |  9.981 |     0.000 |
    | 1025 |  1025 |      15.7 |        1.7 |  9.976 |     0.000 |
    | 1026 |  1026 |      13.6 |        1.7 |  9.983 |    -0.000 |
    | 1027 |  1027 |      14.4 |        2.5 |  9.981 |    -0.000 |
    | 1028 |  1028 |      13.4 |        2.5 |  9.985 |     0.000 |
    | 1029 |  1029 |      12.6 |        1.7 |  9.986 |    -0.000 |
    | 1030 |  1030 |      12.4 |        2.5 |  9.986 |     0.000 |
    | 1031 |  1031 |      16.9 |        0.9 |  9.970 |     0.000 |
    | 1032 |  1032 |      16.9 |        1.9 |  9.973 |    -0.000 |
    | 1033 |  1033 |      16.3 |        2.7 |  9.976 |     0.000 |
    | 1034 |  1034 |      17.3 |        2.6 |  9.971 |    -0.000 |
    | 1035 |  1035 |      17.9 |        1.7 |  9.966 |    -0.000 |
    | 1036 |  1036 |      15.4 |        2.5 |  9.979 |     0.000 |
    | 1037 |  1037 |      15.3 |        3.3 |  9.980 |     0.000 |
    | 1038 |  1038 |      22.2 |        0.9 |  9.935 |     0.000 |
    | 1039 |  1039 |      23.1 |        0.9 |  9.922 |     0.000 |
    | 1040 |  1040 |      24.2 |        1.7 |  9.910 |    -0.000 |
    | 1041 |  1041 |      25.2 |        1.8 |  9.899 |     0.000 |
    | 1042 |  1042 |      24.2 |        2.6 |  9.918 |     0.000 |
    | 1043 |  1043 |      23.2 |        2.6 |  9.926 |     0.000 |
    | 1044 |  1044 |      25.1 |        2.6 |  9.901 |    -0.000 |
    | 1045 |  1045 |      26.1 |        1.8 |  9.878 |     0.000 |
    | 1046 |  1046 |      25.0 |        3.5 |  9.913 |     0.000 |
    | 1047 |  1047 |      26.1 |        2.7 |  9.889 |    -0.000 |
    | 1048 |  1048 |      24.1 |        3.4 |  9.921 |     0.000 |
    | 1049 |  1049 |      23.2 |        1.7 |  9.926 |    -0.000 |
    | 1050 |  1050 |      19.5 |        1.2 |  9.956 |     0.000 |
    | 1051 |  1051 |      10.8 |        0.9 |  9.988 |     0.000 |
    | 1052 |  1052 |       9.6 |        1.8 |  9.990 |     0.000 |
    | 1053 |  1053 |      10.6 |        1.7 |  9.989 |     0.000 |
    | 1054 |  1054 |      10.4 |        2.6 |  9.990 |     0.000 |
    | 1055 |  1055 |      11.4 |        2.6 |  9.989 |     0.000 |
    | 1056 |  1056 |      11.6 |        1.7 |  9.987 |    -0.000 |
    | 1057 |  1057 |      12.2 |        3.4 |  9.988 |     0.000 |
    | 1058 |  1058 |      26.1 |        0.9 |  9.880 |    -0.000 |
    | 1059 |  1059 |      27.1 |        1.8 |  9.863 |    -0.000 |
    | 1060 |  1060 |      29.1 |        0.9 |  9.801 |    -0.000 |
    | 1061 |  1061 |      28.1 |        0.9 |  9.836 |     0.000 |
    | 1062 |  1062 |      30.1 |        1.8 |  9.773 |     0.000 |
    | 1063 |  1063 |       7.8 |        0.9 |  9.993 |     0.000 |
    | 1064 |  1064 |       8.8 |        0.9 |  9.991 |    -0.000 |
    | 1065 |  1065 |       5.8 |        0.9 |  9.994 |    -0.000 |
    | 1066 |  1066 |       5.6 |        1.8 |  9.994 |     0.000 |
    | 1067 |  1067 |       4.7 |        1.8 |  9.995 |     0.000 |
    | 1068 |  1068 |       4.8 |        0.9 |  9.995 |     0.000 |
    | 1069 |  1069 |       3.7 |        1.7 |  9.995 |     0.000 |
    | 1070 |  1070 |      85.5 |        0.9 |  0.020 |     0.000 |
    | 1071 |  1071 |      76.1 |        6.7 |  0.045 |     0.000 |
    | 1072 |  1072 |      33.1 |        6.7 |  9.818 |    -0.000 |
    | 1073 |  1073 |      33.2 |        7.5 |  9.854 |     0.000 |
    | 1074 |  1074 |      33.2 |        8.4 |  9.905 |    -0.000 |
    | 1075 |  1075 |      15.2 |        6.1 |  9.988 |    -0.000 |
    | 1076 |  1076 |      25.1 |        0.9 |  9.893 |    -0.000 |
    | 1077 |  1077 |      34.1 |        0.9 |  9.574 |    -0.000 |
    | 1078 |  1078 |      17.9 |        0.8 |  9.967 |    -0.000 |
    | 1079 |  1079 |      64.3 |        6.2 |  0.320 |     0.000 |
    | 1080 |  1080 |      99.1 |        3.4 |  0.004 |     0.000 |
    | 1081 |  1081 |      13.8 |        0.9 |  9.982 |     0.000 |
    | 1082 |  1082 |       2.9 |        0.8 |  9.995 |    -0.000 |
    | 1083 |  1083 |      91.4 |        0.9 |  0.008 |    -0.000 |
    | 1084 |  1084 |      97.3 |        0.9 |  0.004 |     0.000 |
    | 1085 |  1085 |      59.7 |        9.1 |  0.166 |     0.000 |
    | 1086 |  1086 |      59.3 |        8.2 |  0.345 |    -0.000 |
    | 1087 |  1087 |      60.1 |        8.0 |  0.325 |    -0.000 |
    | 1088 |  1088 |      60.9 |        8.9 |  0.171 |     0.000 |
    | 1089 |  1089 |      61.2 |        8.2 |  0.252 |     0.000 |
    | 1090 |  1090 |      72.3 |        6.8 |  0.077 |    -0.000 |
    | 1091 |  1091 |      19.9 |        6.1 |  9.974 |     0.000 |
    | 1092 |  1092 |      27.6 |        6.4 |  9.917 |     0.000 |
    | 1093 |  1093 |      80.5 |        6.3 |  0.024 |     0.000 |
    | 1094 |  1094 |      11.8 |        0.9 |  9.987 |    -0.000 |
    | 1095 |  1095 |      74.5 |        0.9 |  0.111 |     0.000 |
    | 1096 |  1096 |       6.5 |        6.3 |  9.997 |    -0.000 |
    | 1097 |  1097 |      89.4 |        9.1 |  0.002 |    -0.000 |
    | 1098 |  1098 |      77.9 |        9.2 |  0.008 |     0.000 |
    | 1099 |  1099 |      24.1 |        9.1 |  9.988 |     0.000 |
    | 1100 |  1100 |      81.2 |        3.5 |  0.034 |     0.000 |
    | 1101 |  1101 |      18.6 |        3.4 |  9.967 |    -0.000 |
    | 1102 |  1102 |      96.0 |        2.7 |  0.005 |     0.000 |
    | 1103 |  1103 |      97.2 |        9.2 |  0.001 |    -0.000 |
    | 1104 |  1104 |      95.4 |        9.3 |  0.001 |    -0.000 |
    | 1105 |  1105 |      75.9 |        1.7 |  0.087 |    -0.000 |
    | 1106 |  1106 |      87.7 |        3.5 |  0.012 |    -0.000 |
    | 1107 |  1107 |      82.0 |        3.3 |  0.030 |    -0.000 |
    | 1108 |  1108 |      96.4 |        5.4 |  0.003 |    -0.000 |
    | 1109 |  1109 |      84.9 |        4.2 |  0.018 |    -0.000 |
    | 1110 |  1110 |      78.8 |        4.4 |  0.045 |     0.000 |
    | 1111 |  1111 |      17.5 |        3.6 |  9.972 |     0.000 |
    | 1112 |  1112 |      21.1 |        4.3 |  9.955 |     0.000 |
    | 1113 |  1113 |      26.9 |        3.6 |  9.883 |     0.000 |
    | 1114 |  1114 |      69.6 |        4.4 |  0.186 |    -0.000 |
    | 1115 |  1115 |      72.7 |        3.6 |  0.126 |     0.000 |
    | 1116 |  1116 |      29.8 |        4.4 |  9.832 |    -0.000 |
    | 1117 |  1117 |       9.2 |        4.5 |  9.993 |     0.000 |
    | 1118 |  1118 |      93.4 |        4.3 |  0.005 |     0.000 |
    | 1119 |  1119 |       6.1 |        3.7 |  9.995 |     0.000 |
    | 1120 |  1120 |      64.3 |        3.8 |  0.459 |    -0.000 |
    | 1121 |  1121 |       3.9 |        4.5 |  9.996 |    -0.000 |
    | 1122 |  1122 |      25.1 |        9.1 |  9.985 |    -0.000 |
    | 1123 |  1123 |      30.8 |        4.4 |  9.794 |     0.000 |
    | 1124 |  1124 |      68.7 |        4.3 |  0.225 |     0.000 |
    | 1125 |  1125 |      96.7 |        1.8 |  0.005 |     0.000 |
    | 1126 |  1126 |      92.5 |        4.2 |  0.006 |     0.000 |
    | 1127 |  1127 |      63.3 |        3.7 |  0.546 |     0.000 |
    | 1128 |  1128 |      10.2 |        4.3 |  9.992 |    -0.000 |
    | 1129 |  1129 |      22.0 |        4.2 |  9.950 |    -0.000 |
    | 1130 |  1130 |      77.8 |        4.4 |  0.051 |    -0.000 |
    | 1131 |  1131 |      84.3 |        4.1 |  0.019 |    -0.000 |
    | 1132 |  1132 |      62.6 |        6.2 |  0.406 |    -0.000 |
    | 1133 |  1133 |      48.4 |        4.9 |  6.855 |    -0.000 |
    | 1134 |  1134 |       5.4 |        2.7 |  9.995 |     0.000 |
    | 1135 |  1135 |      97.3 |        5.4 |  0.003 |     0.000 |
    | 1136 |  1136 |      92.4 |        3.4 |  0.006 |    -0.000 |
    | 1137 |  1137 |      68.6 |        3.5 |  0.243 |    -0.000 |
    | 1138 |  1138 |      22.2 |        3.4 |  9.941 |     0.000 |
    | 1139 |  1139 |      77.5 |        3.5 |  0.061 |    -0.000 |
    | 1140 |  1140 |      35.8 |        4.4 |  9.557 |     0.000 |
    | 1141 |  1141 |      36.2 |        1.7 |  9.408 |     0.000 |
    | 1142 |  1142 |      49.0 |        8.7 |  9.315 |     0.000 |
    | 1143 |  1143 |      37.5 |        3.9 |  9.390 |    -0.000 |
    | 1144 |  1144 |      60.2 |        4.3 |  0.819 |     0.000 |
    | 1145 |  1145 |      61.0 |        6.9 |  0.445 |    -0.000 |
    | 1146 |  1146 |      40.1 |        9.0 |  9.834 |    -0.000 |
    | 1147 |  1147 |      42.3 |        2.6 |  8.554 |     0.000 |
    | 1148 |  1148 |      46.1 |        6.7 |  8.631 |    -0.000 |
    | 1149 |  1149 |      30.9 |        3.5 |  9.777 |    -0.000 |
    | 1150 |  1150 |      39.3 |        3.5 |  9.161 |    -0.000 |
    | 1151 |  1151 |      43.7 |        5.7 |  8.749 |     0.000 |
    | 1152 |  1152 |      66.0 |        9.1 |  0.057 |    -0.000 |
    | 1153 |  1153 |      37.1 |        0.9 |  9.295 |    -0.000 |
    | 1154 |  1154 |      95.3 |        0.9 |  0.005 |     0.000 |
    | 1155 |  1155 |      43.3 |        3.4 |  8.418 |     0.000 |
    | 1156 |  1156 |       8.6 |        1.8 |  9.992 |    -0.000 |
    | 1157 |  1157 |      34.1 |        9.2 |  9.944 |    -0.000 |
    | 1158 |  1158 |      58.4 |        8.3 |  0.374 |    -0.000 |
    | 1159 |  1159 |      37.3 |        7.3 |  9.704 |     0.000 |
    | 1160 |  1160 |      48.4 |        5.8 |  7.438 |    -0.000 |
    | 1161 |  1161 |      56.0 |        4.1 |  1.639 |    -0.000 |
    | 1162 |  1162 |      68.9 |        1.8 |  0.264 |     0.000 |
    | 1163 |  1163 |      28.2 |        9.1 |  9.975 |     0.000 |
    | 1164 |  1164 |      47.0 |        5.3 |  7.810 |    -0.000 |
    | 1165 |  1165 |      86.0 |        9.0 |  0.003 |    -0.000 |
    | 1166 |  1166 |      18.7 |        1.5 |  9.963 |     0.000 |
    | 1167 |  1167 |      48.5 |        6.5 |  7.875 |     0.000 |
    | 1168 |  1168 |      54.5 |        3.8 |  2.180 |    -0.000 |
    | 1169 |  1169 |      54.6 |        6.3 |  1.415 |     0.000 |
    | 1170 |  1170 |      10.3 |        3.5 |  9.991 |    -0.000 |
    | 1171 |  1171 |      74.0 |        8.3 |  0.033 |     0.000 |
    | 1172 |  1172 |      90.4 |        9.1 |  0.001 |    -0.000 |
    | 1173 |  1173 |      28.1 |        1.8 |  9.835 |     0.000 |
    | 1174 |  1174 |      59.3 |        6.6 |  0.625 |    -0.000 |
    | 1175 |  1175 |      41.4 |        3.5 |  8.838 |    -0.000 |
    | 1176 |  1176 |      45.1 |        3.6 |  7.921 |    -0.000 |
    | 1177 |  1177 |      86.5 |        0.9 |  0.017 |     0.000 |
    | 1178 |  1178 |      23.3 |        8.3 |  9.979 |    -0.000 |
    | 1179 |  1179 |      78.7 |        1.8 |  0.057 |    -0.000 |
    | 1180 |  1180 |      86.9 |        1.7 |  0.016 |    -0.000 |
    | 1181 |  1181 |      64.1 |        7.1 |  0.256 |    -0.000 |
    | 1182 |  1182 |      61.2 |        0.9 |  0.919 |     0.000 |
    | 1183 |  1183 |      98.2 |        5.4 |  0.003 |    -0.000 |
    | 1184 |  1184 |      14.2 |        8.2 |  9.995 |    -0.000 |
    | 1185 |  1185 |      84.1 |        9.2 |  0.003 |     0.000 |
    | 1186 |  1186 |      40.3 |        2.6 |  8.943 |     0.000 |
    | 1187 |  1187 |       6.6 |        1.8 |  9.994 |    -0.000 |
    | 1188 |  1188 |      93.4 |        9.2 |  0.001 |    -0.000 |
    | 1189 |  1189 |      49.1 |        5.0 |  6.385 |     0.000 |
    | 1190 |  1190 |      31.9 |        3.4 |  9.730 |     0.000 |
    | 1191 |  1191 |      81.9 |        8.3 |  0.009 |     0.000 |
    | 1192 |  1192 |      36.4 |        6.4 |  9.657 |     0.000 |
    | 1193 |  1193 |       0.9 |        1.8 |  9.996 |    -0.000 |
    | 1194 |  1194 |      99.0 |        1.6 |  0.004 |     0.000 |
    | 1195 |  1195 |      98.1 |        9.1 |  0.001 |    -0.000 |
    | 1196 |  1196 |       3.0 |        4.4 |  9.997 |    -0.000 |
    | 1197 |  1197 |      48.0 |        3.0 |  6.402 |    -0.000 |
    | 1198 |  1198 |      69.9 |        1.8 |  0.221 |     0.000 |
    | 1199 |  1199 |      47.9 |        0.8 |  6.280 |     0.000 |
    | 1200 |  1200 |      68.3 |        2.6 |  0.277 |    -0.000 |
    | 1201 |  1201 |      54.0 |        6.7 |  1.319 |    -0.000 |
    | 1202 |  1202 |      66.0 |        4.0 |  0.352 |     0.000 |
    | 1203 |  1203 |      67.0 |        9.2 |  0.048 |    -0.000 |
    | 1204 |  1204 |      46.7 |        3.5 |  7.237 |     0.000 |
    | 1205 |  1205 |      41.3 |        6.1 |  9.247 |    -0.000 |
    | 1206 |  1206 |      94.4 |        0.9 |  0.006 |     0.000 |
    | 1207 |  1207 |      53.0 |        3.0 |  3.046 |     0.000 |
    | 1208 |  1208 |      33.7 |        4.3 |  9.682 |     0.000 |
    | 1209 |  1209 |      50.6 |        2.5 |  4.539 |     0.000 |
    | 1210 |  1210 |      11.3 |        3.4 |  9.989 |    -0.000 |
    | 1211 |  1211 |      46.5 |        6.0 |  8.169 |     0.000 |
    | 1212 |  1212 |      78.5 |        7.6 |  0.021 |     0.000 |
    | 1213 |  1213 |      22.2 |        8.3 |  9.982 |     0.000 |
    | 1214 |  1214 |      57.8 |        4.3 |  1.224 |    -0.000 |
    | 1215 |  1215 |      51.8 |        8.4 |  0.865 |     0.000 |
    | 1216 |  1216 |      84.0 |        1.8 |  0.024 |     0.000 |
    | 1217 |  1217 |      62.0 |        6.9 |  0.384 |     0.000 |
    | 1218 |  1218 |      91.8 |        8.3 |  0.002 |    -0.000 |
    | 1219 |  1219 |      53.6 |        1.0 |  2.991 |     0.000 |
    | 1220 |  1220 |      20.3 |        0.9 |  9.952 |     0.000 |
    | 1221 |  1221 |      21.1 |        9.2 |  9.993 |     0.000 |
    | 1222 |  1222 |      32.8 |        4.2 |  9.717 |    -0.000 |
    | 1223 |  1223 |      63.3 |        6.7 |  0.327 |     0.000 |
    | 1224 |  1224 |      84.9 |        1.8 |  0.021 |    -0.000 |
    | 1225 |  1225 |      74.9 |        9.1 |  0.014 |    -0.000 |
    | 1226 |  1226 |      13.1 |        9.1 |  9.998 |    -0.000 |
    | 1227 |  1227 |      51.3 |        5.2 |  3.042 |     0.000 |
    | 1228 |  1228 |      13.1 |        4.1 |  9.987 |    -0.000 |
    | 1229 |  1229 |      98.2 |        3.6 |  0.004 |     0.000 |
    | 1230 |  1230 |      51.5 |        2.7 |  3.926 |    -0.000 |
    | 1231 |  1231 |      42.1 |        6.6 |  9.240 |     0.000 |
    | 1232 |  1232 |      35.2 |        6.2 |  9.711 |     0.000 |
    | 1233 |  1233 |      83.5 |        9.2 |  0.003 |     0.000 |
    | 1234 |  1234 |      12.2 |        4.1 |  9.988 |     0.000 |
    | 1235 |  1235 |      82.0 |        9.1 |  0.005 |     0.000 |
    | 1236 |  1236 |      48.4 |        8.5 |  9.167 |    -0.000 |
    | 1237 |  1237 |      22.1 |        9.2 |  9.992 |     0.000 |
    | 1238 |  1238 |      51.0 |        4.7 |  3.631 |    -0.000 |
    | 1239 |  1239 |      67.7 |        3.5 |  0.287 |    -0.000 |
    | 1240 |  1240 |      88.3 |        2.6 |  0.012 |     0.000 |
    | 1241 |  1241 |      60.3 |        1.7 |  1.036 |     0.000 |
    | 1242 |  1242 |      46.1 |        2.8 |  7.371 |     0.000 |
    | 1243 |  1243 |      68.0 |        8.4 |  0.078 |     0.000 |
    | 1244 |  1244 |      33.9 |        3.5 |  9.631 |     0.000 |
    | 1245 |  1245 |      93.7 |        1.7 |  0.006 |    -0.000 |
    | 1246 |  1246 |      59.8 |        7.5 |  0.443 |     0.000 |
    | 1247 |  1247 |      51.7 |        7.7 |  1.233 |    -0.000 |
    | 1248 |  1248 |      47.5 |        5.8 |  7.773 |     0.000 |
    | 1249 |  1249 |      21.4 |        1.7 |  9.945 |    -0.000 |
    | 1250 |  1250 |      12.2 |        9.1 |  9.998 |     0.000 |
    | 1251 |  1251 |      84.0 |        3.6 |  0.022 |     0.000 |
    | 1252 |  1252 |      98.1 |        8.2 |  0.001 |     0.000 |
    | 1253 |  1253 |       2.6 |        7.3 |  9.998 |    -0.000 |
    | 1254 |  1254 |      54.9 |        6.8 |  1.140 |    -0.000 |
    | 1255 |  1255 |      92.4 |        9.2 |  0.001 |     0.000 |
    | 1256 |  1256 |      71.5 |        8.5 |  0.043 |     0.000 |
    | 1257 |  1257 |      76.0 |        9.2 |  0.012 |     0.000 |
    | 1258 |  1258 |      42.0 |        5.9 |  9.092 |     0.000 |
    | 1259 |  1259 |      13.2 |        3.3 |  9.985 |    -0.000 |
    | 1260 |  1260 |      20.4 |        8.5 |  9.988 |    -0.000 |
    | 1261 |  1261 |      33.1 |        9.2 |  9.952 |     0.000 |
    | 1262 |  1262 |      52.6 |        8.4 |  0.788 |     0.000 |
    | 1263 |  1263 |      35.1 |        3.6 |  9.575 |    -0.000 |
    | 1264 |  1264 |      67.2 |        4.1 |  0.290 |     0.000 |
    | 1265 |  1265 |      48.7 |        2.9 |  5.980 |    -0.000 |
    | 1266 |  1266 |      98.4 |        2.0 |  0.004 |     0.000 |
    | 1267 |  1267 |      14.1 |        3.4 |  9.984 |    -0.000 |
    | 1268 |  1268 |      18.4 |        9.3 |  9.996 |    -0.000 |
    | 1269 |  1269 |      75.9 |        4.4 |  0.070 |    -0.000 |
    | 1270 |  1270 |      69.0 |        9.2 |  0.032 |    -0.000 |
    | 1271 |  1271 |      11.5 |        8.1 |  9.996 |    -0.000 |
    | 1272 |  1272 |      38.3 |        6.6 |  9.564 |     0.000 |
    | 1273 |  1273 |      19.7 |        8.5 |  9.990 |     0.000 |
    | 1274 |  1274 |      48.3 |        3.6 |  6.467 |    -0.000 |
    | 1275 |  1275 |      15.6 |        3.9 |  9.981 |    -0.000 |
    | 1276 |  1276 |      49.4 |        5.2 |  6.632 |    -0.000 |
    | 1277 |  1277 |      93.8 |        8.4 |  0.001 |     0.000 |
    | 1278 |  1278 |      63.7 |        3.3 |  0.545 |     0.000 |
    | 1279 |  1279 |      55.7 |        8.3 |  0.563 |    -0.000 |
    | 1280 |  1280 |      60.8 |        7.7 |  0.347 |    -0.000 |
    | 1281 |  1281 |       2.8 |        9.1 |  9.999 |    -0.000 |
    | 1282 |  1282 |      82.8 |        3.1 |  0.027 |    -0.000 |
    | 1283 |  1283 |      52.1 |        1.4 |  3.718 |    -0.000 |
    | 1284 |  1284 |      51.7 |        4.9 |  3.067 |     0.000 |
    | 1285 |  1285 |      48.9 |        5.4 |  7.131 |     0.000 |
    | 1286 |  1286 |      76.8 |        8.4 |  0.019 |     0.000 |
    | 1287 |  1287 |      36.3 |        3.4 |  9.459 |     0.000 |
    | 1288 |  1288 |      32.2 |        8.4 |  9.918 |    -0.000 |
    | 1289 |  1289 |      49.1 |        6.0 |  7.371 |    -0.000 |
    | 1290 |  1290 |      50.5 |        4.5 |  4.260 |    -0.000 |
    | 1291 |  1291 |      40.6 |        6.5 |  9.364 |    -0.000 |
    | 1292 |  1292 |      46.0 |        1.7 |  7.218 |    -0.000 |
    | 1293 |  1293 |      65.5 |        3.0 |  0.406 |     0.000 |
    | 1294 |  1294 |      22.3 |        1.7 |  9.933 |    -0.000 |
    | 1295 |  1295 |      92.8 |        1.7 |  0.007 |     0.000 |
    | 1296 |  1296 |      10.3 |        9.3 |  9.999 |    -0.000 |
    | 1297 |  1297 |      53.3 |        8.2 |  0.854 |     0.000 |
    | 1298 |  1298 |      85.6 |        6.7 |  0.010 |    -0.000 |
    | 1299 |  1299 |       1.9 |        1.7 |  9.996 |     0.000 |
    | 1300 |  1300 |      69.7 |        9.2 |  0.028 |    -0.000 |
    | 1301 |  1301 |      62.0 |        8.1 |  0.234 |    -0.000 |
    | 1302 |  1302 |      77.9 |        8.3 |  0.018 |    -0.000 |
    | 1303 |  1303 |      74.7 |        4.5 |  0.084 |     0.000 |
    | 1304 |  1304 |      84.0 |        8.3 |  0.007 |    -0.000 |
    | 1305 |  1305 |      14.8 |        4.2 |  9.983 |     0.000 |
    | 1306 |  1306 |      15.1 |        9.1 |  9.997 |     0.000 |
    | 1307 |  1307 |      42.9 |        6.6 |  9.105 |     0.000 |
    | 1308 |  1308 |      19.4 |        9.2 |  9.995 |    -0.000 |
    | 1309 |  1309 |      37.3 |        3.2 |  9.376 |    -0.000 |
    | 1310 |  1310 |      77.7 |        1.8 |  0.065 |    -0.000 |
    | 1311 |  1311 |      54.9 |        3.3 |  2.118 |     0.000 |
    | 1312 |  1312 |      56.4 |        7.5 |  0.754 |     0.000 |
    | 1313 |  1313 |       2.8 |        5.3 |  9.997 |    -0.000 |
    | 1314 |  1314 |      31.1 |        9.2 |  9.966 |    -0.000 |
    | 1315 |  1315 |      94.8 |        8.5 |  0.001 |     0.000 |
    | 1316 |  1316 |      48.7 |        6.1 |  7.709 |    -0.000 |
    | 1317 |  1317 |      23.9 |        4.3 |  9.932 |    -0.000 |
    | 1318 |  1318 |      45.4 |        5.1 |  8.183 |     0.000 |
    | 1319 |  1319 |      81.5 |        2.5 |  0.035 |     0.000 |
    | 1320 |  1320 |      22.3 |        2.5 |  9.939 |    -0.000 |
    | 1321 |  1321 |      80.5 |        1.8 |  0.043 |     0.000 |
    | 1322 |  1322 |      17.0 |        9.2 |  9.996 |     0.000 |
    | 1323 |  1323 |      61.4 |        7.4 |  0.353 |     0.000 |
    | 1324 |  1324 |      99.1 |        9.1 |  0.001 |    -0.000 |
    | 1325 |  1325 |       0.9 |        9.0 |  9.999 |    -0.000 |
    | 1326 |  1326 |       1.0 |        0.9 |  9.996 |     0.000 |
    | 1327 |  1327 |      99.1 |        0.6 |  0.004 |    -0.000 |
    | 1328 |  1328 |       9.7 |        9.2 |  9.999 |     0.000 |
    | 1329 |  1329 |      48.9 |        9.3 |  9.575 |    -0.000 |
    | 1330 |  1330 |      92.1 |        2.5 |  0.007 |    -0.000 |
    | 1331 |  1331 |       2.8 |        1.7 |  9.996 |     0.000 |
    | 1332 |  1332 |      64.8 |        3.4 |  0.450 |    -0.000 |
    | 1333 |  1333 |      62.4 |        2.0 |  0.727 |    -0.000 |
    | 1334 |  1334 |      90.6 |        4.1 |  0.008 |    -0.000 |
    | 1335 |  1335 |      16.0 |        3.2 |  9.978 |     0.000 |
    | 1336 |  1336 |      18.1 |        2.7 |  9.969 |     0.000 |
    | 1337 |  1337 |      18.0 |        8.9 |  9.994 |     0.000 |
    | 1338 |  1338 |      98.2 |        6.3 |  0.002 |     0.000 |
    | 1339 |  1339 |      55.7 |        3.3 |  1.858 |     0.000 |
    | 1340 |  1340 |      63.0 |        9.2 |  0.092 |     0.000 |
    | 1341 |  1341 |      70.0 |        8.5 |  0.052 |    -0.000 |
    | 1342 |  1342 |      46.0 |        0.9 |  7.202 |     0.000 |
    | 1343 |  1343 |      51.1 |        9.2 |  0.456 |     0.000 |
    | 1344 |  1344 |      78.5 |        9.3 |  0.007 |     0.000 |
    | 1345 |  1345 |      30.1 |        9.2 |  9.971 |    -0.000 |
    | 1346 |  1346 |      24.8 |        4.3 |  9.920 |     0.000 |
    | 1347 |  1347 |      23.1 |        3.4 |  9.935 |     0.000 |
    | 1348 |  1348 |      54.2 |        8.4 |  0.687 |     0.000 |
    | 1349 |  1349 |      73.6 |        3.5 |  0.111 |    -0.000 |
    | 1350 |  1350 |      77.2 |        2.6 |  0.067 |    -0.000 |
    | 1351 |  1351 |      51.7 |        7.0 |  1.752 |     0.000 |
    | 1352 |  1352 |      37.9 |        3.2 |  9.301 |     0.000 |
    | 1353 |  1353 |      95.8 |        8.6 |  0.001 |     0.000 |
    | 1354 |  1354 |      50.6 |        5.0 |  3.649 |    -0.000 |
    | 1355 |  1355 |      98.2 |        4.5 |  0.003 |    -0.000 |
    | 1356 |  1356 |       1.8 |        3.1 |  9.996 |     0.000 |
    | 1357 |  1357 |       8.5 |        8.2 |  9.998 |     0.000 |
    | 1358 |  1358 |      51.1 |        5.7 |  2.584 |    -0.000 |
    | 1359 |  1359 |      48.3 |        7.0 |  8.417 |     0.000 |
    | 1360 |  1360 |      55.5 |        6.8 |  1.101 |    -0.000 |
    | 1361 |  1361 |      34.2 |        7.4 |  9.827 |    -0.000 |
    | 1362 |  1362 |      16.9 |        3.1 |  9.975 |     0.000 |
    | 1363 |  1363 |      56.4 |        3.4 |  1.691 |     0.000 |
    | 1364 |  1364 |      39.3 |        6.3 |  9.449 |    -0.000 |
    | 1365 |  1365 |      98.1 |        1.7 |  0.004 |     0.000 |
    | 1366 |  1366 |       3.5 |        2.6 |  9.996 |    -0.000 |
    | 1367 |  1367 |      51.2 |        2.1 |  4.219 |    -0.000 |
    | 1368 |  1368 |      42.8 |        7.4 |  9.337 |    -0.000 |
    | 1369 |  1369 |      57.5 |        3.5 |  1.408 |     0.000 |
    | 1370 |  1370 |      91.4 |        3.3 |  0.007 |     0.000 |
    | 1371 |  1371 |      63.6 |        2.4 |  0.589 |     0.000 |
    | 1372 |  1372 |      59.4 |        1.6 |  1.167 |    -0.000 |
    | 1373 |  1373 |      60.1 |        2.6 |  0.993 |    -0.000 |
    | 1374 |  1374 |      59.6 |        3.6 |  0.994 |     0.000 |
    | 1375 |  1375 |      89.9 |        4.0 |  0.008 |    -0.000 |
    | 1376 |  1376 |      73.0 |        9.4 |  0.013 |    -0.000 |
    | 1377 |  1377 |      19.7 |        2.0 |  9.958 |     0.000 |
    | 1378 |  1378 |      29.5 |        8.4 |  9.946 |    -0.000 |
    | 1379 |  1379 |      71.3 |        9.4 |  0.016 |    -0.000 |
    | 1380 |  1380 |      59.2 |        4.3 |  0.988 |     0.000 |
    | 1381 |  1381 |      50.3 |        1.9 |  4.827 |    -0.000 |
    | 1382 |  1382 |      53.1 |        0.6 |  3.275 |     0.000 |
    | 1383 |  1383 |      84.8 |        7.4 |  0.009 |    -0.000 |
    | 1384 |  1384 |       3.6 |        8.2 |  9.999 |    -0.000 |
    | 1385 |  1385 |      47.4 |        3.2 |  6.882 |    -0.000 |
    | 1386 |  1386 |      58.6 |        1.6 |  1.354 |     0.000 |
    | 1387 |  1387 |      76.6 |        3.5 |  0.069 |     0.000 |
    | 1388 |  1388 |      38.6 |        3.3 |  9.225 |    -0.000 |
    | 1389 |  1389 |      96.4 |        8.7 |  0.001 |     0.000 |
    | 1390 |  1390 |      50.1 |        4.6 |  4.793 |     0.000 |
    | 1391 |  1391 |      51.3 |        1.5 |  4.168 |     0.000 |
    | 1392 |  1392 |       7.8 |        9.2 |  9.999 |    -0.000 |
    | 1393 |  1393 |      56.1 |        5.9 |  1.254 |    -0.000 |
    | 1394 |  1394 |      86.5 |        6.9 |  0.008 |     0.000 |
    | 1395 |  1395 |      80.0 |        8.9 |  0.008 |     0.000 |
    +------+-------+-----------+------------+--------+-----------+

    Results per element:

    +------+------+------+------+------+-----------+-----------+-----------+--------+--------+
    |   El |    N |    N |    N |    N |        qx |        qy |     q_res |   grad |   grad |
    |   Nb |    1 |    2 |    3 |    4 |   [m/day] |   [m/day] |   [m/day] |    [-] |    [-] |
    |------+------+------+------+------+-----------+-----------+-----------+--------+--------|
    |    1 | 1322 |  209 |  210 |  865 |     0.006 |    -0.085 |     0.120 | -0.000 |  0.004 |
    |    2 |  698 | 1125 |  298 | 1084 |     0.006 |     0.003 |     0.004 | -0.000 | -0.000 |
    |    3 | 1321 |  671 |  958 |  959 |     0.149 |     0.033 |     0.047 | -0.007 | -0.002 |
    |    4 |  366 |  750 |  545 |  318 |    14.769 |    -0.774 |     1.095 | -0.738 |  0.039 |
    |    5 |  301 |  792 |  707 |  705 |     5.389 |     1.325 |     1.873 | -0.269 | -0.066 |
    |    6 |  425 |  713 |  696 | 1284 |     9.321 |    10.830 |    15.316 | -0.466 | -0.542 |
    |    7 | 1349 | 1115 |  633 |  983 |     0.405 |     0.217 |     0.307 | -0.020 | -0.011 |
    |    8 |  863 | 1306 |  247 | 1184 |     0.012 |    -0.064 |     0.090 | -0.001 |  0.003 |
    |    9 | 1045 | 1059 |  634 | 1047 |     0.393 |    -0.140 |     0.198 | -0.020 |  0.007 |
    |   10 | 1159 |  815 | 1192 |  558 |     0.979 |    -1.898 |     2.685 | -0.049 |  0.095 |
    |   11 |  600 | 1067 | 1066 | 1134 |     0.011 |    -0.006 |     0.009 | -0.001 |  0.000 |
    |   12 |  790 | 1159 |  558 | 1272 |     1.139 |    -2.216 |     3.133 | -0.057 |  0.111 |
    |   13 | 1046 | 1044 | 1047 |  525 |     0.312 |    -0.163 |     0.230 | -0.016 |  0.008 |
    |   14 |  844 |  843 |  840 |  246 |     0.010 |     0.041 |     0.058 | -0.000 | -0.002 |
    |   15 |  815 |  816 | 1232 | 1192 |     0.896 |    -1.582 |     2.238 | -0.045 |  0.079 |
    |   16 |  671 | 1179 |  278 |  958 |     0.170 |     0.034 |     0.049 | -0.009 | -0.002 |
    |   17 | 1336 | 1035 | 1166 |  615 |     0.110 |    -0.033 |     0.047 | -0.006 |  0.002 |
    |   18 | 1333 |  678 | 1182 |  966 |     2.595 |     0.610 |     0.862 | -0.130 | -0.030 |
    |   19 | 1186 |  562 | 1150 |  796 |     3.025 |    -1.492 |     2.109 | -0.151 |  0.075 |
    |   20 |  535 | 1193 | 1299 |  755 |     0.003 |    -0.004 |     0.006 | -0.000 |  0.000 |
    |   21 |  923 | 1226 | 1250 |  924 |     0.008 |    -0.047 |     0.067 | -0.000 |  0.002 |
    |   22 | 1342 |   54 |   55 |  817 |     9.306 |    -0.542 |     0.767 | -0.465 |  0.027 |
    |   23 | 1163 |  198 |  199 |  904 |     0.028 |    -0.488 |     0.690 | -0.001 |  0.024 |
    |   24 | 1184 |  247 | 1226 |  923 |     0.012 |    -0.055 |     0.078 | -0.001 |  0.003 |
    |   25 |  956 |  307 | 1338 | 1183 |     0.001 |     0.010 |     0.015 | -0.000 | -0.001 |
    |   26 | 1125 |  686 | 1154 |  298 |     0.008 |     0.003 |     0.005 | -0.000 | -0.000 |
    |   27 | 1195 | 1103 |  681 | 1252 |     0.001 |     0.013 |     0.019 | -0.000 | -0.001 |
    |   28 |  731 |  732 | 1151 |  357 |     3.931 |    -5.666 |     8.013 | -0.197 |  0.283 |
    |   29 |  674 | 1220 |  277 | 1249 |     0.166 |    -0.038 |     0.054 | -0.008 |  0.002 |
    |   30 | 1395 |  912 |  135 |  136 |     0.009 |     0.135 |     0.191 | -0.000 | -0.007 |
    |   31 |  673 | 1245 |  288 | 1206 |     0.014 |     0.004 |     0.006 | -0.001 | -0.000 |
    |   32 |  994 | 1000 |  995 |  992 |     0.030 |     0.012 |     0.016 | -0.002 | -0.001 |
    |   33 |  983 |  633 |  985 |  981 |     0.444 |     0.156 |     0.221 | -0.022 | -0.008 |
    |   34 |  600 | 1366 | 1069 | 1067 |     0.008 |    -0.005 |     0.007 | -0.000 |  0.000 |
    |   35 |  892 |  694 |  891 |  293 |     0.003 |    -0.008 |     0.012 | -0.000 |  0.000 |
    |   36 |  690 | 1312 |  780 | 1279 |     1.846 |     6.052 |     8.559 | -0.092 | -0.303 |
    |   37 |  231 |  232 |  535 |  892 |     0.001 |    -0.007 |     0.009 | -0.000 |  0.000 |
    |   38 | 1261 |  260 | 1288 | 1074 |     0.193 |    -1.071 |     1.515 | -0.010 |  0.054 |
    |   39 |  699 |  265 |  715 |  701 |     7.731 |     1.618 |     2.288 | -0.387 | -0.081 |
    |   40 |  729 |  818 |  733 |  730 |     1.907 |    -6.297 |     8.906 | -0.095 |  0.315 |
    |   41 |  425 |  426 |  429 |  713 |     8.937 |     8.015 |    11.335 | -0.447 | -0.401 |
    |   42 |  366 | 1265 |  742 |  750 |    14.154 |    -2.279 |     3.222 | -0.708 |  0.114 |
    |   43 |  294 |  169 |  170 |  714 |     0.843 |    15.934 |    22.534 | -0.042 | -0.797 |
    |   44 |  255 | 1171 |  869 | 1225 |     0.070 |     0.355 |     0.502 | -0.004 | -0.018 |
    |   45 |  198 | 1163 |  258 |  197 |     0.048 |    -0.561 |     0.794 | -0.002 |  0.028 |
    |   46 |  246 |  840 |  839 | 1097 |     0.007 |     0.036 |     0.051 | -0.000 | -0.002 |
    |   47 |  534 | 1080 | 1229 |  697 |     0.003 |     0.006 |     0.009 | -0.000 | -0.000 |
    |   48 |  129 |  844 |  246 |  128 |     0.005 |     0.046 |     0.065 | -0.000 | -0.002 |
    |   49 |   63 |   64 |  301 |  715 |     6.895 |     0.562 |     0.794 | -0.345 | -0.028 |
    |   50 |  152 |  153 | 1340 |  944 |     0.162 |     1.943 |     2.748 | -0.008 | -0.097 |
    |   51 |  219 |  220 |  241 |  807 |     0.001 |    -0.020 |     0.028 | -0.000 |  0.001 |
    |   52 |   56 | 1199 |  817 |   55 |    10.411 |    -0.394 |     0.557 | -0.521 |  0.020 |
    |   53 |  305 | 1182 |  678 | 1241 |     3.048 |     0.566 |     0.801 | -0.152 | -0.028 |
    |   54 | 1074 | 1073 | 1361 |  814 |     0.409 |    -1.215 |     1.718 | -0.020 |  0.061 |
    |   55 |  686 |  673 | 1206 | 1154 |     0.011 |     0.003 |     0.005 | -0.001 | -0.000 |
    |   56 | 1290 |  757 | 1238 | 1354 |    17.890 |    16.493 |    23.325 | -0.895 | -0.825 |
    |   57 |  173 |  738 | 1289 |  172 |     0.378 |   -16.125 |    22.804 | -0.019 |  0.806 |
    |   58 | 1173 | 1061 | 1060 |  643 |     0.565 |    -0.130 |     0.184 | -0.028 |  0.007 |
    |   59 |  782 |  536 |  785 |  783 |     2.359 |     4.446 |     6.288 | -0.118 | -0.222 |
    |   60 |  897 | 1296 | 1328 |  898 |     0.005 |    -0.032 |     0.045 | -0.000 |  0.002 |
    |   61 |  562 | 1186 |  765 | 1175 |     3.511 |    -1.884 |     2.665 | -0.176 |  0.094 |
    |   62 | 1303 |  486 | 1349 |  988 |     0.303 |     0.215 |     0.304 | -0.015 | -0.011 |
    |   63 |  224 |  225 | 1325 |  823 |    -0.000 |    -0.013 |     0.018 |  0.000 |  0.001 |
    |   64 |  564 |  842 |  843 |  846 |     0.018 |     0.039 |     0.055 | -0.001 | -0.002 |
    |   65 | 1004 |  273 |  981 |  985 |     0.503 |     0.109 |     0.154 | -0.025 | -0.005 |
    |   66 |  287 | 1063 |  647 | 1187 |     0.017 |    -0.004 |     0.006 | -0.001 |  0.000 |
    |   67 |  795 |  304 |  793 |  794 |     3.259 |    -0.660 |     0.933 | -0.163 |  0.033 |
    |   68 | 1240 | 1000 |  999 |  531 |     0.032 |     0.018 |     0.025 | -0.002 | -0.001 |
    |   69 | 1242 |  745 | 1385 | 1204 |     9.434 |    -4.193 |     5.930 | -0.472 |  0.210 |
    |   70 | 1232 |  816 | 1361 |  585 |     0.707 |    -1.309 |     1.851 | -0.035 |  0.065 |
    |   71 | 1363 |  706 | 1369 |  537 |     4.877 |     2.852 |     4.033 | -0.244 | -0.143 |
    |   72 | 1214 |  712 | 1380 |  439 |     3.502 |     2.818 |     3.985 | -0.175 | -0.141 |
    |   73 | 1250 |  245 | 1271 |  924 |     0.010 |    -0.041 |     0.058 | -0.001 |  0.002 |
    |   74 | 1256 |  257 | 1341 |  878 |     0.152 |     0.605 |     0.856 | -0.008 | -0.030 |
    |   75 | 1061 | 1173 | 1059 |  274 |     0.480 |    -0.096 |     0.136 | -0.024 |  0.005 |
    |   76 |  900 |  241 |  901 |  826 |     0.003 |    -0.017 |     0.024 | -0.000 |  0.001 |
    |   77 | 1225 |  869 |  870 | 1257 |     0.056 |     0.307 |     0.434 | -0.003 | -0.015 |
    |   78 | 1147 |  574 | 1175 |  765 |     4.161 |    -2.117 |     2.994 | -0.208 |  0.106 |
    |   79 | 1097 |  839 |  841 | 1172 |     0.006 |     0.031 |     0.044 | -0.000 | -0.002 |
    |   80 | 1285 | 1189 | 1276 |  236 |    12.895 |   -15.506 |    21.929 | -0.645 |  0.775 |
    |   81 |  903 |  256 | 1122 |  883 |     0.087 |    -0.360 |     0.510 | -0.004 |  0.018 |
    |   82 |  748 |  747 |  740 |  739 |    12.503 |    -0.129 |     0.182 | -0.625 |  0.006 |
    |   83 | 1134 | 1066 | 1187 |  635 |     0.013 |    -0.006 |     0.009 | -0.001 |  0.000 |
    |   84 |  727 | 1167 | 1359 |  725 |     4.985 |   -12.325 |    17.430 | -0.249 |  0.616 |
    |   85 |  766 |  578 | 1155 |  761 |     5.617 |    -2.759 |     3.902 | -0.281 |  0.138 |
    |   86 |  767 | 1176 |  578 |  766 |     6.499 |    -3.314 |     4.687 | -0.325 |  0.166 |
    |   87 | 1220 |  674 | 1377 | 1050 |     0.146 |    -0.032 |     0.046 | -0.007 |  0.002 |
    |   88 |  764 |  240 |  762 |  763 |     4.459 |    -0.888 |     1.256 | -0.223 |  0.044 |
    |   89 |  457 | 1202 |  976 | 1264 |     0.991 |     0.719 |     1.017 | -0.050 | -0.036 |
    |   90 |  317 |  756 |  803 |  315 |    22.732 |     0.015 |     0.022 | -1.137 | -0.001 |
    |   91 | 1290 |  756 |  598 |  757 |    22.137 |    10.057 |    14.223 | -1.107 | -0.503 |
    |   92 |   78 |   79 | 1004 | 1005 |     0.644 |     0.049 |     0.069 | -0.032 | -0.002 |
    |   93 |  943 |  941 | 1152 |  261 |     0.289 |     1.428 |     2.019 | -0.014 | -0.071 |
    |   94 |  719 |  754 | 1236 |  602 |     1.295 |   -10.240 |    14.482 | -0.065 |  0.512 |
    |   95 | 1211 | 1164 | 1248 |  726 |     6.196 |    -9.355 |    13.229 | -0.310 |  0.468 |
    |   96 | 1063 | 1064 | 1156 |  647 |     0.021 |    -0.006 |     0.008 | -0.001 |  0.000 |
    |   97 |  531 | 1106 |  639 | 1240 |     0.037 |     0.019 |     0.027 | -0.002 | -0.001 |
    |   98 |  915 |  914 | 1395 |  251 |     0.030 |     0.164 |     0.231 | -0.001 | -0.008 |
    |   99 |  848 |  845 |  844 | 1165 |     0.014 |     0.053 |     0.075 | -0.001 | -0.003 |
    |  100 | 1331 | 1069 | 1366 |  677 |     0.007 |    -0.005 |     0.007 | -0.000 |  0.000 |
    |  101 |  750 |  742 |  741 |  740 |    12.902 |    -1.155 |     1.633 | -0.645 |  0.058 |
    |  102 |  262 |  191 |  192 | 1157 |     0.118 |    -1.445 |     2.044 | -0.006 |  0.072 |
    |  103 | 1048 | 1046 | 1346 | 1317 |     0.240 |    -0.168 |     0.238 | -0.012 |  0.008 |
    |  104 |  736 |  168 |  169 |  294 |     1.178 |    13.614 |    19.253 | -0.059 | -0.681 |
    |  105 |  738 |  173 |  174 |  295 |     1.345 |   -13.854 |    19.592 | -0.067 |  0.693 |
    |  106 |  985 |  644 | 1005 | 1004 |     0.581 |     0.117 |     0.166 | -0.029 | -0.006 |
    |  107 |  538 |  621 |  679 |  624 |     5.671 |    12.143 |    17.173 | -0.284 | -0.607 |
    |  108 |  748 |  749 | 1381 |  747 |    13.031 |     0.486 |     0.687 | -0.652 | -0.024 |
    |  109 |  679 |  695 |  696 |  624 |     7.642 |    13.382 |    18.925 | -0.382 | -0.669 |
    |  110 | 1278 |  970 | 1332 | 1120 |     1.573 |     0.923 |     1.305 | -0.079 | -0.046 |
    |  111 | 1345 |  258 | 1378 |  953 |     0.144 |    -0.689 |     0.975 | -0.007 |  0.034 |
    |  112 |  726 |  725 |  723 |  724 |     4.270 |    -9.712 |    13.735 | -0.213 |  0.486 |
    |  113 |  301 |  703 |  701 |  715 |     6.786 |     1.675 |     2.369 | -0.339 | -0.084 |
    |  114 |  302 |  720 |  754 |  719 |     1.025 |    -9.331 |    13.196 | -0.051 |  0.467 |
    |  115 |  748 |   59 |   60 |  296 |    11.707 |     0.412 |     0.583 | -0.585 | -0.021 |
    |  116 |  598 |  756 |  317 |  358 |    18.814 |     3.253 |     4.601 | -0.941 | -0.163 |
    |  117 |  761 | 1155 |  574 | 1147 |     4.853 |    -2.486 |     3.516 | -0.243 |  0.124 |
    |  118 |  643 | 1060 |  272 | 1062 |     0.658 |    -0.130 |     0.184 | -0.033 |  0.007 |
    |  119 |  238 |  729 |  721 |  788 |     1.165 |    -7.170 |    10.140 | -0.058 |  0.358 |
    |  120 |  304 |  795 |  798 |  820 |     2.783 |    -0.608 |     0.860 | -0.139 |  0.030 |
    |  121 |  244 | 1218 |  833 | 1255 |     0.004 |     0.023 |     0.033 | -0.000 | -0.001 |
    |  122 |   48 |   49 |  240 |  793 |     3.807 |    -0.235 |     0.333 | -0.190 |  0.012 |
    |  123 |  580 |  620 | 1253 |  829 |     0.003 |    -0.013 |     0.018 | -0.000 |  0.001 |
    |  124 |  739 |  740 |  741 |  297 |    12.217 |    -0.787 |     1.112 | -0.611 |  0.039 |
    |  125 |  190 |  812 |  805 |  189 |     0.163 |    -1.968 |     2.784 | -0.008 |  0.098 |
    |  126 | 1235 | 1191 |  918 |  250 |     0.021 |     0.102 |     0.145 | -0.001 | -0.005 |
    |  127 |  713 |  429 |  464 |  538 |     6.857 |     9.446 |    13.358 | -0.343 | -0.472 |
    |  128 |  696 |  713 |  538 |  624 |     7.728 |    10.607 |    15.001 | -0.386 | -0.530 |
    |  129 |  676 | 1338 |  307 |  825 |     0.002 |     0.011 |     0.016 | -0.000 | -0.001 |
    |  130 | 1052 | 1156 | 1064 |  285 |     0.025 |    -0.005 |     0.008 | -0.001 |  0.000 |
    |  131 |  221 |  222 |  308 |  901 |     0.001 |    -0.016 |     0.023 | -0.000 |  0.001 |
    |  132 | 1062 | 1020 | 1019 |  662 |     0.736 |    -0.254 |     0.360 | -0.037 |  0.013 |
    |  133 | 1274 |  424 | 1385 | 1197 |    11.562 |    -5.207 |     7.364 | -0.578 |  0.260 |
    |  134 |  689 | 1279 |  237 |  751 |     1.058 |     6.936 |     9.809 | -0.053 | -0.347 |
    |  135 | 1186 |  796 |  795 |  794 |     3.206 |    -1.181 |     1.670 | -0.160 |  0.059 |
    |  136 |   57 |  297 | 1199 |   56 |    11.436 |    -0.431 |     0.609 | -0.572 |  0.022 |
    |  137 |  714 |  170 |  235 | 1358 |     3.812 |    20.410 |    28.863 | -0.191 | -1.020 |
    |  138 |  759 |  753 |  767 |  766 |     6.864 |    -2.302 |     3.256 | -0.343 |  0.115 |
    |  139 | 1132 | 1223 |  947 | 1217 |     1.123 |     1.974 |     2.792 | -0.056 | -0.099 |
    |  140 |  297 |   57 |   58 |  739 |    12.043 |    -0.127 |     0.179 | -0.602 |  0.006 |
    |  141 |  739 |   58 |   59 |  748 |    12.304 |    -0.094 |     0.133 | -0.615 |  0.005 |
    |  142 |  735 |  303 |  162 |  163 |     0.712 |     8.730 |    12.347 | -0.036 | -0.437 |
    |  143 |  719 |  179 |  180 |  302 |     0.589 |    -8.762 |    12.392 | -0.029 |  0.438 |
    |  144 |  759 |  752 |  266 |  753 |     7.007 |    -1.433 |     2.027 | -0.350 |  0.072 |
    |  145 |   62 |  265 |  699 | 1219 |     8.662 |     0.910 |     1.288 | -0.433 | -0.046 |
    |  146 | 1342 |  266 |   53 |   54 |     8.119 |    -0.480 |     0.679 | -0.406 |  0.024 |
    |  147 |  715 |  265 |   62 |   63 |     7.946 |     0.468 |     0.663 | -0.397 | -0.023 |
    |  148 |  788 |  721 |  720 |  302 |     1.480 |    -8.237 |    11.648 | -0.074 |  0.412 |
    |  149 | 1321 |  960 | 1319 |  596 |     0.122 |     0.046 |     0.065 | -0.006 | -0.002 |
    |  150 |  255 | 1376 |  876 |  874 |     0.069 |     0.453 |     0.641 | -0.003 | -0.023 |
    |  151 |  778 |  769 |  290 |  770 |     1.041 |    -4.586 |     6.485 | -0.052 |  0.229 |
    |  152 |  621 |  538 |  464 |  547 |     5.226 |     9.631 |    13.620 | -0.261 | -0.482 |
    |  153 |  755 | 1299 | 1331 |  677 |     0.004 |    -0.004 |     0.006 | -0.000 |  0.000 |
    |  154 |  723 |  720 |  721 |  722 |     2.257 |    -8.573 |    12.125 | -0.113 |  0.429 |
    |  155 | 1147 |  765 |  764 |  763 |     4.346 |    -1.576 |     2.228 | -0.217 |  0.079 |
    |  156 |  820 |  798 |  800 | 1153 |     2.365 |    -0.472 |     0.668 | -0.118 |  0.024 |
    |  157 |  504 |  507 |  564 |  851 |     0.027 |     0.037 |     0.053 | -0.001 | -0.002 |
    |  158 |  366 |  316 |  680 | 1265 |    15.330 |    -3.248 |     4.593 | -0.766 |  0.162 |
    |  159 |  989 | 1269 | 1303 |  988 |     0.258 |     0.183 |     0.258 | -0.013 | -0.009 |
    |  160 |   50 |   51 |  300 |  762 |     5.201 |    -0.319 |     0.452 | -0.260 |  0.016 |
    |  161 |  751 |  237 |  160 |  161 |     0.510 |     6.695 |     9.468 | -0.025 | -0.335 |
    |  162 |  788 |  181 |  182 |  238 |     0.523 |    -6.783 |     9.593 | -0.026 |  0.339 |
    |  163 |  525 | 1047 |  634 | 1113 |     0.371 |    -0.203 |     0.288 | -0.019 |  0.010 |
    |  164 |  603 |  735 |  163 |  164 |     0.085 |     9.934 |    14.049 | -0.004 | -0.497 |
    |  165 |  602 |  178 |  179 |  719 |     0.083 |    -9.852 |    13.933 | -0.004 |  0.493 |
    |  166 | 1165 |  248 |  850 |  848 |     0.012 |     0.063 |     0.089 | -0.001 | -0.003 |
    |  167 | 1141 |  268 | 1153 |  800 |     2.012 |    -0.436 |     0.616 | -0.101 |  0.022 |
    |  168 |  189 |  805 |  264 |  188 |     0.137 |    -2.314 |     3.272 | -0.007 |  0.116 |
    |  169 |  806 |  813 |  816 |  815 |     0.572 |    -1.676 |     2.370 | -0.029 |  0.084 |
    |  170 |  740 |  747 | 1381 |  750 |    13.213 |    -0.611 |     0.864 | -0.661 |  0.031 |
    |  171 |  752 |  300 |   51 |   52 |     6.055 |    -0.432 |     0.611 | -0.303 |  0.022 |
    |  172 |  792 |  301 |   64 |   65 |     5.969 |     0.568 |     0.804 | -0.298 | -0.028 |
    |  173 |  844 |  845 |  846 |  843 |     0.015 |     0.047 |     0.066 | -0.001 | -0.002 |
    |  174 |  237 |  779 |  159 |  160 |     0.254 |     5.882 |     8.318 | -0.013 | -0.294 |
    |  175 |  883 | 1122 | 1099 |  882 |     0.060 |    -0.307 |     0.433 | -0.003 |  0.015 |
    |  176 | 1177 |  284 | 1002 | 1180 |     0.046 |     0.009 |     0.013 | -0.002 | -0.000 |
    |  177 |  952 | 1314 | 1345 |  953 |     0.132 |    -0.796 |     1.125 | -0.007 |  0.040 |
    |  178 |  266 |  752 |   52 |   53 |     7.064 |    -0.438 |     0.619 | -0.353 |  0.022 |
    |  179 |  261 |  944 |  945 |  943 |     0.386 |     1.719 |     2.431 | -0.019 | -0.086 |
    |  180 |  728 | 1359 |  295 |  737 |     2.509 |   -11.922 |    16.860 | -0.125 |  0.596 |
    |  181 |  941 |  940 | 1203 | 1152 |     0.262 |     1.233 |     1.743 | -0.013 | -0.062 |
    |  182 |  766 |  761 |  760 |  759 |     5.897 |    -2.083 |     2.945 | -0.295 |  0.104 |
    |  183 |  752 |  759 |  760 |  300 |     6.067 |    -1.195 |     1.690 | -0.303 |  0.060 |
    |  184 |  729 |  238 |  768 |  818 |     1.281 |    -6.161 |     8.712 | -0.064 |  0.308 |
    |  185 |  184 |  769 |  768 |  183 |     0.416 |    -5.071 |     7.171 | -0.021 |  0.254 |
    |  186 |  239 |  792 |   65 |   66 |     4.980 |     0.294 |     0.416 | -0.249 | -0.015 |
    |  187 |  303 |  751 |  161 |  162 |     0.382 |     7.772 |    10.992 | -0.019 | -0.389 |
    |  188 |  158 |  159 |  779 |  787 |     0.402 |     5.038 |     7.125 | -0.020 | -0.252 |
    |  189 | 1292 |  746 |  745 | 1242 |     9.344 |    -2.891 |     4.088 | -0.467 |  0.145 |
    |  190 |  773 |  770 |  771 |  772 |     1.587 |    -3.958 |     5.597 | -0.079 |  0.198 |
    |  191 | 1148 |  722 |  730 |  731 |     3.209 |    -7.354 |    10.400 | -0.160 |  0.368 |
    |  192 |  168 |  736 |  802 |  167 |     0.044 |    11.997 |    16.966 | -0.002 | -0.600 |
    |  193 |  662 |  611 |  643 | 1062 |     0.636 |    -0.240 |     0.339 | -0.032 |  0.012 |
    |  194 |  240 |  764 |  794 |  793 |     3.816 |    -0.831 |     1.175 | -0.191 |  0.042 |
    |  195 | 1194 | 1266 | 1365 |  649 |     0.003 |     0.003 |     0.004 | -0.000 | -0.000 |
    |  196 |  789 |  264 |  805 |  804 |     0.522 |    -2.341 |     3.311 | -0.026 |  0.117 |
    |  197 | 1158 | 1086 | 1085 |  289 |     0.746 |     3.943 |     5.576 | -0.037 | -0.197 |
    |  198 |  939 |  563 |  937 |  938 |     0.530 |     1.052 |     1.488 | -0.027 | -0.053 |
    |  199 |  731 |  734 | 1211 | 1148 |     4.865 |    -7.689 |    10.874 | -0.243 |  0.384 |
    |  200 |  187 |  811 | 1146 |  186 |     0.201 |    -3.198 |     4.523 | -0.010 |  0.160 |
    |  201 |   76 |   77 |  271 |  978 |     0.878 |     0.065 |     0.091 | -0.044 | -0.003 |
    |  202 | 1176 |  767 | 1242 |  570 |     7.619 |    -3.789 |     5.359 | -0.381 |  0.189 |
    |  203 |  769 |  778 |  818 |  768 |     0.957 |    -5.335 |     7.544 | -0.048 |  0.267 |
    |  204 |  460 |  937 |  563 |  458 |     0.605 |     0.902 |     1.276 | -0.030 | -0.045 |
    |  205 |  333 | 1155 |  578 |  331 |     5.281 |    -3.537 |     5.002 | -0.264 |  0.177 |
    |  206 | 1272 |  341 | 1364 |  791 |     1.551 |    -2.355 |     3.331 | -0.078 |  0.118 |
    |  207 | 1055 | 1056 | 1029 | 1030 |     0.039 |    -0.013 |     0.019 | -0.002 |  0.001 |
    |  208 |  514 |  836 |  544 |  511 |     0.015 |     0.024 |     0.034 | -0.001 | -0.001 |
    |  209 |  238 |  182 |  183 |  768 |     0.290 |    -5.951 |     8.417 | -0.014 |  0.298 |
    |  210 | 1340 |  946 |  945 |  944 |     0.374 |     1.978 |     2.797 | -0.019 | -0.099 |
    |  211 |  269 |  977 |  975 |  973 |     1.084 |     0.217 |     0.306 | -0.054 | -0.011 |
    |  212 |  633 |  601 |  644 |  985 |     0.529 |     0.195 |     0.276 | -0.026 | -0.010 |
    |  213 |  937 |  460 |  462 |  935 |     0.508 |     0.802 |     1.134 | -0.025 | -0.040 |
    |  214 |  806 |  812 |  262 |  813 |     0.379 |    -1.720 |     2.433 | -0.019 |  0.086 |
    |  215 |  837 |  838 |  899 |  544 |     0.016 |     0.030 |     0.043 | -0.001 | -0.002 |
    |  216 |  683 |  900 |  626 |  819 |     0.006 |    -0.019 |     0.026 | -0.000 |  0.001 |
    |  217 |  563 |  939 |  942 |  955 |     0.671 |     1.220 |     1.726 | -0.034 | -0.061 |
    |  218 |  362 |  425 | 1284 |  758 |    11.734 |     8.654 |    12.238 | -0.587 | -0.433 |
    |  219 |  762 |  240 |   49 |   50 |     4.442 |    -0.338 |     0.477 | -0.222 |  0.017 |
    |  220 |  810 |  239 |   66 |   67 |     4.289 |     0.289 |     0.408 | -0.214 | -0.014 |
    |  221 | 1062 |  272 | 1021 | 1020 |     0.773 |    -0.174 |     0.246 | -0.039 |  0.009 |
    |  222 |  809 |  628 |  448 | 1144 |     2.525 |     1.889 |     2.671 | -0.126 | -0.094 |
    |  223 |  564 |  899 |  838 |  842 |     0.017 |     0.035 |     0.049 | -0.001 | -0.002 |
    |  224 |  928 |  557 |  887 |  888 |     0.085 |    -0.176 |     0.249 | -0.004 |  0.009 |
    |  225 |  241 |  900 |  683 |  807 |     0.003 |    -0.019 |     0.027 | -0.000 |  0.001 |
    |  226 |  878 |  931 |  467 |  877 |     0.260 |     0.571 |     0.807 | -0.013 | -0.029 |
    |  227 | 1191 | 1235 |  912 |  910 |     0.023 |     0.115 |     0.163 | -0.001 | -0.006 |
    |  228 |  302 |  180 |  181 |  788 |     0.290 |    -7.860 |    11.115 | -0.014 |  0.393 |
    |  229 |  836 |  514 |  516 |  834 |     0.012 |     0.021 |     0.029 | -0.001 | -0.001 |
    |  230 |  979 |  269 |  973 |  971 |     1.279 |     0.282 |     0.399 | -0.064 | -0.014 |
    |  231 |  888 |  859 |  858 |  928 |     0.067 |    -0.148 |     0.209 | -0.003 |  0.007 |
    |  232 |  933 |  590 |  931 |  932 |     0.293 |     0.704 |     0.996 | -0.015 | -0.035 |
    |  233 | 1172 |  841 | 1218 |  244 |     0.005 |     0.027 |     0.039 | -0.000 | -0.001 |
    |  234 |  229 |  230 |  293 |  890 |     0.001 |    -0.010 |     0.014 | -0.000 |  0.000 |
    |  235 |  793 |  304 |   47 |   48 |     3.246 |    -0.255 |     0.361 | -0.162 |  0.013 |
    |  236 | 1333 |  966 |  267 |  967 |     2.165 |     0.492 |     0.696 | -0.108 | -0.025 |
    |  237 |  305 |  810 |   67 |   68 |     3.662 |     0.199 |     0.282 | -0.183 | -0.010 |
    |  238 |  917 |  908 |  906 |  916 |     0.040 |     0.081 |     0.114 | -0.002 | -0.004 |
    |  239 |  452 |  454 |  955 | 1079 |     1.018 |     1.273 |     1.800 | -0.051 | -0.064 |
    |  240 |  928 |  858 | 1091 |  389 |     0.087 |    -0.139 |     0.197 | -0.004 |  0.007 |
    |  241 | 1272 |  791 |  777 |  790 |     1.241 |    -2.513 |     3.553 | -0.062 |  0.126 |
    |  242 |  304 |  820 |   46 |   47 |     2.777 |    -0.171 |     0.241 | -0.139 |  0.009 |
    |  243 | 1029 | 1056 | 1094 |  283 |     0.042 |    -0.010 |     0.014 | -0.002 |  0.000 |
    |  244 | 1008 | 1105 |  276 | 1007 |     0.271 |     0.056 |     0.080 | -0.014 | -0.003 |
    |  245 |  282 |  965 |   91 |   92 |     0.081 |     0.005 |     0.007 | -0.004 | -0.000 |
    |  246 |  862 |  860 |  853 |  857 |     0.033 |    -0.087 |     0.123 | -0.002 |  0.004 |
    |  247 |  626 |  893 | 1096 |  819 |     0.007 |    -0.016 |     0.023 | -0.000 |  0.001 |
    |  248 |  921 |  927 |  926 |  920 |     0.017 |    -0.039 |     0.055 | -0.001 |  0.002 |
    |  249 |  903 |  883 |  880 |  905 |     0.128 |    -0.355 |     0.502 | -0.006 |  0.018 |
    |  250 |  554 |  852 |  853 |  860 |     0.044 |    -0.083 |     0.117 | -0.002 |  0.004 |
    |  251 |  884 |  886 |  385 |  383 |     0.136 |    -0.197 |     0.279 | -0.007 |  0.010 |
    |  252 |  828 |  890 |  293 |  891 |     0.001 |    -0.010 |     0.014 | -0.000 |  0.000 |
    |  253 | 1096 |  893 |  421 |  419 |     0.009 |    -0.016 |     0.022 | -0.000 |  0.001 |
    |  254 |  905 |  582 | 1092 |  902 |     0.213 |    -0.392 |     0.554 | -0.011 |  0.020 |
    |  255 |  684 |  613 |  413 |  894 |     0.011 |    -0.025 |     0.035 | -0.001 |  0.001 |
    |  256 |  557 |  886 |  885 |  887 |     0.094 |    -0.198 |     0.280 | -0.005 |  0.010 |
    |  257 | 1010 | 1011 | 1014 | 1012 |     1.206 |    -0.441 |     0.624 | -0.060 |  0.022 |
    |  258 |  867 |  866 |  475 |  559 |     0.184 |     0.291 |     0.412 | -0.009 | -0.015 |
    |  259 | 1010 |  270 | 1077 | 1011 |     1.249 |    -0.242 |     0.343 | -0.062 |  0.012 |
    |  260 | 1075 |  925 |  403 |  401 |     0.037 |    -0.051 |     0.071 | -0.002 |  0.003 |
    |  261 |   72 |   73 |  979 |  968 |     1.661 |     0.128 |     0.181 | -0.083 | -0.006 |
    |  262 |  561 |  830 |  672 |  831 |     0.007 |     0.018 |     0.026 | -0.000 | -0.001 |
    |  263 | 1049 | 1039 |  275 | 1040 |     0.259 |    -0.049 |     0.069 | -0.013 |  0.002 |
    |  264 | 1321 |  959 |  279 |  960 |     0.127 |     0.028 |     0.039 | -0.006 | -0.001 |
    |  265 |  267 |  966 |   70 |   71 |     2.315 |     0.200 |     0.283 | -0.116 | -0.010 |
    |  266 |  156 |  157 |  289 | 1085 |     0.309 |     3.750 |     5.303 | -0.015 | -0.187 |
    |  267 |  968 |  979 |  971 |  969 |     1.506 |     0.331 |     0.468 | -0.075 | -0.017 |
    |  268 | 1022 | 1015 | 1011 | 1077 |     1.467 |    -0.313 |     0.442 | -0.073 |  0.016 |
    |  269 |  871 |  929 |  587 |  930 |     0.093 |     0.211 |     0.298 | -0.005 | -0.011 |
    |  270 |  877 |  467 |  468 | 1090 |     0.275 |     0.493 |     0.697 | -0.014 | -0.025 |
    |  271 |  269 |  979 |   73 |   74 |     1.405 |     0.092 |     0.129 | -0.070 | -0.005 |
    |  272 |   90 |   91 |  965 |  961 |     0.096 |     0.007 |     0.010 | -0.005 | -0.000 |
    |  273 |  353 |  351 |  585 | 1072 |     0.677 |    -1.019 |     1.442 | -0.034 |  0.051 |
    |  274 |  835 |  841 |  839 |  837 |     0.010 |     0.029 |     0.041 | -0.000 | -0.001 |
    |  275 | 1157 |  814 |  813 |  262 |     0.278 |    -1.471 |     2.080 | -0.014 |  0.074 |
    |  276 |  880 |  879 |  582 |  905 |     0.184 |    -0.343 |     0.485 | -0.009 |  0.017 |
    |  277 |  270 | 1010 | 1018 | 1009 |     1.065 |    -0.235 |     0.333 | -0.053 |  0.012 |
    |  278 | 1045 | 1058 |  274 | 1059 |     0.414 |    -0.096 |     0.136 | -0.021 |  0.005 |
    |  279 |  929 |  871 |  868 | 1071 |     0.111 |     0.239 |     0.338 | -0.006 | -0.012 |
    |  280 |  837 |  839 |  840 |  838 |     0.011 |     0.033 |     0.047 | -0.001 | -0.002 |
    |  281 |  925 | 1075 |  861 |  864 |     0.030 |    -0.056 |     0.079 | -0.002 |  0.003 |
    |  282 |  830 |  606 |  670 |  672 |     0.005 |     0.016 |     0.022 | -0.000 | -0.001 |
    |  283 |  276 | 1095 |   82 |   83 |     0.344 |     0.025 |     0.036 | -0.017 | -0.001 |
    |  284 |  957 |  670 |  606 |  607 |     0.004 |     0.015 |     0.021 | -0.000 | -0.001 |
    |  285 |  561 |  518 |  520 |  830 |     0.008 |     0.016 |     0.023 | -0.000 | -0.001 |
    |  286 | 1075 |  554 |  860 |  861 |     0.039 |    -0.068 |     0.096 | -0.002 |  0.003 |
    |  287 | 1187 | 1066 | 1065 |  287 |     0.015 |    -0.004 |     0.006 | -0.001 |  0.000 |
    |  288 |  619 |  626 |  900 |  826 |     0.003 |    -0.016 |     0.023 | -0.000 |  0.001 |
    |  289 | 1058 | 1045 | 1041 | 1076 |     0.352 |    -0.069 |     0.098 | -0.018 |  0.003 |
    |  290 | 1071 |  868 |  867 |  559 |     0.144 |     0.266 |     0.377 | -0.007 | -0.013 |
    |  291 |  938 |  940 |  941 |  939 |     0.402 |     1.160 |     1.640 | -0.020 | -0.058 |
    |  292 |  594 |  954 |  369 |  367 |     0.415 |    -0.683 |     0.967 | -0.021 |  0.034 |
    |  293 |  879 |  880 |  881 |  548 |     0.155 |    -0.289 |     0.409 | -0.008 |  0.014 |
    |  294 | 1048 | 1042 | 1044 | 1046 |     0.276 |    -0.146 |     0.206 | -0.014 |  0.007 |
    |  295 |  684 |  894 |  591 |  895 |     0.012 |    -0.027 |     0.038 | -0.001 |  0.001 |
    |  296 |  660 | 1052 | 1053 | 1054 |     0.028 |    -0.010 |     0.014 | -0.001 |  0.001 |
    |  297 | 1076 | 1041 | 1040 |  275 |     0.304 |    -0.069 |     0.097 | -0.015 |  0.003 |
    |  298 |  898 |  895 |  896 |  897 |     0.009 |    -0.032 |     0.045 | -0.000 |  0.002 |
    |  299 |  830 |  520 |  526 |  606 |     0.006 |     0.015 |     0.021 | -0.000 | -0.001 |
    |  300 |  867 |  872 |  581 |  866 |     0.193 |     0.349 |     0.493 | -0.010 | -0.017 |
    |  301 |  916 |  906 |  496 |  498 |     0.057 |     0.076 |     0.107 | -0.003 | -0.004 |
    |  302 |  943 |  942 |  939 |  941 |     0.500 |     1.386 |     1.960 | -0.025 | -0.069 |
    |  303 |  913 |  919 |  488 |  490 |     0.080 |     0.149 |     0.211 | -0.004 | -0.007 |
    |  304 |  554 |  397 |  395 |  852 |     0.060 |    -0.075 |     0.106 | -0.003 |  0.004 |
    |  305 |  899 |  564 |  507 |  509 |     0.023 |     0.033 |     0.046 | -0.001 | -0.002 |
    |  306 |  926 |  591 |  411 |  409 |     0.017 |    -0.030 |     0.042 | -0.001 |  0.001 |
    |  307 |  571 |  920 |  407 |  405 |     0.025 |    -0.039 |     0.055 | -0.001 |  0.002 |
    |  308 |  929 |  481 |  488 |  587 |     0.109 |     0.196 |     0.277 | -0.005 | -0.010 |
    |  309 |  557 |  928 |  389 |  387 |     0.099 |    -0.155 |     0.219 | -0.005 |  0.008 |
    |  310 | 1071 |  479 |  481 |  929 |     0.139 |     0.216 |     0.305 | -0.007 | -0.011 |
    |  311 |  886 |  557 |  387 |  385 |     0.123 |    -0.186 |     0.264 | -0.006 |  0.009 |
    |  312 | 1036 | 1025 | 1032 | 1033 |     0.073 |    -0.027 |     0.039 | -0.004 |  0.001 |
    |  313 |  834 |  516 |  518 |  561 |     0.010 |     0.019 |     0.026 | -0.000 | -0.001 |
    |  314 | 1018 | 1010 | 1012 | 1013 |     1.013 |    -0.344 |     0.487 | -0.051 |  0.017 |
    |  315 |  879 |  548 |  381 |  379 |     0.197 |    -0.275 |     0.389 | -0.010 |  0.014 |
    |  316 |  920 |  571 |  922 |  921 |     0.020 |    -0.042 |     0.059 | -0.001 |  0.002 |
    |  317 | 1092 |  582 |  377 |  375 |     0.280 |    -0.366 |     0.518 | -0.014 |  0.018 |
    |  318 | 1072 |  948 |  355 |  353 |     0.587 |    -0.909 |     1.286 | -0.029 |  0.045 |
    |  319 |  860 |  862 |  863 |  861 |     0.025 |    -0.073 |     0.103 | -0.001 |  0.004 |
    |  320 |  256 |  903 |  693 |  904 |     0.086 |    -0.421 |     0.595 | -0.004 |  0.021 |
    |  321 |  681 |  957 |  607 |  632 |     0.002 |     0.013 |     0.019 | -0.000 | -0.001 |
    |  322 |  555 |  906 |  908 |  907 |     0.051 |     0.094 |     0.132 | -0.003 | -0.005 |
    |  323 |  867 |  868 |  870 |  869 |     0.113 |     0.299 |     0.423 | -0.006 | -0.015 |
    |  324 | 1040 | 1041 | 1044 | 1042 |     0.290 |    -0.099 |     0.140 | -0.014 |  0.005 |
    |  325 | 1023 |  281 | 1025 | 1024 |     0.067 |    -0.013 |     0.018 | -0.003 |  0.001 |
    |  326 | 1081 | 1023 | 1024 | 1026 |     0.059 |    -0.013 |     0.018 | -0.003 |  0.001 |
    |  327 |  299 |   12 |   13 | 1068 |     0.010 |    -0.001 |     0.001 | -0.000 |  0.000 |
    |  328 | 1392 |  807 |  683 |  718 |     0.004 |    -0.022 |     0.030 | -0.000 |  0.001 |
    |  329 | 1031 |   25 |   26 | 1078 |     0.099 |    -0.008 |     0.011 | -0.005 |  0.000 |
    |  330 |  283 | 1081 | 1026 | 1029 |     0.049 |    -0.009 |     0.013 | -0.002 |  0.000 |
    |  331 |  922 |  923 |  924 |  921 |     0.016 |    -0.046 |     0.065 | -0.001 |  0.002 |
    |  332 |  268 | 1022 |   43 |   44 |     1.721 |    -0.135 |     0.191 | -0.086 |  0.007 |
    |  333 |  494 |  496 |  906 |  555 |     0.066 |     0.090 |     0.127 | -0.003 | -0.004 |
    |  334 |  273 |  980 |  982 |  981 |     0.422 |     0.084 |     0.119 | -0.021 | -0.004 |
    |  335 | 1000 |  994 |  998 |  999 |     0.026 |     0.014 |     0.020 | -0.001 | -0.001 |
    |  336 |  409 |  407 |  920 |  926 |     0.020 |    -0.033 |     0.046 | -0.001 |  0.002 |
    |  337 |  991 |  286 |  993 |  992 |     0.028 |     0.007 |     0.009 | -0.001 | -0.000 |
    |  338 |  413 |  411 |  591 |  894 |     0.013 |    -0.025 |     0.035 | -0.001 |  0.001 |
    |  339 | 1049 | 1040 | 1042 | 1043 |     0.253 |    -0.092 |     0.131 | -0.013 |  0.005 |
    |  340 | 1035 | 1032 | 1031 | 1078 |     0.097 |    -0.019 |     0.027 | -0.005 |  0.001 |
    |  341 |  393 |  550 |  852 |  395 |     0.069 |    -0.084 |     0.119 | -0.003 |  0.004 |
    |  342 |  682 |  614 |  415 |  613 |     0.009 |    -0.021 |     0.030 | -0.000 |  0.001 |
    |  343 |  981 |  982 |  984 |  983 |     0.387 |     0.140 |     0.198 | -0.019 | -0.007 |
    |  344 | 1025 | 1036 | 1027 | 1024 |     0.065 |    -0.022 |     0.032 | -0.003 |  0.001 |
    |  345 | 1024 | 1027 | 1028 | 1026 |     0.054 |    -0.018 |     0.026 | -0.003 |  0.001 |
    |  346 |  285 | 1051 | 1053 | 1052 |     0.030 |    -0.007 |     0.010 | -0.002 |  0.000 |
    |  347 |  992 |  993 |  996 |  994 |     0.025 |     0.010 |     0.013 | -0.001 | -0.000 |
    |  348 |  471 |  473 |  866 |  581 |     0.253 |     0.339 |     0.479 | -0.013 | -0.017 |
    |  349 |  286 | 1083 |  997 |  993 |     0.024 |     0.005 |     0.007 | -0.001 | -0.000 |
    |  350 |  935 |  462 |  465 |  590 |     0.414 |     0.690 |     0.975 | -0.021 | -0.034 |
    |  351 |  984 |  987 |  989 |  988 |     0.297 |     0.150 |     0.213 | -0.015 | -0.008 |
    |  352 |  379 |  377 |  582 |  879 |     0.225 |    -0.301 |     0.426 | -0.011 |  0.015 |
    |  353 |  852 |  550 |  854 |  853 |     0.057 |    -0.101 |     0.143 | -0.003 |  0.005 |
    |  354 |  926 |  896 |  895 |  591 |     0.012 |    -0.031 |     0.044 | -0.001 |  0.002 |
    |  355 |  881 |  880 |  883 |  882 |     0.120 |    -0.302 |     0.427 | -0.006 |  0.015 |
    |  356 |  853 |  854 |  855 |  856 |     0.041 |    -0.110 |     0.156 | -0.002 |  0.006 |
    |  357 |  893 |  579 |  524 |  421 |     0.007 |    -0.014 |     0.019 | -0.000 |  0.001 |
    |  358 |  498 |  500 |  586 |  916 |     0.046 |     0.064 |     0.090 | -0.002 | -0.003 |
    |  359 |  509 |  511 |  544 |  899 |     0.018 |     0.027 |     0.038 | -0.001 | -0.001 |
    |  360 |  872 |  874 |  875 |  873 |     0.159 |     0.447 |     0.632 | -0.008 | -0.022 |
    |  361 |  821 |  306 |  828 |  827 |     0.002 |    -0.012 |     0.016 | -0.000 |  0.001 |
    |  362 |  821 |  228 |  229 |  306 |     0.000 |    -0.011 |     0.016 | -0.000 |  0.001 |
    |  363 |  825 |  307 |  113 |  114 |     0.000 |     0.011 |     0.016 | -0.000 | -0.001 |
    |  364 |  590 |  933 |  936 |  935 |     0.350 |     0.803 |     1.135 | -0.017 | -0.040 |
    |  365 |  872 |  873 | 1090 |  581 |     0.190 |     0.421 |     0.596 | -0.010 | -0.021 |
    |  366 |  691 |  627 |  371 |  625 |     0.240 |    -0.544 |     0.769 | -0.012 |  0.027 |
    |  367 |  907 |  909 | 1093 |  555 |     0.058 |     0.109 |     0.154 | -0.003 | -0.005 |
    |  368 |  594 |  948 |  950 |  949 |     0.404 |    -0.867 |     1.225 | -0.020 |  0.043 |
    |  369 |  272 | 1060 |   37 |   38 |     0.667 |    -0.058 |     0.082 | -0.033 |  0.003 |
    |  370 |  467 |  931 |  590 |  465 |     0.325 |     0.643 |     0.909 | -0.016 | -0.032 |
    |  371 |  273 | 1004 |   79 |   80 |     0.544 |     0.031 |     0.044 | -0.027 | -0.002 |
    |  372 |  609 |  821 |  827 |  822 |     0.001 |    -0.012 |     0.017 | -0.000 |  0.001 |
    |  373 |  371 |  369 |  954 |  625 |     0.322 |    -0.583 |     0.825 | -0.016 |  0.029 |
    |  374 |  609 |  227 |  228 |  821 |     0.001 |    -0.012 |     0.017 | -0.000 |  0.001 |
    |  375 |  608 |  825 |  114 |  115 |     0.001 |     0.012 |     0.017 | -0.000 | -0.001 |
    |  376 |  274 | 1058 |   34 |   35 |     0.417 |    -0.023 |     0.032 | -0.021 |  0.001 |
    |  377 |  278 | 1006 |   85 |   86 |     0.214 |     0.012 |     0.017 | -0.011 | -0.001 |
    |  378 |  550 | 1091 |  858 |  854 |     0.068 |    -0.115 |     0.163 | -0.003 |  0.006 |
    |  379 |  275 | 1039 |   31 |   32 |     0.260 |    -0.022 |     0.031 | -0.013 |  0.001 |
    |  380 |  279 |  959 |   88 |   89 |     0.134 |     0.014 |     0.020 | -0.007 | -0.001 |
    |  381 | 1005 |  271 |   77 |   78 |     0.745 |     0.043 |     0.062 | -0.037 | -0.002 |
    |  382 |  980 |  273 |   80 |   81 |     0.471 |     0.036 |     0.050 | -0.024 | -0.002 |
    |  383 |  286 |  991 |   97 |   98 |     0.031 |     0.002 |     0.003 | -0.002 | -0.000 |
    |  384 |  270 | 1009 |   40 |   41 |     1.074 |    -0.060 |     0.085 | -0.054 |  0.003 |
    |  385 | 1007 |  276 |   83 |   84 |     0.290 |     0.016 |     0.023 | -0.015 | -0.001 |
    |  386 | 1153 |  268 |   44 |   45 |     2.021 |    -0.117 |     0.166 | -0.101 |  0.006 |
    |  387 | 1023 |   23 |   24 |  281 |     0.072 |    -0.007 |     0.009 | -0.004 |  0.000 |
    |  388 | 1078 |   26 |   27 |  280 |     0.114 |    -0.006 |     0.008 | -0.006 |  0.000 |
    |  389 | 1065 |   14 |   15 |  287 |     0.015 |    -0.001 |     0.001 | -0.001 |  0.000 |
    |  390 | 1064 |   17 |   18 |  285 |     0.027 |    -0.003 |     0.004 | -0.001 |  0.000 |
    |  391 | 1094 |   20 |   21 |  283 |     0.044 |    -0.002 |     0.003 | -0.002 |  0.000 |
    |  392 | 1038 |  277 |   29 |   30 |     0.191 |    -0.016 |     0.023 | -0.010 |  0.001 |
    |  393 | 1001 |  284 |   95 |   96 |     0.043 |     0.003 |     0.004 | -0.002 | -0.000 |
    |  394 | 1070 |  282 |   92 |   93 |     0.070 |     0.005 |     0.007 | -0.004 | -0.000 |
    |  395 | 1061 |  274 |   35 |   36 |     0.488 |    -0.043 |     0.061 | -0.024 |  0.002 |
    |  396 |  958 |  278 |   86 |   87 |     0.185 |     0.014 |     0.020 | -0.009 | -0.001 |
    |  397 |  977 |  269 |   74 |   75 |     1.199 |     0.087 |     0.123 | -0.060 | -0.004 |
    |  398 | 1021 |  272 |   38 |   39 |     0.781 |    -0.043 |     0.060 | -0.039 |  0.002 |
    |  399 | 1063 |   16 |   17 | 1064 |     0.022 |    -0.001 |     0.002 | -0.001 |  0.000 |
    |  400 |  288 | 1003 |  100 |  101 |     0.018 |     0.002 |     0.002 | -0.001 | -0.000 |
    |  401 |  951 |  692 |  625 |  954 |     0.279 |    -0.672 |     0.951 | -0.014 |  0.034 |
    |  402 |  949 |  952 |  953 |  951 |     0.250 |    -0.792 |     1.120 | -0.012 |  0.040 |
    |  403 |   89 |   90 |  961 |  279 |     0.113 |     0.009 |     0.013 | -0.006 | -0.000 |
    |  404 |  490 | 1093 |  909 |  913 |     0.073 |     0.126 |     0.178 | -0.004 | -0.006 |
    |  405 |  845 |  848 |  849 |  847 |     0.020 |     0.054 |     0.077 | -0.001 | -0.003 |
    |  406 |  831 |  832 |  834 |  561 |     0.008 |     0.020 |     0.028 | -0.000 | -0.001 |
    |  407 |  978 |  977 |   75 |   76 |     1.017 |     0.060 |     0.085 | -0.051 | -0.003 |
    |  408 |  948 |  594 |  367 |  355 |     0.481 |    -0.771 |     1.091 | -0.024 |  0.039 |
    |  409 |  479 | 1071 |  559 |  477 |     0.165 |     0.241 |     0.341 | -0.008 | -0.012 |
    |  410 |  405 |  403 |  925 |  571 |     0.029 |    -0.042 |     0.060 | -0.001 |  0.002 |
    |  411 |  955 |  456 |  458 |  563 |     0.781 |     1.032 |     1.460 | -0.039 | -0.052 |
    |  412 |  397 |  554 | 1075 |  399 |     0.045 |    -0.061 |     0.086 | -0.002 |  0.003 |
    |  413 |  886 |  884 |  881 |  885 |     0.126 |    -0.233 |     0.329 | -0.006 |  0.012 |
    |  414 |  579 |  619 |  620 |  580 |     0.004 |    -0.014 |     0.019 | -0.000 |  0.001 |
    |  415 | 1095 |  276 | 1105 |  986 |     0.310 |     0.061 |     0.086 | -0.016 | -0.003 |
    |  416 | 1075 |  401 |  400 |  399 |     0.045 |    -0.052 |     0.074 | -0.002 |  0.003 |
    |  417 |  307 |  956 |  112 |  113 |     0.001 |     0.010 |     0.014 | -0.000 | -0.000 |
    |  418 | 1060 | 1061 |   36 |   37 |     0.570 |    -0.031 |     0.044 | -0.029 |  0.002 |
    |  419 | 1039 | 1038 |   30 |   31 |     0.223 |    -0.011 |     0.015 | -0.011 |  0.001 |
    |  420 |  959 |  958 |   87 |   88 |     0.157 |     0.010 |     0.013 | -0.008 | -0.000 |
    |  421 | 1006 | 1007 |   84 |   85 |     0.252 |     0.019 |     0.026 | -0.013 | -0.001 |
    |  422 |  285 |   18 |   19 | 1051 |     0.031 |    -0.002 |     0.002 | -0.002 |  0.000 |
    |  423 |  559 |  475 |  476 |  477 |     0.193 |     0.243 |     0.344 | -0.010 | -0.012 |
    |  424 |  287 |   15 |   16 | 1063 |     0.019 |    -0.002 |     0.003 | -0.001 |  0.000 |
    |  425 |  954 |  594 |  949 |  951 |     0.320 |    -0.735 |     1.039 | -0.016 |  0.037 |
    |  426 |  613 |  415 |  414 |  413 |     0.012 |    -0.021 |     0.029 | -0.001 |  0.001 |
    |  427 |  866 |  473 |  474 |  475 |     0.239 |     0.296 |     0.419 | -0.012 | -0.015 |
    |  428 |  884 |  383 |  382 |  381 |     0.162 |    -0.220 |     0.312 | -0.008 |  0.011 |
    |  429 |  284 | 1001 |  995 | 1002 |     0.040 |     0.009 |     0.013 | -0.002 | -0.000 |
    |  430 |   25 | 1031 |  281 |   24 |     0.085 |    -0.004 |     0.006 | -0.004 |  0.000 |
    |  431 |   96 |   97 |  991 | 1001 |     0.037 |     0.003 |     0.004 | -0.002 | -0.000 |
    |  432 | 1073 |  950 |  948 | 1072 |     0.461 |    -0.983 |     1.390 | -0.023 |  0.049 |
    |  433 |  938 |  937 |  935 |  936 |     0.423 |     0.907 |     1.282 | -0.021 | -0.045 |
    |  434 |  619 |  579 |  893 |  626 |     0.006 |    -0.015 |     0.022 | -0.000 |  0.001 |
    |  435 |  905 |  902 |  693 |  903 |     0.159 |    -0.421 |     0.595 | -0.008 |  0.021 |
    |  436 | 1022 | 1077 |   42 |   43 |     1.472 |    -0.083 |     0.117 | -0.074 |  0.004 |
    |  437 |   39 |   40 | 1009 | 1021 |     0.914 |    -0.076 |     0.108 | -0.046 |  0.004 |
    |  438 |   32 |   33 | 1076 |  275 |     0.304 |    -0.016 |     0.022 | -0.015 |  0.001 |
    |  439 |  544 |  836 |  835 |  837 |     0.012 |     0.026 |     0.037 | -0.001 | -0.001 |
    |  440 |   71 |   72 |  968 |  267 |     1.937 |     0.129 |     0.182 | -0.097 | -0.006 |
    |  441 | 1058 | 1076 |   33 |   34 |     0.356 |    -0.031 |     0.044 | -0.018 |  0.002 |
    |  442 |  956 |  292 |  111 |  112 |     0.000 |     0.009 |     0.012 | -0.000 | -0.000 |
    |  443 |  892 |  293 |  230 |  231 |     0.001 |    -0.008 |     0.011 | -0.000 |  0.000 |
    |  444 |   14 | 1065 | 1068 |   13 |     0.012 |    -0.002 |     0.002 | -0.001 |  0.000 |
    |  445 |   23 | 1023 | 1081 |   22 |     0.060 |    -0.003 |     0.004 | -0.003 |  0.000 |
    |  446 |   41 |   42 | 1077 |  270 |     1.256 |    -0.100 |     0.142 | -0.063 |  0.005 |
    |  447 |   98 |   99 | 1083 |  286 |     0.026 |     0.002 |     0.003 | -0.001 | -0.000 |
    |  448 | 1084 |  298 |  104 |  105 |     0.007 |     0.001 |     0.002 | -0.000 | -0.000 |
    |  449 |  283 |   21 |   22 | 1081 |     0.052 |    -0.005 |     0.006 | -0.003 |  0.000 |
    |  450 |  862 |  865 | 1306 |  863 |     0.017 |    -0.075 |     0.106 | -0.001 |  0.004 |
    |  451 |   12 |  299 | 1082 |   11 |     0.007 |    -0.001 |     0.002 | -0.000 |  0.000 |
    |  452 |  649 | 1084 |  105 |  106 |     0.005 |     0.001 |     0.001 | -0.000 | -0.000 |
    |  453 |  292 | 1080 |  110 |  111 |     0.001 |     0.007 |     0.010 | -0.000 | -0.000 |
    |  454 | 1080 |  534 |  109 |  110 |     0.001 |     0.005 |     0.007 | -0.000 | -0.000 |
    |  455 |  648 |   10 |   11 | 1082 |     0.005 |    -0.001 |     0.001 | -0.000 |  0.000 |
    |  456 |  381 |  548 |  881 |  884 |     0.155 |    -0.245 |     0.347 | -0.008 |  0.012 |
    |  457 | 1003 | 1083 |   99 |  100 |     0.022 |     0.001 |     0.002 | -0.001 | -0.000 |
    |  458 |  468 |  471 |  581 | 1090 |     0.277 |     0.402 |     0.569 | -0.014 | -0.020 |
    |  459 |  787 |  289 |  157 |  158 |     0.217 |     4.400 |     6.222 | -0.011 | -0.220 |
    |  460 |  769 |  184 |  185 |  290 |     0.265 |    -4.364 |     6.172 | -0.013 |  0.218 |
    |  461 | 1017 | 1014 | 1011 | 1015 |     1.417 |    -0.480 |     0.678 | -0.071 |  0.024 |
    |  462 |  390 | 1091 |  550 |  393 |     0.088 |    -0.107 |     0.151 | -0.004 |  0.005 |
    |  463 |  843 |  842 |  838 |  840 |     0.014 |     0.037 |     0.053 | -0.001 | -0.002 |
    |  464 |  962 |  963 |  279 |  961 |     0.102 |     0.026 |     0.037 | -0.005 | -0.001 |
    |  465 |  491 |  494 |  555 | 1093 |     0.080 |     0.105 |     0.148 | -0.004 | -0.005 |
    |  466 | 1032 | 1025 |  281 | 1031 |     0.082 |    -0.020 |     0.028 | -0.004 |  0.001 |
    |  467 | 1047 | 1044 | 1041 | 1045 |     0.343 |    -0.130 |     0.183 | -0.017 |  0.006 |
    |  468 |  992 |  995 | 1001 |  991 |     0.033 |     0.007 |     0.010 | -0.002 | -0.000 |
    |  469 |  804 |  805 |  812 |  806 |     0.389 |    -2.003 |     2.833 | -0.019 |  0.100 |
    |  470 |  566 |  665 |  666 |  645 |     0.013 |     0.009 |     0.012 | -0.001 | -0.000 |
    |  471 |  691 |  693 |  902 |  627 |     0.172 |    -0.481 |     0.680 | -0.009 |  0.024 |
    |  472 | 1054 | 1053 | 1056 | 1055 |     0.033 |    -0.013 |     0.018 | -0.002 |  0.001 |
    |  473 |  299 | 1068 | 1067 | 1069 |     0.009 |    -0.003 |     0.005 | -0.000 |  0.000 |
    |  474 | 1030 | 1029 | 1026 | 1028 |     0.047 |    -0.017 |     0.024 | -0.002 |  0.001 |
    |  475 | 1336 | 1034 | 1032 | 1035 |     0.093 |    -0.039 |     0.055 | -0.005 |  0.002 |
    |  476 |   20 | 1094 | 1051 |   19 |     0.038 |    -0.003 |     0.005 | -0.002 |  0.000 |
    |  477 |  514 |  511 |  512 |  513 |     0.017 |     0.021 |     0.029 | -0.001 | -0.001 |
    |  478 |  509 |  510 |  512 |  511 |     0.021 |     0.025 |     0.035 | -0.001 | -0.001 |
    |  479 |  502 |  503 |  504 |  851 |     0.035 |     0.045 |     0.063 | -0.002 | -0.002 |
    |  480 |  500 |  501 |  503 |  502 |     0.042 |     0.049 |     0.070 | -0.002 | -0.002 |
    |  481 |  498 |  499 |  501 |  500 |     0.053 |     0.057 |     0.081 | -0.003 | -0.003 |
    |  482 |  507 |  508 |  510 |  509 |     0.026 |     0.028 |     0.039 | -0.001 | -0.001 |
    |  483 |  498 |  496 |  495 |  497 |     0.071 |     0.070 |     0.099 | -0.004 | -0.004 |
    |  484 | 1090 |  873 |  875 |  877 |     0.207 |     0.469 |     0.663 | -0.010 | -0.023 |
    |  485 | 1093 |  490 |  489 |  491 |     0.090 |     0.121 |     0.171 | -0.004 | -0.006 |
    |  486 |  987 |  984 |  982 |  986 |     0.324 |     0.112 |     0.159 | -0.016 | -0.006 |
    |  487 |  490 |  488 |  482 |  489 |     0.112 |     0.146 |     0.207 | -0.006 | -0.007 |
    |  488 |  864 |  922 |  571 |  925 |     0.026 |    -0.050 |     0.070 | -0.001 |  0.002 |
    |  489 |  488 |  481 |  480 |  482 |     0.132 |     0.169 |     0.239 | -0.007 | -0.008 |
    |  490 |  409 |  411 |  412 |  410 |     0.019 |    -0.025 |     0.035 | -0.001 |  0.001 |
    |  491 |  407 |  409 |  410 |  408 |     0.023 |    -0.029 |     0.041 | -0.001 |  0.001 |
    |  492 |  405 |  407 |  408 |  406 |     0.027 |    -0.032 |     0.045 | -0.001 |  0.002 |
    |  493 |  403 |  405 |  406 |  404 |     0.033 |    -0.038 |     0.054 | -0.002 |  0.002 |
    |  494 |  401 |  403 |  404 |  402 |     0.039 |    -0.039 |     0.055 | -0.002 |  0.002 |
    |  495 |  397 |  399 |  400 |  398 |     0.053 |    -0.058 |     0.082 | -0.003 |  0.003 |
    |  496 |  395 |  397 |  398 |  396 |     0.066 |    -0.059 |     0.083 | -0.003 |  0.003 |
    |  497 |  481 |  479 |  478 |  480 |     0.167 |     0.201 |     0.285 | -0.008 | -0.010 |
    |  498 | 1013 | 1019 | 1020 | 1018 |     0.874 |    -0.319 |     0.451 | -0.044 |  0.016 |
    |  499 |  504 |  506 |  508 |  507 |     0.033 |     0.033 |     0.047 | -0.002 | -0.002 |
    |  500 |   81 |   82 | 1095 |  980 |     0.397 |     0.022 |     0.031 | -0.020 | -0.001 |
    |  501 | 1091 |  390 |  388 |  389 |     0.099 |    -0.120 |     0.170 | -0.005 |  0.006 |
    |  502 |  496 |  494 |  493 |  495 |     0.085 |     0.085 |     0.120 | -0.004 | -0.004 |
    |  503 |  389 |  388 |  386 |  387 |     0.122 |    -0.144 |     0.204 | -0.006 |  0.007 |
    |  504 |  479 |  477 |  476 |  478 |     0.180 |     0.221 |     0.312 | -0.009 | -0.011 |
    |  505 |  387 |  386 |  384 |  385 |     0.139 |    -0.159 |     0.225 | -0.007 |  0.008 |
    |  506 |  516 |  514 |  513 |  515 |     0.014 |     0.019 |     0.027 | -0.001 | -0.001 |
    |  507 |  385 |  384 |  382 |  383 |     0.166 |    -0.191 |     0.270 | -0.008 |  0.010 |
    |  508 |  859 |  855 |  854 |  858 |     0.057 |    -0.133 |     0.189 | -0.003 |  0.007 |
    |  509 |  411 |  413 |  414 |  412 |     0.015 |    -0.024 |     0.034 | -0.001 |  0.001 |
    |  510 |  395 |  394 |  392 |  393 |     0.090 |    -0.078 |     0.110 | -0.004 |  0.004 |
    |  511 |  379 |  381 |  382 |  380 |     0.206 |    -0.228 |     0.323 | -0.010 |  0.011 |
    |  512 |  471 |  472 |  474 |  473 |     0.283 |     0.311 |     0.440 | -0.014 | -0.016 |
    |  513 |  910 |  911 |  913 |  909 |     0.050 |     0.128 |     0.181 | -0.003 | -0.006 |
    |  514 |  682 |  683 |  819 |  614 |     0.006 |    -0.019 |     0.028 | -0.000 |  0.001 |
    |  515 |  518 |  516 |  515 |  517 |     0.011 |     0.016 |     0.023 | -0.001 | -0.001 |
    |  516 |  379 |  378 |  376 |  377 |     0.284 |    -0.275 |     0.390 | -0.014 |  0.014 |
    |  517 | 1020 | 1021 | 1009 | 1018 |     0.902 |    -0.175 |     0.248 | -0.045 |  0.009 |
    |  518 |  576 |  611 |  662 |  661 |     0.583 |    -0.304 |     0.429 | -0.029 |  0.015 |
    |  519 |  832 |  835 |  836 |  834 |     0.010 |     0.023 |     0.033 | -0.001 | -0.001 |
    |  520 |  468 |  470 |  472 |  471 |     0.334 |     0.356 |     0.503 | -0.017 | -0.018 |
    |  521 |  569 |  663 |  664 |  618 |     0.639 |     0.330 |     0.467 | -0.032 | -0.017 |
    |  522 |  465 |  466 |  468 |  467 |     0.369 |     0.512 |     0.724 | -0.018 | -0.026 |
    |  523 |  377 |  376 |  374 |  375 |     0.319 |    -0.327 |     0.462 | -0.016 |  0.016 |
    |  524 |  584 |  616 |  660 |  659 |     0.021 |    -0.012 |     0.017 | -0.001 |  0.001 |
    |  525 |  972 | 1293 |  969 |  971 |     1.350 |     0.531 |     0.751 | -0.068 | -0.027 |
    |  526 | 1066 | 1067 | 1068 | 1065 |     0.011 |    -0.003 |     0.005 | -0.001 |  0.000 |
    |  527 |  642 |  629 |  556 |  628 |     2.346 |     1.220 |     1.726 | -0.117 | -0.061 |
    |  528 |  573 |  646 |  656 |  655 |     0.154 |    -0.080 |     0.113 | -0.008 |  0.004 |
    |  529 |  369 |  371 |  372 |  370 |     0.389 |    -0.524 |     0.741 | -0.019 |  0.026 |
    |  530 |  552 |  657 |  658 |  640 |     0.162 |     0.086 |     0.122 | -0.008 | -0.004 |
    |  531 |  589 |  653 |  654 |  641 |     0.060 |     0.031 |     0.044 | -0.003 | -0.002 |
    |  532 |  421 |  420 |  418 |  419 |     0.010 |    -0.014 |     0.019 | -0.000 |  0.001 |
    |  533 |  375 |  372 |  371 | 1092 |     0.311 |    -0.431 |     0.610 | -0.016 |  0.022 |
    |  534 |  462 |  463 |  466 |  465 |     0.472 |     0.602 |     0.851 | -0.024 | -0.030 |
    |  535 |  419 |  416 |  415 | 1096 |     0.009 |    -0.017 |     0.024 | -0.000 |  0.001 |
    |  536 |  517 | 1118 |  522 |  519 |     0.010 |     0.013 |     0.018 | -0.001 | -0.001 |
    |  537 | 1322 |  249 |  208 |  209 |     0.004 |    -0.097 |     0.138 | -0.000 |  0.005 |
    |  538 |  944 |  261 |  151 |  152 |     0.081 |     1.688 |     2.388 | -0.004 | -0.084 |
    |  539 | 1088 |  263 |  154 |  155 |     0.262 |     2.723 |     3.852 | -0.013 | -0.136 |
    |  540 |  901 |  241 |  220 |  221 |     0.000 |    -0.018 |     0.025 | -0.000 |  0.001 |
    |  541 |  812 |  190 |  191 |  262 |     0.093 |    -1.698 |     2.401 | -0.005 |  0.085 |
    |  542 |  904 |  199 |  200 |  256 |     0.040 |    -0.413 |     0.584 | -0.002 |  0.021 |
    |  543 | 1395 |  136 |  137 |  251 |     0.017 |     0.157 |     0.222 | -0.001 | -0.008 |
    |  544 |  811 |  187 |  188 |  264 |     0.248 |    -2.706 |     3.826 | -0.012 |  0.135 |
    |  545 |  367 |  369 |  370 |  368 |     0.472 |    -0.593 |     0.839 | -0.024 |  0.030 |
    |  546 |  371 |  627 |  902 | 1092 |     0.263 |    -0.465 |     0.658 | -0.013 |  0.023 |
    |  547 |  460 |  461 |  463 |  462 |     0.561 |     0.667 |     0.943 | -0.028 | -0.033 |
    |  548 |  520 |  518 |  517 |  519 |     0.009 |     0.015 |     0.021 | -0.000 | -0.001 |
    |  549 |  458 |  459 |  461 |  460 |     0.679 |     0.776 |     1.097 | -0.034 | -0.039 |
    |  550 |  494 |  491 |  492 |  493 |     0.096 |     0.097 |     0.138 | -0.005 | -0.005 |
    |  551 |  355 |  367 |  368 |  356 |     0.578 |    -0.697 |     0.986 | -0.029 |  0.035 |
    |  552 |  393 |  392 |  391 |  390 |     0.099 |    -0.092 |     0.130 | -0.005 |  0.005 |
    |  553 |  229 |  890 |  828 |  306 |     0.001 |    -0.010 |     0.014 | -0.000 |  0.001 |
    |  554 |  801 | 1017 | 1015 | 1141 |     1.671 |    -0.584 |     0.826 | -0.084 |  0.029 |
    |  555 |  415 |  614 |  819 | 1096 |     0.009 |    -0.018 |     0.026 | -0.000 |  0.001 |
    |  556 | 1079 |  543 |  450 |  452 |     1.163 |     1.512 |     2.138 | -0.058 | -0.076 |
    |  557 |  524 |  422 |  420 |  421 |     0.009 |    -0.013 |     0.018 | -0.000 |  0.001 |
    |  558 |  645 |  597 |  565 |  566 |     0.011 |     0.008 |     0.012 | -0.001 | -0.000 |
    |  559 |  606 |  526 |  631 |  607 |     0.004 |     0.013 |     0.019 | -0.000 | -0.001 |
    |  560 |  419 |  418 |  417 |  416 |     0.013 |    -0.016 |     0.022 | -0.001 |  0.001 |
    |  561 |  375 |  374 |  373 |  372 |     0.396 |    -0.392 |     0.555 | -0.020 |  0.020 |
    |  562 |  353 |  355 |  356 |  354 |     0.666 |    -0.786 |     1.111 | -0.033 |  0.039 |
    |  563 |  685 |  595 |  567 |  617 |     0.005 |     0.009 |     0.012 | -0.000 | -0.000 |
    |  564 |  506 |  504 |  503 |  505 |     0.039 |     0.038 |     0.053 | -0.002 | -0.002 |
    |  565 |  498 |  497 |  532 |  499 |     0.068 |     0.057 |     0.081 | -0.003 | -0.003 |
    |  566 | 1051 | 1094 | 1056 | 1053 |     0.035 |    -0.007 |     0.010 | -0.002 |  0.000 |
    |  567 |  526 |  520 |  519 |  521 |     0.007 |     0.013 |     0.019 | -0.000 | -0.001 |
    |  568 |  986 | 1105 |  990 |  987 |     0.285 |     0.100 |     0.141 | -0.014 | -0.005 |
    |  569 |  980 | 1095 |  986 |  982 |     0.368 |     0.078 |     0.110 | -0.018 | -0.004 |
    |  570 |  415 |  416 |  417 |  414 |     0.013 |    -0.019 |     0.026 | -0.001 |  0.001 |
    |  571 |  969 |  967 |  267 |  968 |     1.794 |     0.430 |     0.608 | -0.090 | -0.021 |
    |  572 |  595 |  527 |  565 |  567 |     0.007 |     0.009 |     0.013 | -0.000 | -0.000 |
    |  573 |  470 |  468 |  466 |  469 |     0.422 |     0.438 |     0.620 | -0.021 | -0.022 |
    |  574 |  974 |  972 |  971 |  973 |     1.131 |     0.405 |     0.573 | -0.057 | -0.020 |
    |  575 |  638 |  639 | 1106 |  588 |     0.044 |     0.024 |     0.033 | -0.002 | -0.001 |
    |  576 |  566 |  565 |  527 |  522 |     0.009 |     0.010 |     0.014 | -0.000 | -0.000 |
    |  577 |  589 |  588 |  506 |  505 |     0.046 |     0.033 |     0.047 | -0.002 | -0.002 |
    |  578 |  579 |  580 |  593 |  524 |     0.006 |    -0.013 |     0.018 | -0.000 |  0.001 |
    |  579 |  552 |  551 |  493 |  492 |     0.120 |     0.090 |     0.128 | -0.006 | -0.005 |
    |  580 |  573 |  391 |  392 |  572 |     0.120 |    -0.081 |     0.115 | -0.006 |  0.004 |
    |  581 |  584 |  417 |  418 |  583 |     0.014 |    -0.014 |     0.019 | -0.001 |  0.001 |
    |  582 |  616 |  584 |  583 |  612 |     0.018 |    -0.012 |     0.017 | -0.001 |  0.001 |
    |  583 |  203 |  254 | 1099 |  202 |     0.014 |    -0.260 |     0.368 | -0.001 |  0.013 |
    |  584 |  138 |  139 |  253 | 1098 |     0.015 |     0.222 |     0.314 | -0.001 | -0.011 |
    |  585 |  128 |  246 | 1097 |  127 |     0.002 |     0.038 |     0.054 | -0.000 | -0.002 |
    |  586 |  641 |  638 |  588 |  589 |     0.051 |     0.026 |     0.037 | -0.003 | -0.001 |
    |  587 |  646 |  573 |  572 |  599 |     0.132 |    -0.064 |     0.090 | -0.007 |  0.003 |
    |  588 |  640 |  592 |  551 |  552 |     0.137 |     0.073 |     0.104 | -0.007 | -0.004 |
    |  589 |  611 |  576 |  575 |  610 |     0.506 |    -0.278 |     0.393 | -0.025 |  0.014 |
    |  590 |  618 |  601 |  568 |  569 |     0.549 |     0.296 |     0.418 | -0.027 | -0.015 |
    |  591 |  569 |  568 |  470 |  469 |     0.475 |     0.351 |     0.496 | -0.024 | -0.018 |
    |  592 |  268 | 1141 | 1015 | 1022 |     1.712 |    -0.327 |     0.462 | -0.086 |  0.016 |
    |  593 |  456 |  457 |  459 |  458 |     0.798 |     0.812 |     1.148 | -0.040 | -0.041 |
    |  594 |  576 |  373 |  374 |  575 |     0.442 |    -0.326 |     0.461 | -0.022 |  0.016 |
    |  595 |  155 |  156 | 1085 | 1088 |     0.250 |     3.179 |     4.495 | -0.013 | -0.159 |
    |  596 |  519 |  522 |  527 |  521 |     0.008 |     0.012 |     0.017 | -0.000 | -0.001 |
    |  597 |  592 |  596 | 1100 |  551 |     0.122 |     0.066 |     0.094 | -0.006 | -0.003 |
    |  598 |  665 |  566 |  522 | 1118 |     0.012 |     0.011 |     0.016 | -0.001 | -0.001 |
    |  599 |  601 |  633 | 1115 |  568 |     0.463 |     0.241 |     0.340 | -0.023 | -0.012 |
    |  600 |  955 |  454 |  455 |  456 |     0.937 |     1.082 |     1.530 | -0.047 | -0.054 |
    |  601 |  495 |  493 |  551 | 1100 |     0.101 |     0.075 |     0.106 | -0.005 | -0.004 |
    |  602 |  394 | 1101 |  572 |  392 |     0.103 |    -0.066 |     0.093 | -0.005 |  0.003 |
    |  603 |  610 |  575 | 1113 |  634 |     0.424 |    -0.226 |     0.319 | -0.021 |  0.011 |
    |  604 |  395 |  396 | 1111 |  394 |     0.077 |    -0.063 |     0.088 | -0.004 |  0.003 |
    |  605 |  122 |  242 | 1104 |  121 |     0.000 |     0.017 |     0.025 | -0.000 | -0.001 |
    |  606 |  723 |  725 | 1359 |  728 |     2.867 |   -11.057 |    15.638 | -0.143 |  0.553 |
    |  607 |  351 |  353 |  354 |  352 |     0.802 |    -0.925 |     1.309 | -0.040 |  0.046 |
    |  608 |  567 |  565 |  597 | 1102 |     0.008 |     0.007 |     0.011 | -0.000 | -0.000 |
    |  609 |  617 |  567 | 1102 |  637 |     0.006 |     0.007 |     0.010 | -0.000 | -0.000 |
    |  610 |  521 | 1108 |  631 |  526 |     0.006 |     0.012 |     0.018 | -0.000 | -0.001 |
    |  611 |  612 |  583 | 1119 |  635 |     0.014 |    -0.010 |     0.014 | -0.001 |  0.000 |
    |  612 |  202 | 1099 | 1122 |  201 |     0.027 |    -0.302 |     0.427 | -0.001 |  0.015 |
    |  613 |  201 | 1122 |  256 |  200 |     0.020 |    -0.355 |     0.502 | -0.001 |  0.018 |
    |  614 |  673 |  686 | 1102 |  597 |     0.010 |     0.006 |     0.008 | -0.000 | -0.000 |
    |  615 |  637 | 1102 |  686 | 1125 |     0.007 |     0.005 |     0.007 | -0.000 | -0.000 |
    |  616 |  508 |  506 |  588 | 1106 |     0.038 |     0.028 |     0.040 | -0.002 | -0.001 |
    |  617 |  607 |  631 |  675 |  632 |     0.004 |     0.013 |     0.018 | -0.000 | -0.001 |
    |  618 |  661 | 1116 |  373 |  576 |     0.534 |    -0.392 |     0.555 | -0.027 |  0.020 |
    |  619 |  663 |  569 |  469 | 1114 |     0.577 |     0.424 |     0.599 | -0.029 | -0.021 |
    |  620 | 1137 |  663 | 1114 | 1124 |     0.679 |     0.470 |     0.665 | -0.034 | -0.024 |
    |  621 |  420 | 1119 |  583 |  418 |     0.012 |    -0.012 |     0.017 | -0.001 |  0.001 |
    |  622 |  472 |  470 |  568 | 1115 |     0.409 |     0.302 |     0.427 | -0.020 | -0.015 |
    |  623 |  659 | 1117 |  417 |  584 |     0.019 |    -0.017 |     0.023 | -0.001 |  0.001 |
    |  624 |  796 |  797 |  798 |  795 |     2.693 |    -0.959 |     1.356 | -0.135 |  0.048 |
    |  625 |  376 | 1113 |  575 |  374 |     0.386 |    -0.289 |     0.408 | -0.019 |  0.014 |
    |  626 |  655 | 1112 |  391 |  573 |     0.136 |    -0.092 |     0.131 | -0.007 |  0.005 |
    |  627 |  510 |  508 | 1106 |  531 |     0.032 |     0.024 |     0.035 | -0.002 | -0.001 |
    |  628 |  653 |  589 |  505 | 1109 |     0.053 |     0.036 |     0.051 | -0.003 | -0.002 |
    |  629 |  657 |  552 |  492 | 1110 |     0.138 |     0.100 |     0.142 | -0.007 | -0.005 |
    |  630 |  497 |  495 | 1100 | 1107 |     0.092 |     0.064 |     0.090 | -0.005 | -0.003 |
    |  631 |  501 | 1109 |  505 |  503 |     0.049 |     0.045 |     0.063 | -0.002 | -0.002 |
    |  632 |  491 |  489 | 1110 |  492 |     0.117 |     0.116 |     0.164 | -0.006 | -0.006 |
    |  633 |  422 |  528 | 1119 |  420 |     0.010 |    -0.011 |     0.015 | -0.000 |  0.001 |
    |  634 |  390 |  391 | 1112 |  388 |     0.121 |    -0.110 |     0.156 | -0.006 |  0.006 |
    |  635 |  595 | 1108 |  521 |  527 |     0.006 |     0.011 |     0.015 | -0.000 | -0.001 |
    |  636 |  463 | 1114 |  469 |  466 |     0.514 |     0.495 |     0.699 | -0.026 | -0.025 |
    |  637 |  378 |  525 | 1113 |  376 |     0.323 |    -0.236 |     0.334 | -0.016 |  0.012 |
    |  638 |  412 |  414 |  417 | 1117 |     0.016 |    -0.019 |     0.027 | -0.001 |  0.001 |
    |  639 |  372 |  373 | 1116 |  370 |     0.454 |    -0.439 |     0.621 | -0.023 |  0.022 |
    |  640 | 1349 |  486 |  472 | 1115 |     0.345 |     0.253 |     0.358 | -0.017 | -0.013 |
    |  641 |  349 |  351 |  352 |  350 |     0.924 |    -0.966 |     1.367 | -0.046 |  0.048 |
    |  642 | 1138 | 1129 | 1112 |  655 |     0.162 |    -0.111 |     0.157 | -0.008 |  0.006 |
    |  643 |  422 |  524 |  593 | 1121 |     0.006 |    -0.011 |     0.016 | -0.000 |  0.001 |
    |  644 |  515 | 1126 | 1118 |  517 |     0.013 |     0.015 |     0.021 | -0.001 | -0.001 |
    |  645 | 1110 | 1130 | 1139 |  657 |     0.165 |     0.117 |     0.166 | -0.008 | -0.006 |
    |  646 |  121 | 1104 |  309 |  120 |     0.001 |     0.016 |     0.023 | -0.000 | -0.001 |
    |  647 | 1136 |  665 | 1118 | 1126 |     0.014 |     0.012 |     0.017 | -0.001 | -0.001 |
    |  648 |  528 |  422 | 1121 |  529 |     0.008 |    -0.010 |     0.014 | -0.000 |  0.000 |
    |  649 | 1141 |  800 |  799 |  801 |     1.938 |    -0.653 |     0.923 | -0.097 |  0.033 |
    |  650 |  455 |  454 |  452 |  453 |     1.161 |     1.144 |     1.617 | -0.058 | -0.057 |
    |  651 |  408 |  410 | 1128 |  487 |     0.025 |    -0.024 |     0.033 | -0.001 |  0.001 |
    |  652 |  697 |  637 | 1125 |  698 |     0.005 |     0.005 |     0.007 | -0.000 | -0.000 |
    |  653 |  368 | 1123 |  484 |  356 |     0.645 |    -0.594 |     0.840 | -0.032 |  0.030 |
    |  654 |  459 |  485 | 1124 |  461 |     0.723 |     0.632 |     0.893 | -0.036 | -0.032 |
    |  655 |  370 | 1116 | 1123 |  368 |     0.554 |    -0.530 |     0.749 | -0.028 |  0.026 |
    |  656 |  423 |  384 |  386 | 1129 |     0.165 |    -0.147 |     0.208 | -0.008 |  0.007 |
    |  657 |  461 | 1124 | 1114 |  463 |     0.626 |     0.580 |     0.821 | -0.031 | -0.029 |
    |  658 |  483 | 1130 |  482 |  480 |     0.165 |     0.153 |     0.217 | -0.008 | -0.008 |
    |  659 |  513 |  523 | 1126 |  515 |     0.016 |     0.016 |     0.022 | -0.001 | -0.001 |
    |  660 |  533 |  348 |  349 |  350 |     1.019 |    -1.019 |     1.441 | -0.051 |  0.051 |
    |  661 |  388 | 1112 | 1129 |  386 |     0.138 |    -0.124 |     0.175 | -0.007 |  0.006 |
    |  662 |  499 | 1131 | 1109 |  501 |     0.057 |     0.046 |     0.065 | -0.003 | -0.002 |
    |  663 |  489 |  482 | 1130 | 1110 |     0.137 |     0.131 |     0.185 | -0.007 | -0.007 |
    |  664 |  410 |  412 | 1117 | 1128 |     0.021 |    -0.022 |     0.031 | -0.001 |  0.001 |
    |  665 |  120 |  309 | 1103 |  119 |     0.000 |     0.014 |     0.020 | -0.000 | -0.001 |
    |  666 |  635 | 1119 |  528 | 1134 |     0.012 |    -0.009 |     0.013 | -0.001 |  0.000 |
    |  667 |  529 |  600 | 1134 |  528 |     0.009 |    -0.008 |     0.011 | -0.000 |  0.000 |
    |  668 | 1120 |  453 |  452 |  451 |     1.390 |     1.086 |     1.537 | -0.069 | -0.054 |
    |  669 | 1108 | 1135 |  675 |  631 |     0.004 |     0.011 |     0.016 | -0.000 | -0.001 |
    |  670 | 1132 |  447 |  449 |  450 |     1.515 |     1.800 |     2.546 | -0.076 | -0.090 |
    |  671 |  685 | 1135 | 1108 |  595 |     0.005 |     0.010 |     0.015 | -0.000 | -0.001 |
    |  672 | 1344 | 1098 | 1302 |  915 |     0.037 |     0.198 |     0.280 | -0.002 | -0.010 |
    |  673 |  808 |  622 |  667 |  668 |     2.755 |    10.348 |    14.635 | -0.138 | -0.517 |
    |  674 |  797 |  799 |  800 |  798 |     2.312 |    -0.852 |     1.204 | -0.116 |  0.043 |
    |  675 |  452 |  450 |  449 |  451 |     1.436 |     1.445 |     2.044 | -0.072 | -0.072 |
    |  676 |  186 | 1146 |  290 |  185 |     0.317 |    -3.697 |     5.229 | -0.016 |  0.185 |
    |  677 |  628 |  556 |  449 |  448 |     2.096 |     1.513 |     2.140 | -0.105 | -0.076 |
    |  678 |  451 |  449 |  556 | 1127 |     1.793 |     1.297 |     1.834 | -0.090 | -0.065 |
    |  679 |  790 |  777 |  776 |  789 |     0.969 |    -2.651 |     3.749 | -0.048 |  0.133 |
    |  680 |  264 |  789 |  776 |  811 |     0.577 |    -2.710 |     3.833 | -0.029 |  0.136 |
    |  681 |  347 |  346 |  344 |  345 |     1.383 |    -1.410 |     1.994 | -0.069 |  0.070 |
    |  682 | 1086 | 1087 | 1088 | 1085 |     0.835 |     3.347 |     4.733 | -0.042 | -0.167 |
    |  683 |  347 |  348 | 1140 |  346 |     1.277 |    -1.249 |     1.767 | -0.064 |  0.062 |
    |  684 |  449 |  447 |  445 |  448 |     1.863 |     1.957 |     2.767 | -0.093 | -0.098 |
    |  685 |  344 |  346 | 1140 |  560 |     1.473 |    -1.258 |     1.779 | -0.074 |  0.063 |
    |  686 |  345 |  344 |  342 |  343 |     1.489 |    -1.549 |     2.190 | -0.074 |  0.077 |
    |  687 |  977 |  978 | 1162 |  975 |     0.937 |     0.198 |     0.280 | -0.047 | -0.010 |
    |  688 | 1180 |  669 | 1070 | 1177 |     0.055 |     0.012 |     0.017 | -0.003 | -0.001 |
    |  689 | 1384 |  308 | 1281 |  824 |     0.002 |    -0.014 |     0.020 | -0.000 |  0.001 |
    |  690 | 1133 |  546 |  322 |  321 |    10.913 |   -10.738 |    15.186 | -0.546 |  0.537 |
    |  691 |  777 |  775 |  771 |  776 |     1.231 |    -3.043 |     4.303 | -0.062 |  0.152 |
    |  692 |  447 |  446 |  444 |  445 |     1.734 |     2.317 |     3.277 | -0.087 | -0.116 |
    |  693 |  560 | 1143 |  342 |  344 |     1.656 |    -1.424 |     2.014 | -0.083 |  0.071 |
    |  694 |  343 |  342 |  340 |  341 |     1.854 |    -1.922 |     2.718 | -0.093 |  0.096 |
    |  695 |  295 |  174 |  175 |  737 |     0.194 |   -12.505 |    17.685 | -0.010 |  0.625 |
    |  696 |  340 |  342 | 1143 |  549 |     2.051 |    -1.569 |     2.218 | -0.103 |  0.078 |
    |  697 | 1322 |  857 | 1337 |  249 |     0.017 |    -0.095 |     0.134 | -0.001 |  0.005 |
    |  698 |  445 |  444 |  442 |  443 |     2.119 |     2.585 |     3.656 | -0.106 | -0.129 |
    |  699 | 1019 | 1149 |  661 |  662 |     0.696 |    -0.369 |     0.522 | -0.035 |  0.018 |
    |  700 |  998 | 1334 | 1375 |  999 |     0.024 |     0.017 |     0.024 | -0.001 | -0.001 |
    |  701 |  770 |  290 | 1146 |  771 |     0.842 |    -3.857 |     5.454 | -0.042 |  0.193 |
    |  702 |  733 |  818 |  778 | 1368 |     1.929 |    -5.427 |     7.676 | -0.096 |  0.271 |
    |  703 |  712 |  708 |  709 |  710 |     3.898 |     1.989 |     2.812 | -0.195 | -0.099 |
    |  704 |  448 |  445 |  443 | 1144 |     2.337 |     2.101 |     2.972 | -0.117 | -0.105 |
    |  705 |  771 | 1146 |  811 |  776 |     0.825 |    -3.248 |     4.593 | -0.041 |  0.162 |
    |  706 |  340 |  339 |  338 |  541 |     2.442 |    -2.456 |     3.474 | -0.122 |  0.123 |
    |  707 |  446 | 1145 |  577 |  444 |     1.539 |     2.687 |     3.800 | -0.077 | -0.134 |
    |  708 |  443 |  442 |  440 |  441 |     2.513 |     2.734 |     3.866 | -0.126 | -0.137 |
    |  709 |  339 |  337 |  336 |  338 |     3.024 |    -2.932 |     4.147 | -0.151 |  0.147 |
    |  710 |  612 |  647 | 1156 |  616 |     0.019 |    -0.008 |     0.011 | -0.001 |  0.000 |
    |  711 |  785 |  536 |  436 |  438 |     2.996 |     4.187 |     5.921 | -0.150 | -0.209 |
    |  712 | 1078 |  280 | 1166 | 1035 |     0.109 |    -0.026 |     0.036 | -0.005 |  0.001 |
    |  713 | 1328 |  243 | 1357 |  898 |     0.005 |    -0.028 |     0.040 | -0.000 |  0.001 |
    |  714 |  763 |  762 |  300 |  760 |     5.193 |    -1.100 |     1.555 | -0.260 |  0.055 |
    |  715 |  973 |  975 | 1200 |  974 |     0.986 |     0.352 |     0.498 | -0.049 | -0.018 |
    |  716 |  723 |  722 | 1148 |  724 |     3.666 |    -9.077 |    12.837 | -0.183 |  0.454 |
    |  717 |  780 |  781 |  787 |  779 |     0.918 |     5.260 |     7.439 | -0.046 | -0.263 |
    |  718 |  844 |  129 |  130 | 1165 |     0.004 |     0.053 |     0.075 | -0.000 | -0.003 |
    |  719 | 1340 |  263 | 1301 |  946 |     0.488 |     2.340 |     3.309 | -0.024 | -0.117 |
    |  720 |  337 |  335 |  334 |  336 |     3.579 |    -3.268 |     4.622 | -0.179 |  0.163 |
    |  721 |  319 | 1189 | 1285 | 1133 |    13.880 |   -16.474 |    23.297 | -0.694 |  0.824 |
    |  722 |  235 | 1227 |  695 | 1358 |    10.750 |    18.883 |    26.704 | -0.537 | -0.944 |
    |  723 | 1116 |  661 | 1149 | 1123 |     0.617 |    -0.436 |     0.617 | -0.031 |  0.022 |
    |  724 |  255 |  874 |  872 | 1171 |     0.111 |     0.398 |     0.563 | -0.006 | -0.020 |
    |  725 | 1054 | 1170 |  659 |  660 |     0.026 |    -0.015 |     0.021 | -0.001 |  0.001 |
    |  726 |  785 |  438 |  439 |  440 |     3.007 |     3.435 |     4.858 | -0.150 | -0.172 |
    |  727 |  335 |  574 | 1155 |  333 |     4.559 |    -3.070 |     4.342 | -0.228 |  0.154 |
    |  728 |  127 | 1097 | 1172 |  126 |     0.002 |     0.034 |     0.047 | -0.000 | -0.002 |
    |  729 | 1059 | 1173 |  610 |  634 |     0.465 |    -0.179 |     0.253 | -0.023 |  0.009 |
    |  730 |   93 |   94 | 1177 | 1070 |     0.059 |     0.003 |     0.005 | -0.003 | -0.000 |
    |  731 |  261 | 1152 |  150 |  151 |     0.126 |     1.446 |     2.046 | -0.006 | -0.072 |
    |  732 |  730 |  722 |  721 |  729 |     2.364 |    -7.351 |    10.396 | -0.118 |  0.368 |
    |  733 |  708 |  706 |  705 |  707 |     4.794 |     1.913 |     2.706 | -0.240 | -0.096 |
    |  734 |  310 | 1189 |  319 |  312 |    19.849 |   -12.624 |    17.852 | -0.992 |  0.631 |
    |  735 |   68 |   69 | 1182 |  305 |     3.144 |     0.242 |     0.342 | -0.157 | -0.012 |
    |  736 |  562 |  339 |  340 | 1150 |     2.695 |    -2.022 |     2.859 | -0.135 |  0.101 |
    |  737 |   45 |   46 |  820 | 1153 |     2.364 |    -0.186 |     0.263 | -0.118 |  0.009 |
    |  738 |  731 |  730 |  733 |  732 |     3.141 |    -6.341 |     8.968 | -0.157 |  0.317 |
    |  739 |  298 | 1154 |  103 |  104 |     0.009 |     0.001 |     0.001 | -0.000 | -0.000 |
    |  740 |  592 |  640 | 1179 |  671 |     0.157 |     0.058 |     0.083 | -0.008 | -0.003 |
    |  741 |  861 |  863 | 1184 |  864 |     0.024 |    -0.063 |     0.088 | -0.001 |  0.003 |
    |  742 | 1180 | 1002 | 1240 |  639 |     0.042 |     0.016 |     0.022 | -0.002 | -0.001 |
    |  743 | 1088 | 1087 | 1280 | 1089 |     0.810 |     2.991 |     4.230 | -0.040 | -0.150 |
    |  744 |  782 |  783 | 1158 |  781 |     1.471 |     4.558 |     6.446 | -0.074 | -0.228 |
    |  745 |  882 | 1099 |  254 | 1178 |     0.060 |    -0.264 |     0.373 | -0.003 |  0.013 |
    |  746 |   60 |  717 | 1283 |  296 |    11.208 |     0.801 |     1.133 | -0.560 | -0.040 |
    |  747 | 1052 |  660 |  616 | 1156 |     0.024 |    -0.010 |     0.014 | -0.001 |  0.001 |
    |  748 |  271 | 1005 |  644 | 1198 |     0.688 |     0.149 |     0.211 | -0.034 | -0.007 |
    |  749 | 1086 | 1158 |  783 |  786 |     1.432 |     3.984 |     5.635 | -0.072 | -0.199 |
    |  750 |  815 | 1159 |  804 |  806 |     0.732 |    -1.975 |     2.793 | -0.037 |  0.099 |
    |  751 |  789 |  804 | 1159 |  790 |     0.791 |    -2.290 |     3.239 | -0.040 |  0.115 |
    |  752 |  945 | 1181 |  942 |  943 |     0.592 |     1.614 |     2.283 | -0.030 | -0.081 |
    |  753 |  357 | 1151 |  332 |  330 |     4.455 |    -5.020 |     7.100 | -0.223 |  0.251 |
    |  754 | 1289 | 1316 | 1160 | 1285 |     7.465 |   -14.750 |    20.859 | -0.373 |  0.737 |
    |  755 | 1243 |  936 |  933 |  934 |     0.257 |     0.863 |     1.221 | -0.013 | -0.043 |
    |  756 |  301 |  705 |  704 |  703 |     6.056 |     1.776 |     2.512 | -0.303 | -0.089 |
    |  757 | 1302 | 1098 |  253 | 1286 |     0.040 |     0.220 |     0.311 | -0.002 | -0.011 |
    |  758 | 1128 | 1170 | 1210 |  487 |     0.028 |    -0.021 |     0.030 | -0.001 |  0.001 |
    |  759 |  339 |  562 | 1175 |  337 |     3.283 |    -2.367 |     3.347 | -0.164 |  0.118 |
    |  760 | 1012 | 1016 | 1190 | 1013 |     0.959 |    -0.491 |     0.694 | -0.048 |  0.025 |
    |  761 |  909 |  907 | 1191 |  910 |     0.044 |     0.114 |     0.162 | -0.002 | -0.006 |
    |  762 |  438 |  436 |  435 |  437 |     3.847 |     3.968 |     5.612 | -0.192 | -0.198 |
    |  763 | 1198 | 1162 |  978 |  271 |     0.796 |     0.158 |     0.224 | -0.040 | -0.008 |
    |  764 |  335 |  333 |  332 |  334 |     4.196 |    -3.786 |     5.354 | -0.210 |  0.189 |
    |  765 |  720 |  723 |  728 |  754 |     2.534 |    -9.955 |    14.078 | -0.127 |  0.498 |
    |  766 |  652 |  636 | 1196 |  694 |     0.005 |    -0.008 |     0.011 | -0.000 |  0.000 |
    |  767 |  691 | 1163 |  904 |  693 |     0.124 |    -0.495 |     0.700 | -0.006 |  0.025 |
    |  768 |  289 |  787 |  781 | 1158 |     0.961 |     4.551 |     6.436 | -0.048 | -0.228 |
    |  769 |  761 | 1147 |  763 |  760 |     5.082 |    -1.739 |     2.459 | -0.254 |  0.087 |
    |  770 |  343 |  558 | 1192 |  345 |     1.319 |    -1.788 |     2.528 | -0.066 |  0.089 |
    |  771 |  731 |  357 | 1318 |  734 |     4.805 |    -6.572 |     9.294 | -0.240 |  0.329 |
    |  772 |  362 |  758 |  598 |  360 |    14.190 |     7.657 |    10.829 | -0.709 | -0.383 |
    |  773 |  601 |  618 | 1198 |  644 |     0.611 |     0.215 |     0.305 | -0.031 | -0.011 |
    |  774 | 1357 |  243 | 1392 |  718 |     0.004 |    -0.024 |     0.035 | -0.000 |  0.001 |
    |  775 |  704 |  702 |  701 |  703 |     6.631 |     2.445 |     3.457 | -0.332 | -0.122 |
    |  776 |  334 |  332 | 1151 |  365 |     3.803 |    -4.354 |     6.157 | -0.190 |  0.218 |
    |  777 |   27 | 1050 | 1166 |  280 |     0.125 |    -0.015 |     0.021 | -0.006 |  0.001 |
    |  778 |  131 |  248 | 1165 |  130 |     0.005 |     0.063 |     0.090 | -0.000 | -0.003 |
    |  779 |  781 |  780 | 1312 |  782 |     1.842 |     5.340 |     7.552 | -0.092 | -0.267 |
    |  780 | 1123 | 1149 | 1190 |  484 |     0.742 |    -0.522 |     0.739 | -0.037 |  0.026 |
    |  781 |  444 |  577 | 1174 |  442 |     1.819 |     2.934 |     4.150 | -0.091 | -0.147 |
    |  782 |  913 |  914 | 1212 |  919 |     0.064 |     0.157 |     0.223 | -0.003 | -0.008 |
    |  783 | 1030 | 1057 | 1210 | 1055 |     0.036 |    -0.019 |     0.027 | -0.002 |  0.001 |
    |  784 |  743 |  745 |  746 |  744 |    10.624 |    -2.544 |     3.598 | -0.531 |  0.127 |
    |  785 |  333 |  331 |  330 |  332 |     4.922 |    -4.195 |     5.932 | -0.246 |  0.210 |
    |  786 |  785 |  440 |  442 | 1174 |     2.322 |     3.327 |     4.705 | -0.116 | -0.166 |
    |  787 |  148 |  149 | 1203 |  259 |     0.078 |     1.053 |     1.488 | -0.004 | -0.053 |
    |  788 |  456 |  455 | 1202 |  457 |     0.968 |     0.902 |     1.276 | -0.048 | -0.045 |
    |  789 |  101 |  102 | 1206 |  288 |     0.015 |     0.001 |     0.002 | -0.001 | -0.000 |
    |  790 | 1000 | 1240 | 1002 |  995 |     0.035 |     0.013 |     0.018 | -0.002 | -0.001 |
    |  791 | 1258 | 1205 |  338 |  336 |     2.719 |    -3.370 |     4.765 | -0.136 |  0.168 |
    |  792 |  888 |  887 | 1213 |  889 |     0.057 |    -0.184 |     0.261 | -0.003 |  0.009 |
    |  793 | 1224 | 1216 |  965 |  282 |     0.075 |     0.016 |     0.023 | -0.004 | -0.001 |
    |  794 |  669 | 1224 |  282 | 1070 |     0.063 |     0.013 |     0.018 | -0.003 | -0.001 |
    |  795 |  598 |  358 |  359 |  360 |    16.294 |     4.392 |     6.211 | -0.815 | -0.220 |
    |  796 |  869 | 1171 |  872 |  867 |     0.127 |     0.351 |     0.497 | -0.006 | -0.018 |
    |  797 | 1176 |  570 |  325 |  327 |     7.095 |    -4.880 |     6.901 | -0.355 |  0.244 |
    |  798 | 1117 |  659 | 1170 | 1128 |     0.023 |    -0.018 |     0.025 | -0.001 |  0.001 |
    |  799 |  244 |  125 |  126 | 1172 |     0.001 |     0.029 |     0.041 | -0.000 | -0.001 |
    |  800 |  654 |  964 |  962 | 1216 |     0.077 |     0.030 |     0.042 | -0.004 | -0.001 |
    |  801 |  611 |  610 | 1173 |  643 |     0.537 |    -0.191 |     0.270 | -0.027 |  0.010 |
    |  802 |  783 |  785 | 1174 |  786 |     1.961 |     3.692 |     5.222 | -0.098 | -0.185 |
    |  803 |  831 |  833 | 1218 |  832 |     0.006 |     0.022 |     0.031 | -0.000 | -0.001 |
    |  804 |  450 |  543 | 1223 | 1132 |     1.155 |     1.757 |     2.484 | -0.058 | -0.088 |
    |  805 |   27 |   28 | 1220 | 1050 |     0.139 |    -0.014 |     0.020 | -0.007 |  0.001 |
    |  806 | 1237 |  204 |  205 | 1221 |     0.009 |    -0.189 |     0.268 | -0.000 |  0.009 |
    |  807 |  296 | 1391 |  749 |  748 |    12.223 |     0.726 |     1.027 | -0.611 | -0.036 |
    |  808 |  430 |  432 |  542 | 1169 |     4.593 |     6.357 |     8.990 | -0.230 | -0.318 |
    |  809 |  331 |  329 |  328 |  330 |     5.373 |    -4.691 |     6.634 | -0.269 |  0.235 |
    |  810 |  284 | 1177 |   94 |   95 |     0.051 |     0.004 |     0.005 | -0.003 | -0.000 |
    |  811 |  447 | 1132 | 1217 |  446 |     1.393 |     2.205 |     3.119 | -0.070 | -0.110 |
    |  812 |  882 | 1178 |  885 |  881 |     0.091 |    -0.251 |     0.354 | -0.005 |  0.013 |
    |  813 |  434 |  432 |  431 |  433 |     5.239 |     4.983 |     7.047 | -0.262 | -0.249 |
    |  814 | 1208 |  352 |  354 | 1222 |     0.892 |    -0.785 |     1.110 | -0.045 |  0.039 |
    |  815 |  574 |  335 |  337 | 1175 |     3.888 |    -2.764 |     3.909 | -0.194 |  0.138 |
    |  816 |  638 |  641 | 1224 |  669 |     0.058 |     0.021 |     0.029 | -0.003 | -0.001 |
    |  817 | 1288 |  260 | 1314 |  952 |     0.182 |    -0.923 |     1.305 | -0.009 |  0.046 |
    |  818 |  464 |  429 |  428 |  430 |     6.143 |     7.405 |    10.473 | -0.307 | -0.370 |
    |  819 |  742 |  743 |  744 |  741 |    11.984 |    -2.255 |     3.189 | -0.599 |  0.113 |
    |  820 |  431 |  530 | 1161 |  433 |     5.586 |     4.405 |     6.230 | -0.279 | -0.220 |
    |  821 |  322 |  546 | 1248 | 1164 |     7.497 |   -10.053 |    14.217 | -0.375 |  0.503 |
    |  822 |  428 |  431 |  432 |  430 |     5.838 |     6.096 |     8.622 | -0.292 | -0.305 |
    |  823 |  639 |  638 |  669 | 1180 |     0.049 |     0.017 |     0.024 | -0.002 | -0.001 |
    |  824 | 1234 | 1228 |  404 |  406 |     0.035 |    -0.031 |     0.043 | -0.002 |  0.002 |
    |  825 |  434 |  433 | 1161 |  435 |     4.817 |     4.362 |     6.169 | -0.241 | -0.218 |
    |  826 |  637 |  697 | 1229 |  617 |     0.004 |     0.007 |     0.010 | -0.000 | -0.000 |
    |  827 |  351 |  349 | 1232 |  585 |     0.855 |    -1.183 |     1.672 | -0.043 |  0.059 |
    |  828 |  955 |  942 | 1181 | 1079 |     0.853 |     1.448 |     2.048 | -0.043 | -0.072 |
    |  829 |  688 |  690 | 1279 |  689 |     2.299 |     7.096 |    10.035 | -0.115 | -0.355 |
    |  830 |  966 | 1182 |   69 |   70 |     2.685 |     0.153 |     0.217 | -0.134 | -0.008 |
    |  831 |  923 |  922 |  864 | 1184 |     0.018 |    -0.052 |     0.074 | -0.001 |  0.003 |
    |  832 |  918 | 1233 |  133 |  250 |     0.010 |     0.091 |     0.128 | -0.001 | -0.005 |
    |  833 | 1284 | 1227 | 1238 |  758 |    13.327 |    12.975 |    18.349 | -0.666 | -0.649 |
    |  834 | 1211 |  734 |  324 | 1164 |     6.293 |    -7.749 |    10.959 | -0.315 |  0.387 |
    |  835 |  701 |  700 |  651 |  699 |     8.030 |     2.425 |     3.430 | -0.402 | -0.121 |
    |  836 |  362 |  364 |  426 |  425 |    10.296 |     7.581 |    10.721 | -0.515 | -0.379 |
    |  837 |  331 |  578 | 1176 |  329 |     6.049 |    -4.019 |     5.684 | -0.302 |  0.201 |
    |  838 |  912 | 1235 |  134 |  135 |     0.008 |     0.117 |     0.165 | -0.000 | -0.006 |
    |  839 |  765 | 1186 |  794 |  764 |     3.727 |    -1.282 |     1.814 | -0.186 |  0.064 |
    |  840 | 1176 |  327 |  328 |  329 |     6.386 |    -4.852 |     6.862 | -0.319 |  0.243 |
    |  841 |  547 |  622 |  808 |  621 |     3.898 |    10.791 |    15.261 | -0.195 | -0.540 |
    |  842 |  248 |  131 |  132 | 1185 |     0.003 |     0.074 |     0.104 | -0.000 | -0.004 |
    |  843 |  901 |  308 | 1384 |  826 |     0.001 |    -0.015 |     0.022 | -0.000 |  0.001 |
    |  844 | 1257 |  870 | 1286 |  253 |     0.057 |     0.261 |     0.368 | -0.003 | -0.013 |
    |  845 |  635 | 1187 |  647 |  612 |     0.016 |    -0.008 |     0.011 | -0.001 |  0.000 |
    |  846 |  123 | 1188 |  242 |  122 |     0.001 |     0.020 |     0.028 | -0.000 | -0.001 |
    |  847 | 1149 | 1019 | 1013 | 1190 |     0.803 |    -0.406 |     0.574 | -0.040 |  0.020 |
    |  848 |  597 |  645 | 1245 |  673 |     0.012 |     0.006 |     0.008 | -0.001 | -0.000 |
    |  849 |  438 |  437 | 1214 |  439 |     3.441 |     3.502 |     4.952 | -0.172 | -0.175 |
    |  850 |  918 | 1191 |  907 |  908 |     0.031 |     0.097 |     0.138 | -0.002 | -0.005 |
    |  851 |  676 |  825 |  608 | 1252 |     0.001 |     0.012 |     0.017 | -0.000 | -0.001 |
    |  852 |  964 |  654 |  653 | 1251 |     0.066 |     0.036 |     0.051 | -0.003 | -0.002 |
    |  853 |  215 |  245 | 1250 |  214 |     0.002 |    -0.041 |     0.057 | -0.000 |  0.002 |
    |  854 |  232 |  233 | 1193 |  535 |     0.001 |    -0.005 |     0.006 | -0.000 |  0.000 |
    |  855 |  108 |  109 |  534 | 1194 |     0.001 |     0.003 |     0.005 | -0.000 | -0.000 |
    |  856 |  707 |  792 |  239 | 1386 |     4.728 |     0.958 |     1.355 | -0.236 | -0.048 |
    |  857 |  119 | 1103 | 1195 |  118 |     0.001 |     0.014 |     0.019 | -0.000 | -0.001 |
    |  858 |  828 |  829 | 1253 |  827 |     0.002 |    -0.012 |     0.016 | -0.000 |  0.001 |
    |  859 |  529 | 1121 | 1196 |  636 |     0.006 |    -0.009 |     0.012 | -0.000 |  0.000 |
    |  860 |  877 |  875 | 1256 |  878 |     0.191 |     0.540 |     0.763 | -0.010 | -0.027 |
    |  861 |  125 |  244 | 1255 |  124 |     0.002 |     0.025 |     0.036 | -0.000 | -0.001 |
    |  862 |  499 |  532 | 1251 | 1131 |     0.063 |     0.047 |     0.067 | -0.003 | -0.002 |
    |  863 | 1317 |  423 | 1347 | 1048 |     0.218 |    -0.148 |     0.210 | -0.011 |  0.007 |
    |  864 |  667 |  622 |  547 |  623 |     3.624 |     9.613 |    13.595 | -0.181 | -0.481 |
    |  865 |  139 |  140 | 1257 |  253 |     0.014 |     0.259 |     0.366 | -0.001 | -0.013 |
    |  866 |  750 | 1381 | 1209 |  545 |    14.465 |     0.187 |     0.264 | -0.723 | -0.009 |
    |  867 |  709 |  711 | 1373 |  710 |     3.431 |     1.463 |     2.069 | -0.172 | -0.073 |
    |  868 |  972 |  974 | 1239 |  976 |     1.015 |     0.535 |     0.757 | -0.051 | -0.027 |
    |  869 |  429 |  426 |  427 |  428 |     7.977 |     6.666 |     9.427 | -0.399 | -0.333 |
    |  870 |  725 |  726 | 1248 |  727 |     4.867 |   -10.941 |    15.472 | -0.243 |  0.547 |
    |  871 | 1079 | 1181 | 1223 |  543 |     0.956 |     1.603 |     2.267 | -0.048 | -0.080 |
    |  872 | 1162 | 1198 |  618 |  664 |     0.725 |     0.264 |     0.373 | -0.036 | -0.013 |
    |  873 |  921 |  924 | 1271 |  927 |     0.012 |    -0.041 |     0.058 | -0.001 |  0.002 |
    |  874 |  313 |  315 |  803 |  311 |    22.749 |    -4.348 |     6.149 | -1.137 |  0.217 |
    |  875 | 1269 |  989 | 1387 |  483 |     0.222 |     0.157 |     0.221 | -0.011 | -0.008 |
    |  876 |  327 |  325 |  324 |  326 |     7.242 |    -5.999 |     8.484 | -0.362 |  0.300 |
    |  877 | 1259 | 1028 | 1027 | 1267 |     0.050 |    -0.026 |     0.036 | -0.003 |  0.001 |
    |  878 |  427 | 1168 |  431 |  428 |     6.951 |     5.195 |     7.347 | -0.348 | -0.260 |
    |  879 | 1308 |  207 |  208 | 1268 |     0.010 |    -0.120 |     0.169 | -0.001 |  0.006 |
    |  880 |  245 |  897 |  896 | 1271 |     0.008 |    -0.036 |     0.051 | -0.000 |  0.002 |
    |  881 |  194 |  260 | 1261 |  193 |     0.081 |    -1.057 |     1.495 | -0.004 |  0.053 |
    |  882 | 1080 |  292 | 1355 | 1229 |     0.002 |     0.008 |     0.011 | -0.000 | -0.000 |
    |  883 | 1260 | 1273 |  855 |  859 |     0.040 |    -0.139 |     0.197 | -0.002 |  0.007 |
    |  884 |  480 |  478 | 1269 |  483 |     0.189 |     0.173 |     0.244 | -0.009 | -0.009 |
    |  885 |  868 |  871 | 1286 |  870 |     0.079 |     0.248 |     0.351 | -0.004 | -0.012 |
    |  886 |  663 | 1137 | 1200 |  664 |     0.763 |     0.397 |     0.561 | -0.038 | -0.020 |
    |  887 | 1305 | 1275 |  398 |  400 |     0.056 |    -0.047 |     0.067 | -0.003 |  0.002 |
    |  888 |  668 |  667 | 1297 | 1262 |     2.545 |    10.047 |    14.209 | -0.127 | -0.502 |
    |  889 | 1073 | 1074 | 1288 |  950 |     0.356 |    -1.055 |     1.491 | -0.018 |  0.053 |
    |  890 |  453 |  540 | 1202 |  455 |     1.171 |     0.891 |     1.260 | -0.059 | -0.045 |
    |  891 | 1152 | 1203 |  149 |  150 |     0.070 |     1.231 |     1.741 | -0.003 | -0.062 |
    |  892 | 1148 | 1211 |  726 |  724 |     4.508 |    -8.797 |    12.440 | -0.225 |  0.440 |
    |  893 |  451 | 1127 | 1278 | 1120 |     1.540 |     1.132 |     1.600 | -0.077 | -0.057 |
    |  894 |  755 | 1356 |  892 |  535 |     0.002 |    -0.006 |     0.008 | -0.000 |  0.000 |
    |  895 | 1154 | 1206 |  102 |  103 |     0.012 |     0.001 |     0.002 | -0.001 | -0.000 |
    |  896 | 1221 |  889 | 1213 | 1237 |     0.040 |    -0.190 |     0.269 | -0.002 |  0.010 |
    |  897 | 1284 |  696 |  695 | 1227 |     9.841 |    12.568 |    17.773 | -0.492 | -0.628 |
    |  898 |  352 | 1208 |  533 |  350 |     1.006 |    -0.913 |     1.292 | -0.050 |  0.046 |
    |  899 |  500 |  502 | 1298 |  586 |     0.037 |     0.056 |     0.079 | -0.002 | -0.003 |
    |  900 |  897 |  245 |  215 | 1296 |     0.004 |    -0.035 |     0.050 | -0.000 |  0.002 |
    |  901 | 1170 | 1054 | 1055 | 1210 |     0.030 |    -0.016 |     0.023 | -0.001 |  0.001 |
    |  902 | 1270 | 1300 |  146 |  147 |     0.047 |     0.776 |     1.098 | -0.002 | -0.039 |
    |  903 |  226 | 1325 |  225 |    8 |     0.000 |    -0.013 |     0.018 | -0.000 |  0.001 |
    |  904 |  116 |    3 |  117 | 1324 |     0.000 |     0.013 |     0.018 | -0.000 | -0.001 |
    |  905 |  691 | 1378 |  258 | 1163 |     0.123 |    -0.586 |     0.828 | -0.006 |  0.029 |
    |  906 |  650 |  651 | 1207 |  605 |     9.983 |     2.971 |     4.201 | -0.499 | -0.149 |
    |  907 |  472 |  486 | 1303 |  474 |     0.294 |     0.258 |     0.364 | -0.015 | -0.013 |
    |  908 |  822 |  823 | 1325 |  609 |     0.001 |    -0.013 |     0.018 | -0.000 |  0.001 |
    |  909 |  865 |  210 |  211 | 1306 |     0.003 |    -0.074 |     0.105 | -0.000 |  0.004 |
    |  910 |  488 |  919 | 1212 |  587 |     0.088 |     0.170 |     0.241 | -0.004 | -0.009 |
    |  911 |  345 | 1192 | 1232 |  347 |     1.099 |    -1.467 |     2.074 | -0.055 |  0.073 |
    |  912 |  177 | 1329 |  176 |    7 |    -0.279 |   -11.118 |    15.723 |  0.014 |  0.556 |
    |  913 |  222 |  223 | 1281 |  308 |     0.000 |    -0.014 |     0.020 | -0.000 |  0.001 |
    |  914 |  687 |  623 | 1201 |  688 |     3.215 |     8.030 |    11.356 | -0.161 | -0.401 |
    |  915 | 1178 | 1213 |  887 |  885 |     0.078 |    -0.215 |     0.305 | -0.004 |  0.011 |
    |  916 |  259 | 1203 |  940 | 1243 |     0.193 |     1.046 |     1.479 | -0.010 | -0.052 |
    |  917 |  384 |  423 | 1317 |  382 |     0.186 |    -0.169 |     0.238 | -0.009 |  0.008 |
    |  918 |  951 |  953 | 1378 |  692 |     0.195 |    -0.690 |     0.976 | -0.010 |  0.035 |
    |  919 |  913 |  911 | 1395 |  914 |     0.046 |     0.143 |     0.202 | -0.002 | -0.007 |
    |  920 | 1169 | 1254 |  688 | 1201 |     3.364 |     7.458 |    10.547 | -0.168 | -0.373 |
    |  921 |  430 | 1169 | 1201 |  464 |     4.862 |     7.720 |    10.918 | -0.243 | -0.386 |
    |  922 |  234 |    1 |    9 | 1326 |     0.001 |    -0.001 |     0.001 | -0.000 |  0.000 |
    |  923 |  107 |    2 |  108 | 1327 |     0.001 |     0.001 |     0.001 | -0.000 | -0.000 |
    |  924 |  710 | 1374 | 1380 |  712 |     3.369 |     2.205 |     3.118 | -0.168 | -0.110 |
    |  925 |  975 | 1162 |  664 | 1200 |     0.839 |     0.289 |     0.408 | -0.042 | -0.014 |
    |  926 |  217 |  218 | 1392 |  243 |     0.002 |    -0.026 |     0.037 | -0.000 |  0.001 |
    |  927 |  895 |  898 | 1357 |  684 |     0.008 |    -0.028 |     0.039 | -0.000 |  0.001 |
    |  928 | 1164 |  324 |  323 |  322 |     8.548 |    -8.044 |    11.376 | -0.427 |  0.402 |
    |  929 |  969 |  970 | 1371 |  967 |     1.746 |     0.613 |     0.866 | -0.087 | -0.031 |
    |  930 | 1242 | 1204 |  325 |  570 |     8.463 |    -4.572 |     6.466 | -0.423 |  0.229 |
    |  931 |  683 |  682 | 1357 |  718 |     0.006 |    -0.023 |     0.032 | -0.000 |  0.001 |
    |  932 |  876 | 1376 |  143 |  144 |     0.019 |     0.483 |     0.683 | -0.001 | -0.024 |
    |  933 |  915 |  251 |  137 | 1344 |     0.015 |     0.171 |     0.242 | -0.001 | -0.009 |
    |  934 |  146 |  257 | 1379 |  145 |     0.021 |     0.627 |     0.886 | -0.001 | -0.031 |
    |  935 |  197 |  258 | 1345 |  196 |     0.037 |    -0.664 |     0.940 | -0.002 |  0.033 |
    |  936 |  394 | 1111 | 1336 | 1101 |     0.096 |    -0.056 |     0.079 | -0.005 |  0.003 |
    |  937 |  572 |  615 | 1377 |  599 |     0.129 |    -0.051 |     0.072 | -0.006 |  0.003 |
    |  938 |  999 | 1375 |  510 |  531 |     0.026 |     0.020 |     0.029 | -0.001 | -0.001 |
    |  939 | 1046 |  525 |  378 | 1346 |     0.288 |    -0.209 |     0.296 | -0.014 |  0.010 |
    |  940 |  652 |  677 | 1366 |  636 |     0.005 |    -0.006 |     0.009 | -0.000 |  0.000 |
    |  941 |  892 | 1356 |  652 |  694 |     0.003 |    -0.007 |     0.010 | -0.000 |  0.000 |
    |  942 |  961 |  965 | 1216 |  962 |     0.086 |     0.018 |     0.026 | -0.004 | -0.001 |
    |  943 |  841 |  835 |  832 | 1218 |     0.007 |     0.025 |     0.035 | -0.000 | -0.001 |
    |  944 |  556 |  629 | 1333 |  630 |     2.139 |     0.954 |     1.350 | -0.107 | -0.048 |
    |  945 |  440 |  439 | 1380 |  441 |     2.834 |     2.763 |     3.907 | -0.142 | -0.138 |
    |  946 |  435 |  537 | 1214 |  437 |     4.054 |     3.476 |     4.915 | -0.203 | -0.174 |
    |  947 |  330 |  328 | 1318 |  357 |     5.382 |    -5.260 |     7.439 | -0.269 |  0.263 |
    |  948 |  813 |  814 | 1361 |  816 |     0.530 |    -1.449 |     2.049 | -0.026 |  0.072 |
    |  949 |  277 | 1220 |   28 |   29 |     0.165 |    -0.008 |     0.012 | -0.008 |  0.000 |
    |  950 |  206 |  252 | 1221 |  205 |     0.012 |    -0.161 |     0.228 | -0.001 |  0.008 |
    |  951 |  784 | 1393 |  436 |  536 |     3.342 |     5.096 |     7.207 | -0.167 | -0.255 |
    |  952 |  802 |  736 | 1247 | 1215 |     1.521 |    11.667 |    16.500 | -0.076 | -0.583 |
    |  953 |  356 |  484 | 1222 |  354 |     0.779 |    -0.702 |     0.992 | -0.039 |  0.035 |
    |  954 |  255 | 1225 |  141 |  142 |     0.024 |     0.353 |     0.499 | -0.001 | -0.018 |
    |  955 |  974 | 1200 | 1137 | 1239 |     0.877 |     0.439 |     0.621 | -0.044 | -0.022 |
    |  956 | 1216 | 1224 |  641 |  654 |     0.066 |     0.023 |     0.033 | -0.003 | -0.001 |
    |  957 | 1016 | 1222 |  484 | 1190 |     0.854 |    -0.581 |     0.822 | -0.043 |  0.029 |
    |  958 |  247 |  212 |  213 | 1226 |     0.003 |    -0.055 |     0.078 | -0.000 |  0.003 |
    |  959 |  598 |  758 | 1238 |  757 |    16.683 |     9.590 |    13.562 | -0.834 | -0.479 |
    |  960 | 1222 | 1016 | 1244 | 1208 |     1.019 |    -0.687 |     0.972 | -0.051 |  0.034 |
    |  961 |  360 |  361 |  363 |  362 |    12.459 |     4.892 |     6.918 | -0.623 | -0.245 |
    |  962 | 1243 |  934 | 1270 |  259 |     0.176 |     0.903 |     1.277 | -0.009 | -0.045 |
    |  963 |  947 | 1223 | 1181 |  945 |     0.831 |     1.856 |     2.625 | -0.042 | -0.093 |
    |  964 |  772 |  774 | 1231 |  773 |     2.027 |    -4.031 |     5.700 | -0.101 |  0.202 |
    |  965 | 1255 |  833 | 1277 | 1188 |     0.003 |     0.021 |     0.030 | -0.000 | -0.001 |
    |  966 |  164 |  165 | 1343 |  603 |     0.534 |    10.566 |    14.943 | -0.027 | -0.528 |
    |  967 | 1050 | 1377 |  615 | 1166 |     0.126 |    -0.038 |     0.054 | -0.006 |  0.002 |
    |  968 |  311 |  803 | 1390 |  291 |    29.840 |    -3.902 |     5.518 | -1.492 |  0.195 |
    |  969 |  402 |  404 | 1228 |  539 |     0.040 |    -0.036 |     0.051 | -0.002 |  0.002 |
    |  970 |  716 | 1391 |  296 | 1283 |    11.681 |     1.435 |     2.029 | -0.584 | -0.072 |
    |  971 |  772 | 1291 | 1205 |  774 |     2.134 |    -3.589 |     5.076 | -0.107 |  0.179 |
    |  972 |  464 | 1201 |  623 |  547 |     4.114 |     9.003 |    12.732 | -0.206 | -0.450 |
    |  973 | 1006 |  278 | 1179 | 1310 |     0.200 |     0.043 |     0.061 | -0.010 | -0.002 |
    |  974 |  323 |  424 |  321 |  322 |    10.231 |    -7.575 |    10.712 | -0.512 |  0.379 |
    |  975 | 1381 |  749 | 1367 | 1209 |    13.187 |     1.281 |     1.812 | -0.659 | -0.064 |
    |  976 |  348 |  347 | 1232 |  349 |     1.088 |    -1.204 |     1.703 | -0.054 |  0.060 |
    |  977 | 1087 | 1086 |  786 | 1246 |     1.245 |     3.488 |     4.933 | -0.062 | -0.174 |
    |  978 | 1185 |  132 |  133 | 1233 |     0.005 |     0.084 |     0.119 | -0.000 | -0.004 |
    |  979 |  297 |  741 |  744 | 1199 |    11.496 |    -1.059 |     1.498 | -0.575 |  0.053 |
    |  980 |  701 |  702 | 1311 |  700 |     7.191 |     3.323 |     4.700 | -0.360 | -0.166 |
    |  981 |  317 |  315 |  313 |  316 |    19.134 |    -1.733 |     2.451 | -0.957 |  0.087 |
    |  982 |  651 |  650 | 1219 |  699 |     9.131 |     2.019 |     2.855 | -0.457 | -0.101 |
    |  983 |  432 |  434 | 1393 |  542 |     4.560 |     5.615 |     7.941 | -0.228 | -0.281 |
    |  984 |  406 |  408 |  487 | 1234 |     0.030 |    -0.028 |     0.040 | -0.002 |  0.001 |
    |  985 |  746 |  817 | 1199 |  744 |    10.496 |    -1.575 |     2.227 | -0.525 |  0.079 |
    |  986 |  705 |  706 | 1363 |  704 |     5.326 |     2.422 |     3.426 | -0.266 | -0.121 |
    |  987 | 1234 | 1057 | 1259 | 1228 |     0.039 |    -0.027 |     0.038 | -0.002 |  0.001 |
    |  988 |  426 |  364 | 1207 |  427 |     9.226 |     5.364 |     7.585 | -0.461 | -0.268 |
    |  989 |  133 |  134 | 1235 |  250 |     0.004 |     0.102 |     0.145 | -0.000 | -0.005 |
    |  990 |  972 |  976 | 1202 | 1293 |     1.111 |     0.659 |     0.932 | -0.056 | -0.033 |
    |  991 | 1133 |  321 |  320 |  319 |    12.943 |    -9.692 |    13.706 | -0.647 |  0.485 |
    |  992 |  204 | 1237 |  254 |  203 |     0.019 |    -0.221 |     0.312 | -0.001 |  0.011 |
    |  993 | 1072 |  585 | 1361 | 1073 |     0.579 |    -1.145 |     1.619 | -0.029 |  0.057 |
    |  994 |  580 |  829 | 1313 |  593 |     0.004 |    -0.011 |     0.016 | -0.000 |  0.001 |
    |  995 |  178 |  602 | 1329 |  177 |     0.587 |   -10.542 |    14.908 | -0.029 |  0.527 |
    |  996 |  532 | 1282 |  964 | 1251 |     0.075 |     0.044 |     0.062 | -0.004 | -0.002 |
    |  997 | 1124 |  485 | 1239 | 1137 |     0.806 |     0.555 |     0.784 | -0.040 | -0.028 |
    |  998 |  651 |  700 |  427 | 1207 |     8.814 |     4.159 |     5.882 | -0.441 | -0.208 |
    |  999 |  732 | 1307 |  365 | 1151 |     3.212 |    -5.006 |     7.080 | -0.161 |  0.250 |
    | 1000 | 1135 | 1183 | 1338 |  675 |     0.003 |     0.011 |     0.015 | -0.000 | -0.001 |
    | 1001 |  324 |  325 | 1204 |  323 |     8.311 |    -6.009 |     8.498 | -0.416 |  0.300 |
    | 1002 |  317 |  318 |  545 |  358 |    17.139 |     0.543 |     0.768 | -0.857 | -0.027 |
    | 1003 | 1215 |  603 | 1343 |  802 |     0.379 |    10.931 |    15.459 | -0.019 | -0.547 |
    | 1004 |  311 |  312 |  314 |  313 |    20.485 |    -6.290 |     8.896 | -1.024 |  0.315 |
    | 1005 |  604 |  605 | 1207 |  363 |    10.907 |     3.582 |     5.065 | -0.545 | -0.179 |
    | 1006 |  908 |  917 | 1304 |  918 |     0.028 |     0.083 |     0.118 | -0.001 | -0.004 |
    | 1007 | 1178 |  254 | 1237 | 1213 |     0.042 |    -0.222 |     0.314 | -0.002 |  0.011 |
    | 1008 |  640 |  658 | 1310 | 1179 |     0.180 |     0.063 |     0.089 | -0.009 | -0.003 |
    | 1009 |  366 |  318 |  317 |  316 |    16.941 |    -1.907 |     2.697 | -0.847 |  0.095 |
    | 1010 |  312 |  319 |  320 |  314 |    16.809 |   -10.213 |    14.444 | -0.840 |  0.511 |
    | 1011 |  617 | 1229 | 1355 |  685 |     0.004 |     0.008 |     0.012 | -0.000 | -0.000 |
    | 1012 |  687 | 1348 |  303 | 1297 |     1.919 |     8.410 |    11.893 | -0.096 | -0.420 |
    | 1013 |  362 |  363 | 1207 |  364 |    11.220 |     5.291 |     7.483 | -0.561 | -0.265 |
    | 1014 |  936 | 1243 |  940 |  938 |     0.332 |     1.005 |     1.422 | -0.017 | -0.050 |
    | 1015 | 1016 | 1012 | 1014 | 1244 |     1.094 |    -0.551 |     0.779 | -0.055 |  0.028 |
    | 1016 |  165 |    4 |  166 | 1343 |    -0.267 |    11.019 |    15.584 |  0.013 | -0.551 |
    | 1017 |  674 | 1249 |  656 |  646 |     0.162 |    -0.053 |     0.075 | -0.008 |  0.003 |
    | 1018 |  931 |  878 | 1341 |  932 |     0.207 |     0.673 |     0.951 | -0.010 | -0.034 |
    | 1019 |  324 |  734 | 1318 |  326 |     6.512 |    -6.796 |     9.611 | -0.326 |  0.340 |
    | 1020 | 1385 |  424 |  323 | 1204 |    10.300 |    -6.075 |     8.592 | -0.515 |  0.304 |
    | 1021 | 1226 |  213 |  214 | 1250 |     0.004 |    -0.047 |     0.066 | -0.000 |  0.002 |
    | 1022 |  849 | 1298 | 1394 |  847 |     0.026 |     0.053 |     0.075 | -0.001 | -0.003 |
    | 1023 |  577 | 1246 |  786 | 1174 |     1.574 |     3.286 |     4.647 | -0.079 | -0.164 |
    | 1024 | 1307 | 1368 |  773 | 1231 |     2.301 |    -4.596 |     6.500 | -0.115 |  0.230 |
    | 1025 | 1057 | 1234 |  487 | 1210 |     0.032 |    -0.023 |     0.032 | -0.002 |  0.001 |
    | 1026 |  753 |  266 | 1342 | 1292 |     8.179 |    -1.517 |     2.146 | -0.409 |  0.076 |
    | 1027 |  632 |  676 | 1252 |  681 |     0.002 |     0.013 |     0.018 | -0.000 | -0.001 |
    | 1028 | 1109 | 1131 | 1251 |  653 |     0.061 |     0.042 |     0.059 | -0.003 | -0.002 |
    | 1029 |  824 |  822 |  827 | 1253 |     0.002 |    -0.013 |     0.018 | -0.000 |  0.001 |
    | 1030 |  497 | 1107 | 1282 |  532 |     0.081 |     0.054 |     0.076 | -0.004 | -0.003 |
    | 1031 |  604 |  716 | 1283 |  605 |    11.439 |     2.120 |     2.998 | -0.572 | -0.106 |
    | 1032 | 1368 | 1307 |  732 |  733 |     2.502 |    -5.160 |     7.297 | -0.125 |  0.258 |
    | 1033 | 1188 |  123 |  124 | 1255 |     0.001 |     0.022 |     0.031 | -0.000 | -0.001 |
    | 1034 | 1003 |  288 | 1245 | 1295 |     0.017 |     0.004 |     0.006 | -0.001 | -0.000 |
    | 1035 |  874 |  876 | 1256 |  875 |     0.119 |     0.493 |     0.697 | -0.006 | -0.025 |
    | 1036 |  976 | 1239 |  485 | 1264 |     0.896 |     0.602 |     0.852 | -0.045 | -0.030 |
    | 1037 | 1038 | 1294 | 1249 |  277 |     0.192 |    -0.035 |     0.050 | -0.010 |  0.002 |
    | 1038 |  623 |  687 | 1297 |  667 |     2.477 |     9.320 |    13.180 | -0.124 | -0.466 |
    | 1039 |  735 |  603 | 1215 | 1262 |     1.310 |    10.258 |    14.508 | -0.065 | -0.513 |
    | 1040 |  737 | 1236 |  754 |  728 |     1.239 |   -11.239 |    15.895 | -0.062 |  0.562 |
    | 1041 | 1225 | 1257 |  140 |  141 |     0.022 |     0.301 |     0.426 | -0.001 | -0.015 |
    | 1042 |  175 | 1142 | 1236 |  737 |     0.886 |   -11.236 |    15.890 | -0.044 |  0.562 |
    | 1043 | 1301 | 1323 | 1217 |  947 |     1.014 |     2.336 |     3.304 | -0.051 | -0.117 |
    | 1044 |  657 | 1139 | 1350 |  658 |     0.186 |     0.095 |     0.134 | -0.009 | -0.005 |
    | 1045 | 1315 |  242 | 1188 | 1277 |     0.002 |     0.018 |     0.026 | -0.000 | -0.001 |
    | 1046 | 1393 |  784 | 1360 |  542 |     3.582 |     5.749 |     8.130 | -0.179 | -0.287 |
    | 1047 |  291 |  310 |  312 |  311 |    24.813 |   -11.688 |    16.530 | -1.241 |  0.584 |
    | 1048 |  889 | 1260 |  859 |  888 |     0.053 |    -0.161 |     0.228 | -0.003 |  0.008 |
    | 1049 |  546 | 1160 |  727 | 1248 |     7.260 |   -11.906 |    16.838 | -0.363 |  0.595 |
    | 1050 | 1057 | 1030 | 1028 | 1259 |     0.042 |    -0.021 |     0.029 | -0.002 |  0.001 |
    | 1051 | 1244 | 1014 | 1017 | 1263 |     1.318 |    -0.707 |     0.999 | -0.066 |  0.035 |
    | 1052 | 1336 | 1111 | 1362 | 1034 |     0.086 |    -0.047 |     0.066 | -0.004 |  0.002 |
    | 1053 | 1298 |  849 | 1383 |  586 |     0.030 |     0.059 |     0.083 | -0.001 | -0.003 |
    | 1054 | 1236 | 1142 | 1329 |  602 |     0.386 |   -10.974 |    15.520 | -0.019 |  0.549 |
    | 1055 |  779 |  237 | 1279 |  780 |     1.245 |     6.075 |     8.591 | -0.062 | -0.304 |
    | 1056 | 1308 | 1268 | 1337 |  856 |     0.019 |    -0.118 |     0.167 | -0.001 |  0.006 |
    | 1057 |  564 |  846 | 1394 |  851 |     0.023 |     0.044 |     0.063 | -0.001 | -0.002 |
    | 1058 |  336 |  334 |  365 | 1258 |     3.217 |    -3.899 |     5.515 | -0.161 |  0.195 |
    | 1059 |  845 |  847 | 1394 |  846 |     0.021 |     0.049 |     0.069 | -0.001 | -0.002 |
    | 1060 |  770 |  773 | 1368 |  778 |     1.515 |    -4.640 |     6.563 | -0.076 |  0.232 |
    | 1061 |  546 | 1133 | 1285 | 1160 |     8.272 |   -13.562 |    19.180 | -0.414 |  0.678 |
    | 1062 |  712 | 1214 |  537 | 1369 |     4.124 |     2.895 |     4.094 | -0.206 | -0.145 |
    | 1063 | 1333 |  967 | 1371 |  630 |     2.015 |     0.725 |     1.025 | -0.101 | -0.036 |
    | 1064 |  348 |  533 | 1263 | 1140 |     1.264 |    -1.013 |     1.432 | -0.063 |  0.051 |
    | 1065 |  457 | 1264 |  485 |  459 |     0.835 |     0.731 |     1.034 | -0.042 | -0.037 |
    | 1066 | 1282 | 1107 | 1319 |  963 |     0.097 |     0.044 |     0.062 | -0.005 | -0.002 |
    | 1067 | 1265 |  680 | 1274 | 1197 |    13.733 |    -4.827 |     6.826 | -0.687 |  0.241 |
    | 1068 |  193 | 1261 | 1157 |  192 |     0.067 |    -1.241 |     1.755 | -0.003 |  0.062 |
    | 1069 | 1036 | 1037 | 1267 | 1027 |     0.058 |    -0.029 |     0.040 | -0.003 |  0.001 |
    | 1070 |  252 |  206 |  207 | 1308 |     0.005 |    -0.142 |     0.201 | -0.000 |  0.007 |
    | 1071 |  259 | 1270 |  147 |  148 |     0.048 |     0.900 |     1.273 | -0.002 | -0.045 |
    | 1072 |  926 |  927 | 1271 |  896 |     0.014 |    -0.035 |     0.050 | -0.001 |  0.002 |
    | 1073 |  678 |  642 | 1373 | 1241 |     2.952 |     1.019 |     1.441 | -0.148 | -0.051 |
    | 1074 | 1263 |  533 | 1208 | 1244 |     1.144 |    -0.821 |     1.162 | -0.057 |  0.041 |
    | 1075 |  242 | 1315 | 1353 | 1104 |     0.002 |     0.017 |     0.024 | -0.000 | -0.001 |
    | 1076 |  536 |  782 | 1312 |  784 |     2.501 |     5.197 |     7.350 | -0.125 | -0.260 |
    | 1077 |  294 |  714 |  679 | 1351 |     4.376 |    14.235 |    20.132 | -0.219 | -0.712 |
    | 1078 | 1242 |  767 |  753 | 1292 |     7.920 |    -2.727 |     3.857 | -0.396 |  0.136 |
    | 1079 |  814 | 1157 | 1261 | 1074 |     0.269 |    -1.257 |     1.778 | -0.013 |  0.063 |
    | 1080 |  363 |  361 | 1230 |  604 |    12.453 |     3.716 |     5.255 | -0.623 | -0.186 |
    | 1081 |  341 | 1272 |  558 |  343 |     1.408 |    -2.121 |     3.000 | -0.070 |  0.106 |
    | 1082 | 1205 | 1258 | 1231 |  774 |     2.450 |    -3.928 |     5.555 | -0.122 |  0.196 |
    | 1083 |  359 |  358 |  545 | 1209 |    15.359 |     1.817 |     2.569 | -0.768 | -0.091 |
    | 1084 | 1087 | 1246 |  577 | 1280 |     1.234 |     3.056 |     4.322 | -0.062 | -0.153 |
    | 1085 |  360 |  359 | 1230 |  361 |    13.515 |     4.355 |     6.159 | -0.676 | -0.218 |
    | 1086 |  829 |  828 |  891 | 1313 |     0.003 |    -0.010 |     0.015 | -0.000 |  0.001 |
    | 1087 |  453 | 1120 | 1332 |  540 |     1.334 |     0.928 |     1.313 | -0.067 | -0.046 |
    | 1088 |  823 | 1281 |  223 |  224 |     0.001 |    -0.014 |     0.019 | -0.000 |  0.001 |
    | 1089 |  801 | 1287 | 1263 | 1017 |     1.568 |    -0.806 |     1.139 | -0.078 |  0.040 |
    | 1090 |  396 |  398 | 1275 |  553 |     0.065 |    -0.049 |     0.069 | -0.003 |  0.002 |
    | 1091 |  833 |  831 |  672 | 1277 |     0.005 |     0.019 |     0.027 | -0.000 | -0.001 |
    | 1092 |  401 |  402 | 1305 |  400 |     0.046 |    -0.046 |     0.065 | -0.002 |  0.002 |
    | 1093 | 1334 |  998 | 1370 |  523 |     0.021 |     0.015 |     0.022 | -0.001 | -0.001 |
    | 1094 |  822 |  824 | 1281 |  823 |     0.001 |    -0.013 |     0.019 | -0.000 |  0.001 |
    | 1095 | 1130 |  483 | 1387 | 1139 |     0.189 |     0.131 |     0.185 | -0.009 | -0.007 |
    | 1096 | 1332 | 1293 | 1202 |  540 |     1.258 |     0.797 |     1.127 | -0.063 | -0.040 |
    | 1097 |  556 |  630 | 1278 | 1127 |     1.881 |     1.066 |     1.508 | -0.094 | -0.053 |
    | 1098 |  340 |  541 | 1364 |  341 |     1.927 |    -2.357 |     3.333 | -0.096 |  0.118 |
    | 1099 | 1302 | 1286 |  871 |  930 |     0.074 |     0.219 |     0.310 | -0.004 | -0.011 |
    | 1100 |  689 |  751 |  303 | 1348 |     1.500 |     7.926 |    11.210 | -0.075 | -0.396 |
    | 1101 |  962 |  964 | 1282 |  963 |     0.085 |     0.037 |     0.052 | -0.004 | -0.002 |
    | 1102 |  679 |  714 | 1358 |  695 |     5.026 |    14.083 |    19.916 | -0.251 | -0.704 |
    | 1103 |  952 |  949 |  950 | 1288 |     0.274 |    -0.902 |     1.276 | -0.014 |  0.045 |
    | 1104 | 1263 | 1287 |  560 | 1140 |     1.496 |    -1.070 |     1.513 | -0.075 |  0.053 |
    | 1105 |  478 |  476 | 1303 | 1269 |     0.224 |     0.212 |     0.299 | -0.011 | -0.011 |
    | 1106 | 1219 | 1382 |   61 |   62 |     9.378 |     0.419 |     0.592 | -0.469 | -0.021 |
    | 1107 |  916 |  586 | 1383 |  917 |     0.036 |     0.070 |     0.099 | -0.002 | -0.003 |
    | 1108 |  702 |  704 | 1363 | 1339 |     6.032 |     3.158 |     4.466 | -0.302 | -0.158 |
    | 1109 |  771 |  775 | 1291 |  772 |     1.586 |    -3.452 |     4.882 | -0.079 |  0.173 |
    | 1110 | 1185 | 1233 |  918 | 1304 |     0.015 |     0.084 |     0.118 | -0.001 | -0.004 |
    | 1111 |  338 | 1205 | 1291 |  541 |     2.176 |    -3.200 |     4.526 | -0.109 |  0.160 |
    | 1112 |  587 | 1212 | 1302 |  930 |     0.072 |     0.199 |     0.282 | -0.004 | -0.010 |
    | 1113 | 1039 | 1049 | 1294 | 1038 |     0.223 |    -0.049 |     0.070 | -0.011 |  0.002 |
    | 1114 | 1280 | 1323 | 1301 | 1089 |     0.938 |     2.578 |     3.645 | -0.047 | -0.129 |
    | 1115 |  889 | 1221 |  252 | 1260 |     0.027 |    -0.165 |     0.233 | -0.001 |  0.008 |
    | 1116 |  680 |  316 |  313 |  314 |    17.286 |    -5.428 |     7.676 | -0.864 |  0.271 |
    | 1117 | 1088 | 1089 | 1301 |  263 |     0.583 |     2.652 |     3.750 | -0.029 | -0.133 |
    | 1118 | 1083 | 1003 | 1295 |  997 |     0.020 |     0.005 |     0.007 | -0.001 | -0.000 |
    | 1119 | 1104 | 1353 | 1389 |  309 |     0.001 |     0.015 |     0.022 | -0.000 | -0.001 |
    | 1120 | 1287 |  801 |  799 | 1309 |     1.920 |    -0.934 |     1.320 | -0.096 |  0.047 |
    | 1121 |  945 |  946 | 1301 |  947 |     0.671 |     2.112 |     2.987 | -0.034 | -0.106 |
    | 1122 |  175 |  176 | 1329 | 1142 |     0.615 |   -11.539 |    16.319 | -0.031 |  0.577 |
    | 1123 |  303 |  735 | 1262 | 1297 |     1.219 |     9.426 |    13.330 | -0.061 | -0.471 |
    | 1124 |  233 |  234 | 1326 | 1193 |     0.001 |    -0.003 |     0.004 | -0.000 |  0.000 |
    | 1125 |  475 |  474 | 1303 |  476 |     0.227 |     0.255 |     0.361 | -0.011 | -0.013 |
    | 1126 |  697 | 1266 | 1194 |  534 |     0.003 |     0.005 |     0.006 | -0.000 | -0.000 |
    | 1127 |  933 |  932 | 1341 |  934 |     0.225 |     0.763 |     1.078 | -0.011 | -0.038 |
    | 1128 |  118 | 1195 | 1324 |  117 |    -0.000 |     0.013 |     0.019 |  0.000 | -0.001 |
    | 1129 |  212 |  247 | 1306 |  211 |     0.006 |    -0.064 |     0.090 | -0.000 |  0.003 |
    | 1130 | 1247 |  668 | 1262 | 1215 |     1.491 |    10.771 |    15.233 | -0.075 | -0.539 |
    | 1131 |  655 |  656 | 1320 | 1138 |     0.174 |    -0.086 |     0.121 | -0.009 |  0.004 |
    | 1132 | 1007 | 1006 | 1310 | 1008 |     0.230 |     0.045 |     0.064 | -0.011 | -0.002 |
    | 1133 |  702 | 1339 |  530 | 1311 |     6.325 |     3.716 |     5.256 | -0.316 | -0.186 |
    | 1134 |  321 |  424 | 1274 |  320 |    12.841 |    -7.727 |    10.928 | -0.642 |  0.386 |
    | 1135 |  666 | 1330 |  997 | 1295 |     0.018 |     0.007 |     0.010 | -0.001 | -0.000 |
    | 1136 |  215 |  216 | 1328 | 1296 |     0.003 |    -0.034 |     0.048 | -0.000 |  0.002 |
    | 1137 | 1121 |  593 | 1313 | 1196 |     0.005 |    -0.010 |     0.014 | -0.000 |  0.001 |
    | 1138 | 1287 | 1309 | 1143 |  560 |     1.833 |    -1.167 |     1.650 | -0.092 |  0.058 |
    | 1139 | 1277 |  672 |  670 | 1315 |     0.004 |     0.017 |     0.025 | -0.000 | -0.001 |
    | 1140 |  260 |  194 |  195 | 1314 |     0.045 |    -0.911 |     1.288 | -0.002 |  0.046 |
    | 1141 |  648 | 1082 | 1331 | 1299 |     0.005 |    -0.003 |     0.004 | -0.000 |  0.000 |
    | 1142 |  314 |  320 | 1274 |  680 |    14.709 |    -6.187 |     8.750 | -0.735 |  0.309 |
    | 1143 |  577 | 1145 | 1323 | 1280 |     1.240 |     2.746 |     3.884 | -0.062 | -0.137 |
    | 1144 |  279 |  963 | 1319 |  960 |     0.110 |     0.034 |     0.048 | -0.006 | -0.002 |
    | 1145 | 1267 |  539 | 1228 | 1259 |     0.044 |    -0.029 |     0.041 | -0.002 |  0.001 |
    | 1146 | 1197 | 1385 |  745 |  743 |    11.232 |    -3.929 |     5.557 | -0.562 |  0.196 |
    | 1147 | 1294 | 1049 | 1043 | 1320 |     0.215 |    -0.071 |     0.100 | -0.011 |  0.004 |
    | 1148 |  658 | 1350 | 1008 | 1310 |     0.211 |     0.076 |     0.107 | -0.011 | -0.004 |
    | 1149 |  710 | 1373 |  809 | 1374 |     3.221 |     1.682 |     2.379 | -0.161 | -0.084 |
    | 1150 |  857 | 1322 |  865 |  862 |     0.015 |    -0.088 |     0.125 | -0.001 |  0.004 |
    | 1151 |  694 | 1196 | 1313 |  891 |     0.003 |    -0.009 |     0.013 | -0.000 |  0.000 |
    | 1152 |  596 |  592 |  671 | 1321 |     0.134 |     0.049 |     0.069 | -0.007 | -0.002 |
    | 1153 |  665 | 1136 | 1330 |  666 |     0.016 |     0.010 |     0.014 | -0.001 | -0.001 |
    | 1154 |  227 |  609 | 1325 |  226 |    -0.000 |    -0.012 |     0.018 |  0.000 |  0.001 |
    | 1155 |  115 |  116 | 1324 |  608 |    -0.000 |     0.013 |     0.018 |  0.000 | -0.001 |
    | 1156 |  681 | 1103 |  309 | 1389 |     0.002 |     0.014 |     0.020 | -0.000 | -0.001 |
    | 1157 |   10 |  648 | 1326 |    9 |     0.003 |    -0.001 |     0.001 | -0.000 |  0.000 |
    | 1158 |  106 |  107 | 1327 |  649 |     0.003 |     0.001 |     0.001 | -0.000 | -0.000 |
    | 1159 |  650 |  605 | 1283 |  717 |    10.450 |     1.782 |     2.520 | -0.523 | -0.089 |
    | 1160 |  243 | 1328 |  216 |  217 |     0.001 |    -0.029 |     0.041 | -0.000 |  0.001 |
    | 1161 |  402 |  539 | 1267 | 1305 |     0.048 |    -0.035 |     0.049 | -0.002 |  0.002 |
    | 1162 |  382 | 1317 | 1346 |  380 |     0.219 |    -0.198 |     0.281 | -0.011 |  0.010 |
    | 1163 |  993 |  997 | 1330 |  996 |     0.022 |     0.009 |     0.012 | -0.001 | -0.000 |
    | 1164 |  969 | 1293 | 1332 |  970 |     1.457 |     0.679 |     0.961 | -0.073 | -0.034 |
    | 1165 | 1069 | 1331 | 1082 |  299 |     0.007 |    -0.003 |     0.004 | -0.000 |  0.000 |
    | 1166 | 1138 | 1320 | 1043 | 1347 |     0.206 |    -0.105 |     0.149 | -0.010 |  0.005 |
    | 1167 |  743 |  742 | 1265 | 1197 |    12.439 |    -3.096 |     4.378 | -0.622 |  0.155 |
    | 1168 | 1037 | 1036 | 1033 | 1335 |     0.069 |    -0.035 |     0.050 | -0.003 |  0.002 |
    | 1169 | 1129 | 1138 | 1347 |  423 |     0.182 |    -0.124 |     0.176 | -0.009 |  0.006 |
    | 1170 |  523 |  513 |  512 | 1334 |     0.019 |     0.019 |     0.026 | -0.001 | -0.001 |
    | 1171 |  853 |  856 | 1337 |  857 |     0.029 |    -0.105 |     0.149 | -0.001 |  0.005 |
    | 1172 |  642 |  678 | 1333 |  629 |     2.493 |     0.893 |     1.264 | -0.125 | -0.045 |
    | 1173 | 1309 |  799 |  797 | 1352 |     2.138 |    -1.060 |     1.498 | -0.107 |  0.053 |
    | 1174 |  572 | 1101 | 1336 |  615 |     0.108 |    -0.056 |     0.080 | -0.005 |  0.003 |
    | 1175 |  632 |  675 | 1338 |  676 |     0.002 |     0.012 |     0.017 | -0.000 | -0.001 |
    | 1176 |  263 | 1340 |  153 |  154 |     0.138 |     2.300 |     3.252 | -0.007 | -0.115 |
    | 1177 | 1105 | 1008 | 1350 |  990 |     0.242 |     0.083 |     0.118 | -0.012 | -0.004 |
    | 1178 | 1098 | 1344 |  137 |  138 |     0.007 |     0.194 |     0.275 | -0.000 | -0.010 |
    | 1179 | 1136 | 1370 |  996 | 1330 |     0.019 |     0.011 |     0.016 | -0.001 | -0.001 |
    | 1180 | 1314 |  195 |  196 | 1345 |     0.057 |    -0.776 |     1.097 | -0.003 |  0.039 |
    | 1181 |  621 |  808 | 1351 |  679 |     4.084 |    11.259 |    15.923 | -0.204 | -0.563 |
    | 1182 |  604 | 1230 | 1367 |  716 |    12.254 |     2.666 |     3.770 | -0.613 | -0.133 |
    | 1183 | 1042 | 1048 | 1347 | 1043 |     0.232 |    -0.116 |     0.164 | -0.012 |  0.006 |
    | 1184 |  988 | 1349 |  983 |  984 |     0.334 |     0.175 |     0.247 | -0.017 | -0.009 |
    | 1185 |  797 | 1388 |  549 | 1352 |     2.286 |    -1.191 |     1.684 | -0.114 |  0.060 |
    | 1186 |  379 |  380 | 1346 |  378 |     0.241 |    -0.233 |     0.329 | -0.012 |  0.012 |
    | 1187 | 1315 |  670 |  957 | 1353 |     0.003 |     0.015 |     0.022 | -0.000 | -0.001 |
    | 1188 |  327 |  326 | 1318 |  328 |     5.942 |    -5.714 |     8.080 | -0.297 |  0.286 |
    | 1189 | 1160 | 1316 | 1167 |  727 |     5.094 |   -13.215 |    18.688 | -0.255 |  0.661 |
    | 1190 |  677 |  652 | 1356 |  755 |     0.004 |    -0.006 |     0.008 | -0.000 |  0.000 |
    | 1191 | 1032 | 1034 | 1362 | 1033 |     0.084 |    -0.034 |     0.048 | -0.004 |  0.002 |
    | 1192 |  613 |  684 | 1357 |  682 |     0.009 |    -0.023 |     0.033 | -0.000 |  0.001 |
    | 1193 | 1126 |  523 | 1370 | 1136 |     0.018 |     0.014 |     0.020 | -0.001 | -0.001 |
    | 1194 |  628 |  809 | 1373 |  642 |     2.741 |     1.440 |     2.036 | -0.137 | -0.072 |
    | 1195 |  553 | 1335 | 1033 | 1362 |     0.072 |    -0.039 |     0.056 | -0.004 |  0.002 |
    | 1196 | 1135 |  685 | 1355 | 1183 |     0.003 |     0.010 |     0.014 | -0.000 | -0.000 |
    | 1197 | 1139 | 1387 |  990 | 1350 |     0.218 |     0.113 |     0.160 | -0.011 | -0.006 |
    | 1198 |  698 | 1084 |  649 | 1365 |     0.004 |     0.003 |     0.004 | -0.000 | -0.000 |
    | 1199 |  396 |  553 | 1362 | 1111 |     0.078 |    -0.047 |     0.066 | -0.004 |  0.002 |
    | 1200 | 1309 | 1352 |  549 | 1143 |     2.071 |    -1.238 |     1.751 | -0.104 |  0.062 |
    | 1201 |  994 |  996 | 1370 |  998 |     0.023 |     0.013 |     0.018 | -0.001 | -0.001 |
    | 1202 |  529 |  636 | 1366 |  600 |     0.008 |    -0.007 |     0.010 | -0.000 |  0.000 |
    | 1203 |  712 | 1369 |  706 |  708 |     4.347 |     2.313 |     3.271 | -0.217 | -0.116 |
    | 1204 |  431 | 1168 | 1311 |  530 |     6.381 |     4.359 |     6.165 | -0.319 | -0.218 |
    | 1205 | 1260 |  252 | 1308 | 1273 |     0.028 |    -0.144 |     0.203 | -0.001 |  0.007 |
    | 1206 |  625 |  692 | 1378 |  691 |     0.219 |    -0.600 |     0.849 | -0.011 |  0.030 |
    | 1207 | 1334 |  512 |  510 | 1375 |     0.023 |     0.020 |     0.029 | -0.001 | -0.001 |
    | 1208 |  142 |  143 | 1376 |  255 |     0.027 |     0.403 |     0.571 | -0.001 | -0.020 |
    | 1209 |  876 |  144 |  145 | 1379 |     0.030 |     0.540 |     0.763 | -0.002 | -0.027 |
    | 1210 |  646 |  599 | 1377 |  674 |     0.146 |    -0.055 |     0.078 | -0.007 |  0.003 |
    | 1211 |  443 |  441 | 1380 | 1144 |     2.705 |     2.464 |     3.485 | -0.135 | -0.123 |
    | 1212 |  619 |  826 | 1384 |  620 |     0.003 |    -0.015 |     0.021 | -0.000 |  0.001 |
    | 1213 |  848 |  850 | 1383 |  849 |     0.022 |     0.062 |     0.087 | -0.001 | -0.003 |
    | 1214 |  736 |  294 | 1351 | 1247 |     1.634 |    12.404 |    17.541 | -0.082 | -0.620 |
    | 1215 | 1353 |  957 |  681 | 1389 |     0.002 |     0.015 |     0.021 | -0.000 | -0.001 |
    | 1216 |  910 |  912 | 1395 |  911 |     0.030 |     0.135 |     0.191 | -0.002 | -0.007 |
    | 1217 |  987 |  990 | 1387 |  989 |     0.251 |     0.128 |     0.181 | -0.013 | -0.006 |
    | 1218 |  807 | 1392 |  218 |  219 |     0.001 |    -0.022 |     0.031 | -0.000 |  0.001 |
    | 1219 |  917 | 1383 |  850 | 1304 |     0.024 |     0.071 |     0.100 | -0.001 | -0.004 |
    | 1220 |  248 | 1185 | 1304 |  850 |     0.016 |     0.073 |     0.103 | -0.001 | -0.004 |
    | 1221 |  756 | 1290 | 1390 |  803 |    27.441 |     3.743 |     5.293 | -1.372 | -0.187 |
    | 1222 |  824 | 1253 |  620 | 1384 |     0.002 |    -0.013 |     0.019 | -0.000 |  0.001 |
    | 1223 |  292 |  956 | 1183 | 1355 |     0.002 |     0.009 |     0.013 | -0.000 | -0.000 |
    | 1224 | 1299 | 1193 | 1326 |  648 |     0.003 |    -0.003 |     0.004 | -0.000 |  0.000 |
    | 1225 |  108 | 1194 |  649 | 1327 |     0.002 |     0.002 |     0.003 | -0.000 | -0.000 |
    | 1226 |  167 |  802 | 1343 |  166 |     0.586 |    11.351 |    16.053 | -0.029 | -0.568 |
    | 1227 |  809 | 1144 | 1380 | 1374 |     2.903 |     2.080 |     2.941 | -0.145 | -0.104 |
    | 1228 |  427 |  700 | 1311 | 1168 |     7.508 |     4.430 |     6.265 | -0.375 | -0.221 |
    | 1229 |  502 |  851 | 1394 | 1298 |     0.030 |     0.048 |     0.068 | -0.002 | -0.002 |
    | 1230 |  340 |  549 | 1388 | 1150 |     2.337 |    -1.652 |     2.336 | -0.117 |  0.083 |
    | 1231 | 1252 |  608 | 1324 | 1195 |     0.001 |     0.013 |     0.018 | -0.000 | -0.001 |
    | 1232 |  914 |  915 | 1302 | 1212 |     0.050 |     0.175 |     0.247 | -0.003 | -0.009 |
    | 1233 | 1373 |  711 | 1372 | 1241 |     3.389 |     1.068 |     1.510 | -0.169 | -0.053 |
    | 1234 |  708 |  707 | 1386 |  709 |     4.332 |     1.434 |     2.028 | -0.217 | -0.072 |
    | 1235 |  172 | 1289 | 1285 |  236 |     5.045 |   -22.344 |    31.599 | -0.252 |  1.117 |
    | 1236 |  688 |  689 | 1348 |  687 |     2.160 |     7.788 |    11.013 | -0.108 | -0.389 |
    | 1237 |  435 |  436 | 1393 |  434 |     4.085 |     4.559 |     6.447 | -0.204 | -0.228 |
    | 1238 |  435 | 1161 | 1363 |  537 |     4.797 |     3.612 |     5.108 | -0.240 | -0.181 |
    | 1239 |  310 |  291 |  171 |    6 |    45.040 |   -20.562 |    29.079 | -2.252 |  1.028 |
    | 1240 |  668 | 1247 | 1351 |  808 |     3.045 |    11.708 |    16.557 | -0.152 | -0.585 |
    | 1241 |  797 |  796 | 1150 | 1388 |     2.614 |    -1.428 |     2.019 | -0.131 |  0.071 |
    | 1242 | 1320 |  656 | 1249 | 1294 |     0.188 |    -0.068 |     0.096 | -0.009 |  0.003 |
    | 1243 | 1189 |  310 |    6 | 1276 |    19.380 |   -26.922 |    38.073 | -0.969 |  1.346 |
    | 1244 |  645 |  666 | 1295 | 1245 |     0.015 |     0.007 |     0.010 | -0.001 | -0.000 |
    | 1245 | 1258 |  365 | 1307 | 1231 |     2.769 |    -4.219 |     5.966 | -0.138 |  0.211 |
    | 1246 |  446 | 1217 | 1323 | 1145 |     1.315 |     2.376 |     3.360 | -0.066 | -0.119 |
    | 1247 |  817 |  746 | 1292 | 1342 |     9.194 |    -1.530 |     2.164 | -0.460 |  0.077 |
    | 1248 |  530 | 1339 | 1363 | 1161 |     5.706 |     3.565 |     5.041 | -0.285 | -0.178 |
    | 1249 | 1227 |  235 | 1354 | 1238 |    14.881 |    15.177 |    21.463 | -0.744 | -0.759 |
    | 1250 |  257 | 1256 |  876 | 1379 |     0.085 |     0.568 |     0.803 | -0.004 | -0.028 |
    | 1251 | 1275 | 1305 | 1267 | 1037 |     0.055 |    -0.038 |     0.054 | -0.003 |  0.002 |
    | 1252 |  855 | 1273 | 1308 |  856 |     0.029 |    -0.133 |     0.188 | -0.001 |  0.007 |
    | 1253 |  810 |  305 | 1241 | 1372 |     3.505 |     0.704 |     0.995 | -0.175 | -0.035 |
    | 1254 |  738 |  295 | 1359 | 1167 |     2.195 |   -13.399 |    18.949 | -0.110 |  0.670 |
    | 1255 | 1100 |  596 | 1319 | 1107 |     0.107 |     0.054 |     0.076 | -0.005 | -0.003 |
    | 1256 | 1291 |  775 | 1364 |  541 |     1.887 |    -2.862 |     4.048 | -0.094 |  0.143 |
    | 1257 |  688 | 1254 | 1360 |  690 |     2.947 |     6.731 |     9.519 | -0.147 | -0.337 |
    | 1258 |  208 |  249 | 1337 | 1268 |     0.010 |    -0.108 |     0.153 | -0.001 |  0.005 |
    | 1259 |  650 |  717 | 1382 | 1219 |     9.969 |     1.437 |     2.032 | -0.498 | -0.072 |
    | 1260 | 1270 |  934 | 1341 | 1300 |     0.138 |     0.775 |     1.096 | -0.007 | -0.039 |
    | 1261 |   60 |   61 | 1382 |  717 |    10.454 |     0.240 |     0.339 | -0.523 | -0.012 |
    | 1262 | 1167 | 1316 | 1289 |  738 |     4.890 |   -16.061 |    22.714 | -0.244 |  0.803 |
    | 1263 | 1037 | 1335 |  553 | 1275 |     0.067 |    -0.039 |     0.055 | -0.003 |  0.002 |
    | 1264 | 1169 |  542 | 1360 | 1254 |     3.943 |     6.698 |     9.472 | -0.197 | -0.335 |
    | 1265 | 1354 |    5 | 1390 | 1290 |    30.971 |    21.205 |    29.989 | -1.549 | -1.060 |
    | 1266 |  970 | 1278 |  630 | 1371 |     1.743 |     0.880 |     1.244 | -0.087 | -0.044 |
    | 1267 |  697 |  698 | 1365 | 1266 |     0.004 |     0.005 |     0.007 | -0.000 | -0.000 |
    | 1268 |  359 | 1209 | 1367 | 1230 |    14.079 |     2.515 |     3.557 | -0.704 | -0.126 |
    | 1269 |  146 | 1300 | 1341 |  257 |     0.082 |     0.708 |     1.002 | -0.004 | -0.035 |
    | 1270 |  716 | 1367 |  749 | 1391 |    12.703 |     1.561 |     2.208 | -0.635 | -0.078 |
    | 1271 |  784 | 1312 |  690 | 1360 |     2.798 |     6.135 |     8.676 | -0.140 | -0.307 |
    | 1272 |  777 |  791 | 1364 |  775 |     1.541 |    -2.716 |     3.842 | -0.077 |  0.136 |
    | 1273 |  291 | 1390 |    5 |  171 |    50.406 |     6.583 |     9.310 | -2.520 | -0.329 |
    | 1274 |  711 |  709 | 1386 | 1372 |     3.901 |     1.229 |     1.737 | -0.195 | -0.061 |
    | 1275 |  810 | 1372 | 1386 |  239 |     4.079 |     0.760 |     1.074 | -0.204 | -0.038 |
    | 1276 |  235 |  170 |    5 | 1354 |    10.918 |    27.953 |    39.532 | -0.546 | -1.398 |
    | 1277 |    6 |  172 |  236 | 1276 |     9.450 |   -28.379 |    40.134 | -0.472 |  1.419 |
    +------+------+------+------+------+-----------+-----------+-----------+--------+--------+
    ```

=== "Heat problem"

    ```
    -------------------------------------------------------------
    -------------- Model input ----------------------------------
    -------------------------------------------------------------

    Model parameters:

    +-------------+---------+
    | Parameter   |   Value |
    |-------------+---------|
    | w [m]       |       1 |
    | h [m]       |       1 |
    | a [m]       |     0.2 |
    | b [m]       |     0.2 |
    | x [m]       |     0.2 |
    | y [m]       |     0.2 |
    | t [m]       |       1 |
    | kx [W/mC]   |     1.7 |
    | ky [W/mC]   |     1.7 |
    +-------------+---------+

    Model boundary conditions:

    +----------+---------------+
    |   Marker |   Temperature |
    |----------+---------------|
    |       10 |            20 |
    |       20 |            20 |
    |       30 |            20 |
    |       40 |            20 |
    |      110 |           120 |
    |      120 |           120 |
    |      130 |           120 |
    |      140 |           120 |
    +----------+---------------+

    -------------------------------------------------------------
    -------------- Results --------------------------------------
    -------------------------------------------------------------

    Model info:

    +---------------------+------+
    | Max element size:   | 0.05 |
    | Dofs per node:      |    1 |
    | Element type:       |    3 |
    | Number of dofs:     |  509 |
    | Number of elements: |  461 |
    | Number of nodes:    |  509 |
    +---------------------+------+

    Summary of results:

    +--------------------------+----------+--------+
    | Max Temperature:         |  120.000 | [C]    |
    | Min Temperature:         |   20.000 | [C]    |
    | Max boundary heat power: |   87.834 | [W/m]  |
    | Min boundary heat power: |  -36.921 | [W/m]  |
    | Sum boundary heat power: |   -0.000 | [W/m]  |
    | Max resulting heat flux: | 1343.549 | [W/m2] |
    | Min resulting heat flux: |    8.913 | [W/m2] |
    +--------------------------+----------+--------+

    Results per node:

    +-----+-------+-----------+------------+---------+--------------+
    |   N |   dof |   x_coord |    y_coord |       T |   Heat power |
    |     |       |       [m] |        [m] |     [C] |        [W/m] |
    |-----+-------+-----------+------------+---------+--------------|
    |   1 |     1 |    0.0000 |     0.0000 |  20.000 |   -5.649e-01 |
    |   2 |     2 |    1.0000 |     0.0000 |  20.000 |   -3.151e-01 |
    |   3 |     3 |    1.0000 |     1.0000 |  20.000 |   -5.862e-01 |
    |   4 |     4 |    0.0000 |     1.0000 |  20.000 |   -3.824e+00 |
    |   5 |     5 |    0.2000 |     0.6000 | 120.000 |    7.486e+01 |
    |   6 |     6 |    0.4000 |     0.6000 | 120.000 |    5.507e+01 |
    |   7 |     7 |    0.4000 |     0.8000 | 120.000 |    7.509e+01 |
    |   8 |     8 |    0.2000 |     0.8000 | 120.000 |    8.783e+01 |
    |   9 |     9 |    0.0500 |     0.0000 |  20.000 |   -1.166e+00 |
    |  10 |    10 |    0.1000 |     0.0000 |  20.000 |   -2.440e+00 |
    |  11 |    11 |    0.1500 |     0.0000 |  20.000 |   -3.832e+00 |
    |  12 |    12 |    0.2000 |     0.0000 |  20.000 |   -4.333e+00 |
    |  13 |    13 |    0.2500 |     0.0000 |  20.000 |   -5.483e+00 |
    |  14 |    14 |    0.3000 |     0.0000 |  20.000 |   -5.580e+00 |
    |  15 |    15 |    0.3500 |     0.0000 |  20.000 |   -6.417e+00 |
    |  16 |    16 |    0.4000 |     0.0000 |  20.000 |   -6.130e+00 |
    |  17 |    17 |    0.4500 |     0.0000 |  20.000 |   -6.537e+00 |
    |  18 |    18 |    0.5000 |     0.0000 |  20.000 |   -5.825e+00 |
    |  19 |    19 |    0.5500 |     0.0000 |  20.000 |   -5.979e+00 |
    |  20 |    20 |    0.6000 |     0.0000 |  20.000 |   -5.068e+00 |
    |  21 |    21 |    0.6500 |     0.0000 |  20.000 |   -4.878e+00 |
    |  22 |    22 |    0.7000 |     0.0000 |  20.000 |   -3.945e+00 |
    |  23 |    23 |    0.7500 |     0.0000 |  20.000 |   -3.698e+00 |
    |  24 |    24 |    0.8000 |     0.0000 |  20.000 |   -2.679e+00 |
    |  25 |    25 |    0.8500 |     0.0000 |  20.000 |   -2.179e+00 |
    |  26 |    26 |    0.9000 |     0.0000 |  20.000 |   -1.337e+00 |
    |  27 |    27 |    0.9500 |     0.0000 |  20.000 |   -7.094e-01 |
    |  28 |    28 |    1.0000 |     0.0500 |  20.000 |   -7.116e-01 |
    |  29 |    29 |    1.0000 |     0.1000 |  20.000 |   -1.339e+00 |
    |  30 |    30 |    1.0000 |     0.1500 |  20.000 |   -2.163e+00 |
    |  31 |    31 |    1.0000 |     0.2000 |  20.000 |   -2.703e+00 |
    |  32 |    32 |    1.0000 |     0.2500 |  20.000 |   -3.565e+00 |
    |  33 |    33 |    1.0000 |     0.3000 |  20.000 |   -4.009e+00 |
    |  34 |    34 |    1.0000 |     0.3500 |  20.000 |   -4.836e+00 |
    |  35 |    35 |    1.0000 |     0.4000 |  20.000 |   -5.141e+00 |
    |  36 |    36 |    1.0000 |     0.4500 |  20.000 |   -5.862e+00 |
    |  37 |    37 |    1.0000 |     0.5000 |  20.000 |   -5.928e+00 |
    |  38 |    38 |    1.0000 |     0.5500 |  20.000 |   -6.430e+00 |
    |  39 |    39 |    1.0000 |     0.6000 |  20.000 |   -6.175e+00 |
    |  40 |    40 |    1.0000 |     0.6500 |  20.000 |   -6.350e+00 |
    |  41 |    41 |    1.0000 |     0.7000 |  20.000 |   -5.678e+00 |
    |  42 |    42 |    1.0000 |     0.7500 |  20.000 |   -5.418e+00 |
    |  43 |    43 |    1.0000 |     0.8000 |  20.000 |   -4.347e+00 |
    |  44 |    44 |    1.0000 |     0.8500 |  20.000 |   -3.620e+00 |
    |  45 |    45 |    1.0000 |     0.9000 |  20.000 |   -2.385e+00 |
    |  46 |    46 |    1.0000 |     0.9500 |  20.000 |   -1.258e+00 |
    |  47 |    47 |    0.9500 |     1.0000 |  20.000 |   -1.309e+00 |
    |  48 |    48 |    0.9000 |     1.0000 |  20.000 |   -2.537e+00 |
    |  49 |    49 |    0.8500 |     1.0000 |  20.000 |   -4.261e+00 |
    |  50 |    50 |    0.8000 |     1.0000 |  20.000 |   -5.701e+00 |
    |  51 |    51 |    0.7500 |     1.0000 |  20.000 |   -8.022e+00 |
    |  52 |    52 |    0.7000 |     1.0000 |  20.000 |   -9.731e+00 |
    |  53 |    53 |    0.6500 |     1.0000 |  20.000 |   -1.332e+01 |
    |  54 |    54 |    0.6000 |     1.0000 |  20.000 |   -1.569e+01 |
    |  55 |    55 |    0.5500 |     1.0000 |  20.000 |   -2.124e+01 |
    |  56 |    56 |    0.5000 |     1.0000 |  20.000 |   -2.385e+01 |
    |  57 |    57 |    0.4500 |     1.0000 |  20.000 |   -3.045e+01 |
    |  58 |    58 |    0.4000 |     1.0000 |  20.000 |   -3.198e+01 |
    |  59 |    59 |    0.3500 |     1.0000 |  20.000 |   -3.692e+01 |
    |  60 |    60 |    0.3000 |     1.0000 |  20.000 |   -3.516e+01 |
    |  61 |    61 |    0.2500 |     1.0000 |  20.000 |   -3.610e+01 |
    |  62 |    62 |    0.2000 |     1.0000 |  20.000 |   -2.855e+01 |
    |  63 |    63 |    0.1500 |     1.0000 |  20.000 |   -2.474e+01 |
    |  64 |    64 |    0.1000 |     1.0000 |  20.000 |   -1.555e+01 |
    |  65 |    65 |    0.0500 |     1.0000 |  20.000 |   -8.095e+00 |
    |  66 |    66 |    0.0000 |     0.9500 |  20.000 |   -8.615e+00 |
    |  67 |    67 |    0.0000 |     0.9000 |  20.000 |   -1.577e+01 |
    |  68 |    68 |    0.0000 |     0.8500 |  20.000 |   -2.515e+01 |
    |  69 |    69 |    0.0000 |     0.8000 |  20.000 |   -2.939e+01 |
    |  70 |    70 |    0.0000 |     0.7500 |  20.000 |   -3.520e+01 |
    |  71 |    71 |    0.0000 |     0.7000 |  20.000 |   -3.486e+01 |
    |  72 |    72 |    0.0000 |     0.6500 |  20.000 |   -3.677e+01 |
    |  73 |    73 |    0.0000 |     0.6000 |  20.000 |   -3.197e+01 |
    |  74 |    74 |    0.0000 |     0.5500 |  20.000 |   -3.102e+01 |
    |  75 |    75 |    0.0000 |     0.5000 |  20.000 |   -2.397e+01 |
    |  76 |    76 |    0.0000 |     0.4500 |  20.000 |   -2.079e+01 |
    |  77 |    77 |    0.0000 |     0.4000 |  20.000 |   -1.539e+01 |
    |  78 |    78 |    0.0000 |     0.3500 |  20.000 |   -1.371e+01 |
    |  79 |    79 |    0.0000 |     0.3000 |  20.000 |   -9.710e+00 |
    |  80 |    80 |    0.0000 |     0.2500 |  20.000 |   -7.943e+00 |
    |  81 |    81 |    0.0000 |     0.2000 |  20.000 |   -5.548e+00 |
    |  82 |    82 |    0.0000 |     0.1500 |  20.000 |   -4.097e+00 |
    |  83 |    83 |    0.0000 |     0.1000 |  20.000 |   -2.546e+00 |
    |  84 |    84 |    0.0000 |     0.0500 |  20.000 |   -1.331e+00 |
    |  85 |    85 |    0.2500 |     0.6000 | 120.000 |    4.660e+01 |
    |  86 |    86 |    0.3000 |     0.6000 | 120.000 |    3.130e+01 |
    |  87 |    87 |    0.3500 |     0.6000 | 120.000 |    3.997e+01 |
    |  88 |    88 |    0.4000 |     0.6500 | 120.000 |    4.053e+01 |
    |  89 |    89 |    0.4000 |     0.7000 | 120.000 |    3.102e+01 |
    |  90 |    90 |    0.4000 |     0.7500 | 120.000 |    4.697e+01 |
    |  91 |    91 |    0.3500 |     0.8000 | 120.000 |    5.966e+01 |
    |  92 |    92 |    0.3000 |     0.8000 | 120.000 |    4.756e+01 |
    |  93 |    93 |    0.2500 |     0.8000 | 120.000 |    6.157e+01 |
    |  94 |    94 |    0.2000 |     0.7500 | 120.000 |    6.396e+01 |
    |  95 |    95 |    0.2000 |     0.7000 | 120.000 |    4.746e+01 |
    |  96 |    96 |    0.2000 |     0.6500 | 120.000 |    5.898e+01 |
    |  97 |    97 |    0.6081 |     0.0451 |  22.908 |   -7.105e-15 |
    |  98 |    98 |    0.9583 |     0.3861 |  22.609 |   -7.105e-15 |
    |  99 |    99 |    0.9567 |     0.6815 |  23.187 |    1.776e-14 |
    | 100 |   100 |    0.3052 |     0.5559 | 102.464 |    2.842e-14 |
    | 101 |   101 |    0.4191 |     0.0393 |  23.098 |    2.665e-14 |
    | 102 |   102 |    0.4433 |     0.6447 | 105.262 |   -3.197e-14 |
    | 103 |   103 |    0.4433 |     0.5940 |  93.323 |   -2.842e-14 |
    | 104 |   104 |    0.0490 |     0.2801 |  24.737 |   -1.421e-14 |
    | 105 |   105 |    0.9584 |     0.5367 |  22.992 |    0.000e+00 |
    | 106 |   106 |    0.4866 |     0.5906 |  86.277 |    1.421e-14 |
    | 107 |   107 |    0.4867 |     0.5397 |  75.106 |    0.000e+00 |
    | 108 |   108 |    0.7055 |     0.9521 |  25.558 |    2.309e-14 |
    | 109 |   109 |    0.5299 |     0.5893 |  72.169 |   -2.842e-14 |
    | 110 |   110 |    0.5300 |     0.5387 |  71.300 |   -8.527e-14 |
    | 111 |   111 |    0.2668 |     0.0471 |  22.995 |   -1.066e-14 |
    | 112 |   112 |    0.7657 |     0.0483 |  21.773 |   -7.105e-15 |
    | 113 |   113 |    0.4867 |     0.4893 |  71.751 |    0.000e+00 |
    | 114 |   114 |    0.5300 |     0.4883 |  62.319 |   -1.421e-14 |
    | 115 |   115 |    0.4866 |     0.4391 |  60.592 |    0.000e+00 |
    | 116 |   116 |    0.5299 |     0.4379 |  59.604 |    0.000e+00 |
    | 117 |   117 |    0.5733 |     0.4377 |  52.649 |   -1.421e-14 |
    | 118 |   118 |    0.5300 |     0.6400 |  78.625 |   -2.842e-14 |
    | 119 |   119 |    0.5733 |     0.5887 |  67.153 |    5.684e-14 |
    | 120 |   120 |    0.5733 |     0.3871 |  50.351 |    7.105e-15 |
    | 121 |   121 |    0.4859 |     0.3890 |  56.922 |    0.000e+00 |
    | 122 |   122 |    0.6174 |     0.4367 |  50.624 |    0.000e+00 |
    | 123 |   123 |    0.6172 |     0.3855 |  44.847 |    1.421e-14 |
    | 124 |   124 |    0.5733 |     0.6392 |  65.638 |   -1.137e-13 |
    | 125 |   125 |    0.6165 |     0.6387 |  60.512 |   -2.842e-14 |
    | 126 |   126 |    0.9584 |     0.2364 |  21.554 |    7.105e-15 |
    | 127 |   127 |    0.0435 |     0.4475 |  29.556 |   -7.105e-15 |
    | 128 |   128 |    0.5600 |     0.9530 |  30.242 |    7.105e-15 |
    | 129 |   129 |    0.6165 |     0.6900 |  57.314 |    1.421e-14 |
    | 130 |   130 |    0.6598 |     0.6379 |  51.496 |   -1.421e-14 |
    | 131 |   131 |    0.6622 |     0.3802 |  42.819 |    2.842e-14 |
    | 132 |   132 |    0.6607 |     0.3284 |  38.296 |    0.000e+00 |
    | 133 |   133 |    0.6169 |     0.3342 |  42.788 |    0.000e+00 |
    | 134 |   134 |    0.9563 |     0.8114 |  22.311 |   -7.105e-15 |
    | 135 |   135 |    0.6600 |     0.2778 |  36.479 |    2.842e-14 |
    | 136 |   136 |    0.6597 |     0.6890 |  52.644 |    5.684e-14 |
    | 137 |   137 |    0.7044 |     0.3171 |  36.373 |   -1.421e-14 |
    | 138 |   138 |    0.6166 |     0.2835 |  37.573 |    2.842e-14 |
    | 139 |   139 |    0.7032 |     0.6863 |  45.147 |    2.842e-14 |
    | 140 |   140 |    0.7031 |     0.6353 |  47.492 |    2.842e-14 |
    | 141 |   141 |    0.7035 |     0.7380 |  44.624 |    0.000e+00 |
    | 142 |   142 |    0.7034 |     0.2672 |  32.766 |    0.000e+00 |
    | 143 |   143 |    0.6164 |     0.7428 |  57.025 |    8.527e-14 |
    | 144 |   144 |    0.7474 |     0.3041 |  32.352 |    4.263e-14 |
    | 145 |   145 |    0.4433 |     0.4442 |  68.470 |    4.263e-14 |
    | 146 |   146 |    0.4419 |     0.3950 |  57.338 |    2.842e-14 |
    | 147 |   147 |    0.3980 |     0.4052 |  63.813 |    0.000e+00 |
    | 148 |   148 |    0.3914 |     0.3572 |  53.482 |    5.684e-14 |
    | 149 |   149 |    0.3488 |     0.3732 |  58.752 |    2.842e-14 |
    | 150 |   150 |    0.3335 |     0.3347 |  50.163 |   -6.395e-14 |
    | 151 |   151 |    0.3055 |     0.3532 |  54.488 |    4.263e-14 |
    | 152 |   152 |    0.3740 |     0.3109 |  49.401 |    7.105e-14 |
    | 153 |   153 |    0.3027 |     0.3019 |  46.944 |    0.000e+00 |
    | 154 |   154 |    0.2673 |     0.3411 |  48.757 |    5.684e-14 |
    | 155 |   155 |    0.2620 |     0.2725 |  40.436 |   -2.842e-14 |
    | 156 |   156 |    0.2687 |     0.3865 |  58.123 |    1.563e-13 |
    | 157 |   157 |    0.7466 |     0.2550 |  31.159 |    5.684e-14 |
    | 158 |   158 |    0.8068 |     0.9547 |  23.015 |    7.105e-15 |
    | 159 |   159 |    0.0432 |     0.1759 |  22.490 |   -1.421e-14 |
    | 160 |   160 |    0.5129 |     0.0441 |  23.297 |   -2.132e-14 |
    | 161 |   161 |    0.1548 |     0.0462 |  21.994 |   -1.421e-14 |
    | 162 |   162 |    0.7503 |     0.3519 |  35.179 |    4.974e-14 |
    | 163 |   163 |    0.2901 |     0.2339 |  38.888 |    0.000e+00 |
    | 164 |   164 |    0.7465 |     0.6286 |  41.026 |   -4.263e-14 |
    | 165 |   165 |    0.7464 |     0.5778 |  42.189 |   -7.816e-14 |
    | 166 |   166 |    0.8597 |     0.0436 |  20.964 |    7.105e-15 |
    | 167 |   167 |    0.2305 |     0.3554 |  50.465 |   -2.132e-14 |
    | 168 |   168 |    0.2286 |     0.3857 |  52.671 |    4.263e-14 |
    | 169 |   169 |    0.2479 |     0.2114 |  34.323 |    1.421e-14 |
    | 170 |   170 |    0.7028 |     0.2181 |  31.174 |   -3.553e-14 |
    | 171 |   171 |    0.3059 |     0.1899 |  33.982 |   -2.842e-14 |
    | 172 |   172 |    0.2201 |     0.2434 |  36.864 |   -2.665e-14 |
    | 173 |   173 |    0.4436 |     0.7903 |  89.638 |    1.421e-14 |
    | 174 |   174 |    0.4416 |     0.8224 |  89.698 |    0.000e+00 |
    | 175 |   175 |    0.0572 |     0.5293 |  38.166 |    0.000e+00 |
    | 176 |   176 |    0.1247 |     0.9615 |  29.288 |    7.105e-15 |
    | 177 |   177 |    0.7898 |     0.6156 |  37.752 |    4.263e-14 |
    | 178 |   178 |    0.7912 |     0.6659 |  36.201 |   -1.421e-14 |
    | 179 |   179 |    0.7895 |     0.5650 |  36.617 |    2.132e-14 |
    | 180 |   180 |    0.7897 |     0.2930 |  30.568 |   -2.842e-14 |
    | 181 |   181 |    0.7896 |     0.2438 |  28.178 |    0.000e+00 |
    | 182 |   182 |    0.7032 |     0.5840 |  45.664 |    9.592e-14 |
    | 183 |   183 |    0.2199 |     0.1934 |  32.819 |    2.132e-14 |
    | 184 |   184 |    0.5732 |     0.2859 |  41.241 |   -2.842e-14 |
    | 185 |   185 |    0.5731 |     0.2358 |  35.739 |    7.105e-15 |
    | 186 |   186 |    0.4845 |     0.7920 |  82.404 |   -1.421e-14 |
    | 187 |   187 |    0.2245 |     0.8461 |  84.006 |   -8.527e-14 |
    | 188 |   188 |    0.1617 |     0.6473 |  98.822 |    2.842e-14 |
    | 189 |   189 |    0.3162 |     0.9599 |  38.285 |    0.000e+00 |
    | 190 |   190 |    0.0473 |     0.8147 |  37.054 |   -7.105e-15 |
    | 191 |   191 |    0.0428 |     0.7026 |  39.518 |    8.882e-15 |
    | 192 |   192 |    0.8328 |     0.5988 |  32.877 |    1.421e-14 |
    | 193 |   193 |    0.2091 |     0.5575 |  90.562 |   -1.421e-14 |
    | 194 |   194 |    0.1871 |     0.3797 |  49.743 |   -5.684e-14 |
    | 195 |   195 |    0.1805 |     0.4234 |  53.137 |   -2.842e-14 |
    | 196 |   196 |    0.1816 |     0.2058 |  31.119 |    0.000e+00 |
    | 197 |   197 |    0.8323 |     0.5489 |  33.541 |    3.020e-14 |
    | 198 |   198 |    0.7886 |     0.5157 |  37.078 |    3.553e-14 |
    | 199 |   199 |    0.8312 |     0.4990 |  32.296 |    0.000e+00 |
    | 200 |   200 |    0.2681 |     0.4277 |  63.220 |   -1.421e-14 |
    | 201 |   201 |    0.4701 |     0.8478 |  67.177 |    5.684e-14 |
    | 202 |   202 |    0.1834 |     0.1546 |  28.561 |    4.974e-14 |
    | 203 |   203 |    0.1623 |     0.5582 |  83.740 |    0.000e+00 |
    | 204 |   204 |    0.1876 |     0.8572 |  78.157 |    0.000e+00 |
    | 205 |   205 |    0.9573 |     0.1393 |  20.944 |    7.105e-15 |
    | 206 |   206 |    0.4206 |     0.8687 |  74.501 |    0.000e+00 |
    | 207 |   207 |    0.6172 |     0.4874 |  51.986 |   -5.684e-14 |
    | 208 |   208 |    0.5292 |     0.2860 |  41.329 |    2.842e-14 |
    | 209 |   209 |    0.5293 |     0.2352 |  38.424 |   -5.684e-14 |
    | 210 |   210 |    0.5727 |     0.1862 |  33.229 |    2.842e-14 |
    | 211 |   211 |    0.5292 |     0.1850 |  32.945 |    4.974e-14 |
    | 212 |   212 |    0.4854 |     0.1808 |  34.462 |   -4.263e-14 |
    | 213 |   213 |    0.7853 |     0.4687 |  35.437 |    0.000e+00 |
    | 214 |   214 |    0.8276 |     0.4490 |  32.564 |    0.000e+00 |
    | 215 |   215 |    0.5301 |     0.6909 |  74.853 |   -5.684e-14 |
    | 216 |   216 |    0.3995 |     0.4528 |  69.207 |    4.263e-14 |
    | 217 |   217 |    0.1523 |     0.7667 |  91.699 |    5.684e-14 |
    | 218 |   218 |    0.8323 |     0.2345 |  26.726 |   -2.132e-14 |
    | 219 |   219 |    0.8315 |     0.1852 |  24.958 |   -1.421e-14 |
    | 220 |   220 |    0.8318 |     0.2842 |  27.567 |    2.132e-14 |
    | 221 |   221 |    0.7887 |     0.1945 |  27.053 |    5.684e-14 |
    | 222 |   222 |    0.4835 |     0.1321 |  29.487 |    1.421e-14 |
    | 223 |   223 |    0.4414 |     0.1736 |  33.164 |    5.684e-14 |
    | 224 |   224 |    0.5713 |     0.1376 |  28.820 |    2.132e-14 |
    | 225 |   225 |    0.2248 |     0.1618 |  29.814 |    7.105e-15 |
    | 226 |   226 |    0.8286 |     0.1375 |  24.034 |    7.105e-15 |
    | 227 |   227 |    0.2040 |     0.8838 |  62.882 |   -1.421e-14 |
    | 228 |   228 |    0.0398 |     0.6087 |  36.647 |    1.421e-14 |
    | 229 |   229 |    0.3171 |     0.8403 |  95.851 |    5.684e-14 |
    | 230 |   230 |    0.0436 |     0.0886 |  21.258 |    1.421e-14 |
    | 231 |   231 |    0.9543 |     0.9057 |  21.305 |    7.105e-15 |
    | 232 |   232 |    0.1860 |     0.2768 |  36.666 |   -1.421e-14 |
    | 233 |   233 |    0.6160 |     0.1848 |  31.010 |    4.263e-14 |
    | 234 |   234 |    0.6166 |     0.5883 |  57.097 |   -1.421e-14 |
    | 235 |   235 |    0.7459 |     0.4812 |  40.198 |   -2.842e-14 |
    | 236 |   236 |    0.7472 |     0.4471 |  37.956 |   -4.263e-14 |
    | 237 |   237 |    0.3506 |     0.2027 |  36.867 |    2.842e-14 |
    | 238 |   238 |    0.3555 |     0.1556 |  31.654 |    2.842e-14 |
    | 239 |   239 |    0.3130 |     0.1443 |  31.107 |   -2.842e-14 |
    | 240 |   240 |    0.3585 |     0.1114 |  28.792 |    2.487e-14 |
    | 241 |   241 |    0.5278 |     0.7953 |  65.757 |   -1.421e-14 |
    | 242 |   242 |    0.5724 |     0.7970 |  59.555 |    5.684e-14 |
    | 243 |   243 |    0.5705 |     0.8513 |  49.062 |    7.105e-14 |
    | 244 |   244 |    0.6152 |     0.8515 |  45.283 |   -8.527e-14 |
    | 245 |   245 |    0.6599 |     0.8501 |  39.215 |   -5.684e-14 |
    | 246 |   246 |    0.4713 |     0.9060 |  51.889 |    1.421e-14 |
    | 247 |   247 |    0.2347 |     0.8834 |  71.874 |    2.132e-14 |
    | 248 |   248 |    0.1414 |     0.1922 |  28.856 |    6.395e-14 |
    | 249 |   249 |    0.1302 |     0.1504 |  25.896 |    2.842e-14 |
    | 250 |   250 |    0.1356 |     0.4163 |  47.175 |    1.421e-14 |
    | 251 |   251 |    0.1424 |     0.3714 |  41.187 |   -1.563e-13 |
    | 252 |   252 |    0.8738 |     0.4852 |  29.470 |   -5.684e-14 |
    | 253 |   253 |    0.8742 |     0.2781 |  25.926 |   -4.263e-14 |
    | 254 |   254 |    0.8750 |     0.5855 |  29.969 |   -2.842e-14 |
    | 255 |   255 |    0.8737 |     0.1803 |  23.890 |   -7.105e-14 |
    | 256 |   256 |    0.8765 |     0.6348 |  29.154 |   -5.684e-14 |
    | 257 |   257 |    0.8720 |     0.4349 |  28.346 |   -7.105e-15 |
    | 258 |   258 |    0.8179 |     0.3930 |  31.139 |   -4.263e-14 |
    | 259 |   259 |    0.8724 |     0.3295 |  26.590 |   -3.020e-14 |
    | 260 |   260 |    0.2145 |     0.5144 |  82.199 |    2.842e-14 |
    | 261 |   261 |    0.2621 |     0.5139 |  84.491 |    0.000e+00 |
    | 262 |   262 |    0.3990 |     0.5486 |  91.783 |    5.684e-14 |
    | 263 |   263 |    0.0462 |     0.9099 |  28.624 |    0.000e+00 |
    | 264 |   264 |    0.6592 |     0.9028 |  33.970 |   -1.776e-14 |
    | 265 |   265 |    0.7056 |     0.8466 |  36.706 |    7.105e-14 |
    | 266 |   266 |    0.7517 |     0.8396 |  32.719 |   -3.553e-15 |
    | 267 |   267 |    0.7503 |     0.7854 |  37.081 |    1.421e-14 |
    | 268 |   268 |    0.7937 |     0.8326 |  31.108 |   -8.171e-14 |
    | 269 |   269 |    0.8047 |     0.7767 |  32.193 |   -6.040e-14 |
    | 270 |   270 |    0.8344 |     0.7990 |  29.719 |    2.842e-14 |
    | 271 |   271 |    0.8309 |     0.8300 |  28.396 |    1.421e-14 |
    | 272 |   272 |    0.8515 |     0.7433 |  30.351 |    1.421e-14 |
    | 273 |   273 |    0.8649 |     0.7844 |  27.849 |   -2.487e-14 |
    | 274 |   274 |    0.8914 |     0.7212 |  27.202 |   -7.105e-15 |
    | 275 |   275 |    0.5293 |     0.3368 |  48.230 |    0.000e+00 |
    | 276 |   276 |    0.7058 |     0.4842 |  42.769 |   -2.842e-14 |
    | 277 |   277 |    0.4825 |     0.2843 |  44.617 |    0.000e+00 |
    | 278 |   278 |    0.4381 |     0.2230 |  38.989 |   -2.842e-14 |
    | 279 |   279 |    0.8713 |     0.1338 |  22.733 |   -2.842e-14 |
    | 280 |   280 |    0.4400 |     0.1259 |  30.084 |   -7.105e-15 |
    | 281 |   281 |    0.7857 |     0.1458 |  24.969 |    2.842e-14 |
    | 282 |   282 |    0.7436 |     0.1572 |  26.904 |    4.263e-14 |
    | 283 |   283 |    0.4150 |     0.9564 |  37.461 |   -5.684e-14 |
    | 284 |   284 |    0.6151 |     0.1368 |  28.793 |    0.000e+00 |
    | 285 |   285 |    0.6581 |     0.1333 |  27.034 |   -2.132e-14 |
    | 286 |   286 |    0.9174 |     0.6297 |  26.406 |    2.842e-14 |
    | 287 |   287 |    0.9158 |     0.7652 |  25.338 |    7.105e-14 |
    | 288 |   288 |    0.9163 |     0.4813 |  25.781 |   -2.842e-14 |
    | 289 |   289 |    0.9160 |     0.3290 |  24.607 |    2.753e-14 |
    | 290 |   290 |    0.1473 |     0.8809 |  53.192 |    1.421e-14 |
    | 291 |   291 |    0.1329 |     0.8283 |  67.461 |    4.263e-14 |
    | 292 |   292 |    0.0981 |     0.3632 |  35.627 |   -1.421e-14 |
    | 293 |   293 |    0.2317 |     0.9189 |  51.761 |    0.000e+00 |
    | 294 |   294 |    0.5673 |     0.0906 |  26.328 |   -1.421e-14 |
    | 295 |   295 |    0.6573 |     0.0898 |  25.178 |   -2.842e-14 |
    | 296 |   296 |    0.1052 |     0.3307 |  33.376 |    7.105e-15 |
    | 297 |   297 |    0.7363 |     0.1092 |  24.563 |    4.619e-14 |
    | 298 |   298 |    0.4768 |     0.0860 |  26.673 |    9.948e-14 |
    | 299 |   299 |    0.0928 |     0.8732 |  44.202 |   -3.553e-14 |
    | 300 |   300 |    0.3170 |     0.1005 |  27.031 |    0.000e+00 |
    | 301 |   301 |    0.3612 |     0.0804 |  25.787 |    3.553e-14 |
    | 302 |   302 |    0.1656 |     0.5132 |  67.038 |   -2.842e-14 |
    | 303 |   303 |    0.1158 |     0.5612 |  60.377 |    7.105e-15 |
    | 304 |   304 |    0.7652 |     0.8942 |  28.719 |    5.684e-14 |
    | 305 |   305 |    0.2198 |     0.4706 |  67.565 |   -2.842e-14 |
    | 306 |   306 |    0.1083 |     0.5090 |  54.141 |    1.137e-13 |
    | 307 |   307 |    0.5666 |     0.9034 |  41.616 |    2.132e-14 |
    | 308 |   308 |    0.2770 |     0.9209 |  56.304 |   -2.842e-14 |
    | 309 |   309 |    0.0499 |     0.3454 |  27.022 |    8.527e-14 |
    | 310 |   310 |    0.0913 |     0.1842 |  25.736 |   -1.421e-14 |
    | 311 |   311 |    0.0839 |     0.1293 |  23.403 |    8.527e-14 |
    | 312 |   312 |    0.1158 |     0.2154 |  27.892 |    0.000e+00 |
    | 313 |   313 |    0.1459 |     0.0944 |  24.051 |   -2.842e-14 |
    | 314 |   314 |    0.0942 |     0.0771 |  22.145 |   -9.548e-15 |
    | 315 |   315 |    0.4433 |     0.7445 | 103.725 |    0.000e+00 |
    | 316 |   316 |    0.4434 |     0.6952 | 102.541 |    5.684e-14 |
    | 317 |   317 |    0.2566 |     0.5569 | 103.593 |    0.000e+00 |
    | 318 |   318 |    0.3089 |     0.5117 |  90.493 |   -1.563e-13 |
    | 319 |   319 |    0.3547 |     0.5070 |  84.291 |    1.421e-14 |
    | 320 |   320 |    0.3108 |     0.4675 |  74.207 |   -1.137e-13 |
    | 321 |   321 |    0.5114 |     0.9531 |  33.657 |   -2.132e-14 |
    | 322 |   322 |    0.0402 |     0.5767 |  34.217 |   -1.421e-14 |
    | 323 |   323 |    0.0789 |     0.6096 |  50.900 |    0.000e+00 |
    | 324 |   324 |    0.0815 |     0.6544 |  57.058 |   -9.948e-14 |
    | 325 |   325 |    0.1592 |     0.7242 |  96.132 |   -2.842e-14 |
    | 326 |   326 |    0.1116 |     0.7479 |  67.950 |   -2.842e-14 |
    | 327 |   327 |    0.1299 |     0.7010 |  82.234 |   -1.279e-13 |
    | 328 |   328 |    0.0407 |     0.6539 |  36.749 |   -1.421e-14 |
    | 329 |   329 |    0.8560 |     0.9560 |  21.915 |   -2.842e-14 |
    | 330 |   330 |    0.8616 |     0.9120 |  23.887 |   -2.842e-14 |
    | 331 |   331 |    0.9081 |     0.9097 |  22.412 |    1.243e-14 |
    | 332 |   332 |    0.8155 |     0.9095 |  25.163 |   -4.263e-14 |
    | 333 |   333 |    0.7565 |     0.9516 |  24.072 |   -5.684e-14 |
    | 334 |   334 |    0.0458 |     0.4897 |  33.195 |   -7.105e-14 |
    | 335 |   335 |    0.0847 |     0.4513 |  40.730 |    0.000e+00 |
    | 336 |   336 |    0.4626 |     0.9542 |  34.619 |    1.137e-13 |
    | 337 |   337 |    0.0547 |     0.7580 |  41.725 |    4.263e-14 |
    | 338 |   338 |    0.0982 |     0.7849 |  61.134 |    4.263e-14 |
    | 339 |   339 |    0.3644 |     0.8374 |  99.506 |   -1.705e-13 |
    | 340 |   340 |    0.3259 |     0.8801 |  77.568 |    7.105e-14 |
    | 341 |   341 |    0.6559 |     0.9524 |  26.594 |    2.842e-14 |
    | 342 |   342 |    0.0462 |     0.8625 |  31.646 |    4.263e-14 |
    | 343 |   343 |    0.9553 |     0.8578 |  21.772 |   -5.684e-14 |
    | 344 |   344 |    0.9105 |     0.8640 |  23.627 |    4.974e-14 |
    | 345 |   345 |    0.4432 |     0.5437 |  89.646 |   -7.461e-14 |
    | 346 |   346 |    0.3993 |     0.5003 |  84.051 |   -2.665e-14 |
    | 347 |   347 |    0.6073 |     0.9525 |  28.993 |   -3.553e-15 |
    | 348 |   348 |    0.4650 |     0.0423 |  22.988 |    3.553e-15 |
    | 349 |   349 |    0.4432 |     0.4941 |  73.535 |   -2.842e-14 |
    | 350 |   350 |    0.3081 |     0.3832 |  57.032 |    5.684e-14 |
    | 351 |   351 |    0.3536 |     0.4162 |  63.515 |   -5.684e-14 |
    | 352 |   352 |    0.1476 |     0.2461 |  32.603 |   -1.421e-14 |
    | 353 |   353 |    0.1130 |     0.2784 |  30.774 |    0.000e+00 |
    | 354 |   354 |    0.7106 |     0.0586 |  22.880 |   -2.842e-14 |
    | 355 |   355 |    0.1626 |     0.6800 |  97.835 |   -1.137e-13 |
    | 356 |   356 |    0.1746 |     0.9470 |  36.355 |   -4.263e-14 |
    | 357 |   357 |    0.8125 |     0.0454 |  21.455 |    1.421e-14 |
    | 358 |   358 |    0.4867 |     0.6922 |  91.404 |   -1.421e-14 |
    | 359 |   359 |    0.8347 |     0.6493 |  33.274 |    1.421e-14 |
    | 360 |   360 |    0.9063 |     0.0438 |  20.709 |   -3.553e-15 |
    | 361 |   361 |    0.0555 |     0.2138 |  23.988 |   -1.279e-13 |
    | 362 |   362 |    0.2173 |     0.0505 |  23.005 |    4.441e-14 |
    | 363 |   363 |    0.1927 |     0.1069 |  25.597 |   -4.263e-14 |
    | 364 |   364 |    0.2427 |     0.0891 |  25.336 |   -3.553e-14 |
    | 365 |   365 |    0.7455 |     0.2062 |  28.356 |   -7.105e-15 |
    | 366 |   366 |    0.0425 |     0.1347 |  21.721 |    3.553e-14 |
    | 367 |   367 |    0.9581 |     0.1881 |  21.345 |    7.105e-15 |
    | 368 |   368 |    0.6596 |     0.2277 |  32.337 |    1.421e-14 |
    | 369 |   369 |    0.4835 |     0.3377 |  48.216 |    2.842e-14 |
    | 370 |   370 |    0.5733 |     0.5385 |  61.035 |   -3.553e-14 |
    | 371 |   371 |    0.7473 |     0.6784 |  41.564 |    7.105e-14 |
    | 372 |   372 |    0.5235 |     0.8498 |  59.368 |   -1.137e-13 |
    | 373 |   373 |    0.7171 |     0.4259 |  40.800 |   -2.842e-14 |
    | 374 |   374 |    0.5598 |     0.0449 |  22.868 |    1.421e-14 |
    | 375 |   375 |    0.6553 |     0.0469 |  22.497 |    6.395e-14 |
    | 376 |   376 |    0.2276 |     0.3063 |  43.650 |   -8.527e-14 |
    | 377 |   377 |    0.3654 |     0.9584 |  36.851 |    6.040e-14 |
    | 378 |   378 |    0.3739 |     0.9168 |  56.945 |   -8.527e-14 |
    | 379 |   379 |    0.6163 |     0.2332 |  35.354 |   -2.132e-14 |
    | 380 |   380 |    0.5295 |     0.3876 |  51.207 |    0.000e+00 |
    | 381 |   381 |    0.6599 |     0.5870 |  53.033 |   -7.105e-14 |
    | 382 |   382 |    0.6605 |     0.5366 |  49.488 |   -2.842e-14 |
    | 383 |   383 |    0.4867 |     0.6419 |  86.078 |    1.066e-13 |
    | 384 |   384 |    0.7750 |     0.4281 |  36.133 |   -2.842e-14 |
    | 385 |   385 |    0.6598 |     0.7951 |  46.203 |   -1.421e-14 |
    | 386 |   386 |    0.5732 |     0.3361 |  43.727 |   -2.842e-14 |
    | 387 |   387 |    0.5267 |     0.1360 |  30.195 |    0.000e+00 |
    | 388 |   388 |    0.6169 |     0.5378 |  57.390 |    4.974e-14 |
    | 389 |   389 |    0.8220 |     0.0915 |  22.579 |    2.842e-14 |
    | 390 |   390 |    0.8745 |     0.5367 |  29.248 |    4.263e-14 |
    | 391 |   391 |    0.4834 |     0.2313 |  37.644 |   -1.421e-14 |
    | 392 |   392 |    0.5733 |     0.6906 |  68.350 |   -5.684e-14 |
    | 393 |   393 |    0.6635 |     0.4325 |  44.188 |    8.527e-14 |
    | 394 |   394 |    0.7080 |     0.3677 |  37.548 |   -2.842e-14 |
    | 395 |   395 |    0.7602 |     0.3925 |  34.993 |    7.105e-15 |
    | 396 |   396 |    0.8744 |     0.2293 |  24.567 |   -3.553e-15 |
    | 397 |   397 |    0.2005 |     0.3409 |  44.119 |    2.842e-14 |
    | 398 |   398 |    0.2635 |     0.1737 |  32.644 |    2.842e-14 |
    | 399 |   399 |    0.7895 |     0.3400 |  31.339 |    1.066e-14 |
    | 400 |   400 |    0.5735 |     0.4880 |  59.796 |   -7.105e-14 |
    | 401 |   401 |    0.7461 |     0.5273 |  40.170 |    0.000e+00 |
    | 402 |   402 |    0.3434 |     0.2593 |  41.167 |    2.132e-14 |
    | 403 |   403 |    0.6591 |     0.1791 |  30.409 |    2.132e-14 |
    | 404 |   404 |    0.4361 |     0.3439 |  53.052 |    1.421e-14 |
    | 405 |   405 |    0.4234 |     0.2812 |  43.549 |   -5.684e-14 |
    | 406 |   406 |    0.7016 |     0.1697 |  27.972 |   -2.132e-14 |
    | 407 |   407 |    0.2676 |     0.8419 |  96.637 |    7.461e-14 |
    | 408 |   408 |    0.7879 |     0.3740 |  33.517 |   -1.421e-14 |
    | 409 |   409 |    0.7037 |     0.5334 |  46.145 |    5.684e-14 |
    | 410 |   410 |    0.7031 |     0.1250 |  26.315 |    2.132e-14 |
    | 411 |   411 |    0.3882 |     0.2491 |  42.117 |    2.842e-14 |
    | 412 |   412 |    0.6623 |     0.4850 |  48.963 |    2.842e-14 |
    | 413 |   413 |    0.3935 |     0.2124 |  36.859 |   -9.237e-14 |
    | 414 |   414 |    0.1485 |     0.3198 |  38.401 |    2.842e-14 |
    | 415 |   415 |    0.8291 |     0.3346 |  29.661 |   -1.421e-14 |
    | 416 |   416 |    0.3975 |     0.1652 |  33.598 |    2.132e-14 |
    | 417 |   417 |    0.3982 |     0.1196 |  28.871 |    1.421e-14 |
    | 418 |   418 |    0.8707 |     0.3832 |  28.160 |    4.263e-14 |
    | 419 |   419 |    0.4863 |     0.7424 |  83.718 |   -8.527e-14 |
    | 420 |   420 |    0.2769 |     0.8818 |  71.884 |    0.000e+00 |
    | 421 |   421 |    0.5295 |     0.7426 |  75.898 |    1.066e-13 |
    | 422 |   422 |    0.5730 |     0.7432 |  62.412 |    4.263e-14 |
    | 423 |   423 |    0.6160 |     0.7967 |  50.120 |    2.842e-14 |
    | 424 |   424 |    0.6598 |     0.7412 |  48.469 |   -4.263e-14 |
    | 425 |   425 |    0.7046 |     0.7913 |  40.021 |    2.842e-14 |
    | 426 |   426 |    0.7493 |     0.7305 |  38.732 |    0.000e+00 |
    | 427 |   427 |    0.7957 |     0.7169 |  35.685 |   -2.398e-14 |
    | 428 |   428 |    0.8403 |     0.6980 |  31.395 |   -1.421e-14 |
    | 429 |   429 |    0.8809 |     0.6810 |  29.005 |   -2.842e-14 |
    | 430 |   430 |    0.9122 |     0.8186 |  24.372 |    4.263e-14 |
    | 431 |   431 |    0.2654 |     0.4701 |  75.981 |    2.842e-14 |
    | 432 |   432 |    0.3248 |     0.0531 |  23.966 |   -6.395e-14 |
    | 433 |   433 |    0.0924 |     0.9598 |  27.539 |    4.263e-14 |
    | 434 |   434 |    0.9556 |     0.0927 |  20.712 |    1.421e-14 |
    | 435 |   435 |    0.9044 |     0.9549 |  21.364 |    0.000e+00 |
    | 436 |   436 |    0.9110 |     0.0888 |  21.251 |    2.132e-14 |
    | 437 |   437 |    0.9559 |     0.7770 |  22.528 |   -1.421e-14 |
    | 438 |   438 |    0.9583 |     0.6362 |  23.001 |    3.553e-14 |
    | 439 |   439 |    0.9165 |     0.5327 |  26.433 |    2.842e-14 |
    | 440 |   440 |    0.9163 |     0.2795 |  23.687 |   -4.619e-14 |
    | 441 |   441 |    0.9584 |     0.5880 |  23.224 |    1.421e-14 |
    | 442 |   442 |    0.9582 |     0.4373 |  22.692 |   -2.842e-14 |
    | 443 |   443 |    0.9154 |     0.3812 |  24.925 |    2.842e-14 |
    | 444 |   444 |    0.9182 |     0.7059 |  25.861 |   -3.197e-14 |
    | 445 |   445 |    0.9164 |     0.2304 |  23.268 |    1.421e-14 |
    | 446 |   446 |    0.9158 |     0.1823 |  22.438 |    4.263e-14 |
    | 447 |   447 |    0.9456 |     0.7226 |  23.550 |   -1.421e-14 |
    | 448 |   448 |    0.9169 |     0.5834 |  26.126 |    5.684e-14 |
    | 449 |   449 |    0.9181 |     0.6751 |  25.754 |    2.132e-14 |
    | 450 |   450 |    0.9156 |     0.4328 |  25.801 |    4.263e-14 |
    | 451 |   451 |    0.9582 |     0.4887 |  23.075 |   -3.553e-14 |
    | 452 |   452 |    0.9582 |     0.3377 |  22.190 |   -2.842e-14 |
    | 453 |   453 |    0.9583 |     0.2881 |  22.017 |   -1.776e-14 |
    | 454 |   454 |    0.2718 |     0.1328 |  28.807 |    1.421e-14 |
    | 455 |   455 |    0.1191 |     0.0423 |  21.498 |    3.553e-14 |
    | 456 |   456 |    0.1726 |     0.4676 |  62.197 |   -8.527e-14 |
    | 457 |   457 |    0.7956 |     0.8644 |  28.722 |    7.105e-15 |
    | 458 |   458 |    0.6886 |     0.0923 |  24.413 |   -1.066e-14 |
    | 459 |   459 |    0.7078 |     0.8995 |  30.564 |    8.527e-14 |
    | 460 |   460 |    0.0766 |     0.0412 |  20.984 |    3.553e-14 |
    | 461 |   461 |    0.8694 |     0.8255 |  26.692 |   -5.684e-14 |
    | 462 |   462 |    0.4211 |     0.9113 |  52.340 |   -3.553e-14 |
    | 463 |   463 |    0.4326 |     0.0810 |  25.874 |    2.842e-14 |
    | 464 |   464 |    0.0927 |     0.8259 |  48.972 |   -1.279e-13 |
    | 465 |   465 |    0.0712 |     0.5687 |  47.346 |    9.592e-14 |
    | 466 |   466 |    0.3750 |     0.0356 |  22.561 |   -7.105e-15 |
    | 467 |   467 |    0.7770 |     0.0984 |  23.742 |    0.000e+00 |
    | 468 |   468 |    0.6121 |     0.0906 |  25.297 |   -2.842e-14 |
    | 469 |   469 |    0.6122 |     0.9028 |  36.276 |    4.263e-14 |
    | 470 |   470 |    0.3549 |     0.4610 |  76.455 |    1.208e-13 |
    | 471 |   471 |    0.5182 |     0.9030 |  45.066 |    1.421e-14 |
    | 472 |   472 |    0.5211 |     0.0892 |  26.057 |    0.000e+00 |
    | 473 |   473 |    0.3720 |     0.8760 |  72.403 |   -2.842e-14 |
    | 474 |   474 |    0.1196 |     0.6060 |  71.936 |   -5.684e-14 |
    | 475 |   475 |    0.1265 |     0.7951 |  67.559 |    2.842e-14 |
    | 476 |   476 |    0.0819 |     0.3088 |  30.340 |    2.132e-14 |
    | 477 |   477 |    0.0459 |     0.4013 |  29.028 |    5.684e-14 |
    | 478 |   478 |    0.1338 |     0.9304 |  38.981 |   -3.553e-14 |
    | 479 |   479 |    0.2657 |     0.9600 |  36.354 |   -5.684e-14 |
    | 480 |   480 |    0.2180 |     0.9572 |  37.688 |    8.527e-14 |
    | 481 |   481 |    0.3235 |     0.9195 |  54.520 |   -1.137e-13 |
    | 482 |   482 |    0.1225 |     0.6508 |  73.831 |   -5.684e-14 |
    | 483 |   483 |    0.0834 |     0.7004 |  55.760 |    0.000e+00 |
    | 484 |   484 |    0.0851 |     0.7341 |  58.428 |    8.527e-14 |
    | 485 |   485 |    0.0914 |     0.9177 |  34.582 |   -1.421e-14 |
    | 486 |   486 |    0.3524 |     0.5531 | 103.675 |   -1.741e-13 |
    | 487 |   487 |    0.0473 |     0.9550 |  24.156 |    7.105e-15 |
    | 488 |   488 |    0.9523 |     0.9528 |  20.655 |   -4.441e-14 |
    | 489 |   489 |    0.9530 |     0.0465 |  20.347 |    4.263e-14 |
    | 490 |   490 |    0.0445 |     0.0446 |  20.592 |    0.000e+00 |
    | 491 |   491 |    0.8665 |     0.8680 |  25.140 |    3.553e-14 |
    | 492 |   492 |    0.1266 |     0.4600 |  49.241 |    2.842e-14 |
    | 493 |   493 |    0.3103 |     0.4237 |  67.536 |    1.421e-14 |
    | 494 |   494 |    0.1909 |     0.9079 |  56.139 |    8.527e-14 |
    | 495 |   495 |    0.8662 |     0.0890 |  22.038 |    1.421e-14 |
    | 496 |   496 |    0.8253 |     0.8680 |  27.440 |    2.132e-14 |
    | 497 |   497 |    0.9139 |     0.1356 |  21.998 |    4.974e-14 |
    | 498 |   498 |    0.2336 |     0.1206 |  27.799 |   -5.995e-14 |
    | 499 |   499 |    0.0740 |     0.1633 |  23.786 |    1.421e-14 |
    | 500 |   500 |    0.0817 |     0.2469 |  27.416 |   -1.776e-14 |
    | 501 |   501 |    0.0910 |     0.4075 |  36.804 |    2.842e-14 |
    | 502 |   502 |    0.0308 |     0.3068 |  23.777 |   -7.105e-15 |
    | 503 |   503 |    0.2243 |     0.4268 |  61.855 |   -7.816e-14 |
    | 504 |   504 |    0.0783 |     0.4819 |  39.989 |    2.665e-15 |
    | 505 |   505 |    0.2741 |     0.0918 |  26.421 |   -4.263e-14 |
    | 506 |   506 |    0.3947 |     0.0727 |  25.730 |   -4.974e-14 |
    | 507 |   507 |    0.4082 |     0.8311 |  87.663 |   -5.684e-14 |
    | 508 |   508 |    0.1613 |     0.6033 |  86.789 |    1.137e-13 |
    | 509 |   509 |    0.1702 |     0.8150 |  82.436 |   -2.842e-13 |
    +-----+-------+-----------+------------+---------+--------------+

    Results per element:

    +------+-----+-----+-----+-----+----------+----------+----------+---------+---------+
    |   El |   N |   N |   N |   N |       qx |       qy |    q_res |   dT/dx |   dT/dy |
    |   Nb |   1 |   2 |   3 |   4 |   [W/m2] |   [W/m2] |   [W/m2] |   [C/m] |   [C/m] |
    |------+-----+-----+-----+-----+----------+----------+----------+---------+---------|
    |    1 | 344 | 343 | 231 | 331 |     59.0 |     32.7 |     67.4 |   -34.7 |   -19.3 |
    |    2 | 348 | 160 | 472 | 298 |     12.6 |   -126.3 |    127.0 |    -7.4 |    74.3 |
    |    3 | 336 | 246 | 471 | 321 |    160.7 |    521.4 |    545.6 |   -94.5 |  -306.7 |
    |    4 | 342 | 299 | 485 | 263 |   -389.3 |    227.8 |    451.1 |   229.0 |  -134.0 |
    |    5 | 341 | 347 | 469 | 264 |     84.0 |    258.0 |    271.4 |   -49.4 |  -151.8 |
    |    6 | 375 | 295 | 468 |  97 |     10.9 |    -98.3 |     98.9 |    -6.4 |    57.8 |
    |    7 | 354 | 112 | 467 | 297 |     19.1 |    -68.7 |     71.3 |   -11.2 |    40.4 |
    |    8 | 474 | 482 | 324 | 323 |   -796.2 |   -104.6 |    803.0 |   468.3 |    61.5 |
    |    9 | 308 | 479 | 480 | 293 |    -96.1 |    716.7 |    723.2 |    56.5 |  -421.6 |
    |   10 | 483 | 327 | 326 | 484 |   -851.7 |     69.1 |    854.5 |   501.0 |   -40.6 |
    |   11 | 299 | 290 | 478 | 485 |   -314.7 |    381.2 |    494.3 |   185.1 |  -224.3 |
    |   12 | 343 | 344 | 430 | 134 |     78.9 |     26.1 |     83.1 |   -46.4 |   -15.3 |
    |   13 | 346 | 262 | 486 | 319 |    167.4 |   -483.6 |    511.8 |   -98.5 |   284.5 |
    |   14 | 333 | 108 | 459 | 304 |     60.6 |    155.2 |    166.7 |   -35.7 |   -91.3 |
    |   15 | 161 | 313 | 314 | 455 |    -30.2 |    -67.7 |     74.1 |    17.8 |    39.8 |
    |   16 | 456 | 195 | 503 | 305 |   -241.2 |   -319.6 |    400.4 |   141.9 |   188.0 |
    |   17 | 501 | 477 | 309 | 292 |   -276.8 |    -84.6 |    289.4 |   162.8 |    49.7 |
    |   18 | 287 | 437 | 134 | 430 |     97.0 |     26.3 |    100.5 |   -57.0 |   -15.5 |
    |   19 | 294 | 472 | 160 | 374 |      5.7 |   -117.6 |    117.7 |    -3.4 |    69.2 |
    |   20 | 307 | 128 | 321 | 471 |    119.0 |    404.1 |    421.2 |   -70.0 |  -237.7 |
    |   21 | 482 | 327 | 483 | 324 |   -841.2 |    -43.4 |    842.3 |   494.8 |    25.5 |
    |   22 | 479 | 308 | 481 | 189 |      9.4 |    776.0 |    776.1 |    -5.5 |  -456.5 |
    |   23 | 462 | 378 | 473 | 206 |    146.4 |    762.3 |    776.2 |   -86.1 |  -448.4 |
    |   24 | 464 | 338 | 475 | 291 |   -671.2 |    289.1 |    730.8 |   394.8 |  -170.1 |
    |   25 | 465 | 303 | 474 | 323 |   -705.6 |   -205.4 |    734.9 |   415.1 |   120.8 |
    |   26 | 261 | 318 | 100 | 317 |   -107.1 |   -616.8 |    626.1 |    63.0 |   362.8 |
    |   27 |  95 | 355 | 188 |  96 |   -982.6 |     31.3 |    983.1 |   578.0 |   -18.4 |
    |   28 |   7 |  90 | 315 | 173 |    955.5 |    247.4 |    987.0 |  -562.0 |  -145.5 |
    |   29 |  90 |  89 | 316 | 315 |    659.1 |    -19.9 |    659.4 |  -387.7 |    11.7 |
    |   30 |  89 |  88 | 102 | 316 |    636.5 |     45.8 |    638.1 |  -374.4 |   -27.0 |
    |   31 | 127 | 334 |  75 |  76 |   -441.1 |    -55.7 |    444.6 |   259.4 |    32.8 |
    |   32 | 263 |  67 |  68 | 342 |   -385.4 |     52.7 |    389.0 |   226.7 |   -31.0 |
    |   33 | 191 |  71 |  72 | 328 |   -736.3 |    -32.2 |    737.0 |   433.1 |    18.9 |
    |   34 | 317 |  85 |   5 | 193 |   -232.8 |   -953.1 |    981.1 |   137.0 |   560.6 |
    |   35 | 102 | 383 | 358 | 316 |    591.6 |    -44.4 |    593.2 |  -348.0 |    26.1 |
    |   36 |  57 | 336 | 321 |  56 |     22.3 |    524.4 |    524.8 |   -13.1 |  -308.5 |
    |   37 |  74 |  75 | 334 | 175 |   -525.9 |    -27.8 |    526.6 |   309.4 |    16.4 |
    |   38 | 107 | 345 | 349 | 113 |    285.4 |   -330.9 |    437.0 |  -167.9 |   194.7 |
    |   39 | 342 |  68 |  69 | 190 |   -547.4 |     88.2 |    554.4 |   322.0 |   -51.9 |
    |   40 | 346 | 216 | 145 | 349 |    159.8 |   -346.9 |    381.9 |   -94.0 |   204.1 |
    |   41 | 329 |  49 |  50 | 158 |     17.6 |     96.3 |     97.9 |   -10.3 |   -56.7 |
    |   42 | 333 |  51 |  52 | 108 |     26.0 |    173.2 |    175.1 |   -15.3 |  -101.9 |
    |   43 |  54 | 347 | 341 |  53 |     41.8 |    284.3 |    287.4 |   -24.6 |  -167.2 |
    |   44 | 343 | 134 |  43 |  44 |     80.8 |     10.3 |     81.4 |   -47.5 |    -6.1 |
    |   45 | 228 |  73 |  74 | 322 |   -632.1 |    -53.9 |    634.4 |   371.8 |    31.7 |
    |   46 | 345 | 103 |   6 | 262 |    491.7 |   -538.9 |    729.5 |  -289.3 |   317.0 |
    |   47 | 327 | 355 |  95 | 325 |   -916.5 |     -4.2 |    916.6 |   539.1 |     2.5 |
    |   48 | 145 | 115 | 113 | 349 |    157.3 |   -275.4 |    317.2 |   -92.5 |   162.0 |
    |   49 | 231 | 343 |  44 |  45 |     59.2 |      8.7 |     59.9 |   -34.8 |    -5.1 |
    |   50 | 353 | 352 | 232 | 414 |   -140.1 |   -131.9 |    192.5 |    82.4 |    77.6 |
    |   51 | 217 | 326 | 327 | 325 |   -963.4 |     83.4 |    967.0 |   566.7 |   -49.1 |
    |   52 | 349 | 345 | 262 | 346 |    191.9 |   -413.5 |    455.9 |  -112.9 |   243.3 |
    |   53 | 108 |  52 |  53 | 341 |     18.2 |    218.4 |    219.2 |   -10.7 |  -128.5 |
    |   54 |  17 |  18 | 160 | 348 |     -3.1 |   -122.7 |    122.7 |     1.8 |    72.2 |
    |   55 | 313 | 161 | 362 | 363 |    -26.9 |    -84.1 |     88.3 |    15.8 |    49.5 |
    |   56 | 370 | 400 | 207 | 388 |    221.3 |   -111.0 |    247.6 |  -130.2 |    65.3 |
    |   57 | 190 |  69 |  70 | 337 |   -653.1 |     29.1 |    653.7 |   384.2 |   -17.1 |
    |   58 |  71 | 191 | 337 |  70 |   -724.6 |     46.4 |    726.1 |   426.2 |   -27.3 |
    |   59 |  12 |  13 | 111 | 362 |     -3.3 |   -103.3 |    103.4 |     1.9 |    60.8 |
    |   60 | 359 | 192 | 254 | 256 |    142.9 |      2.4 |    142.9 |   -84.1 |    -1.4 |
    |   61 | 166 | 357 |  24 |  25 |      7.7 |    -48.2 |     48.8 |    -4.5 |    28.3 |
    |   62 | 329 | 158 | 332 | 330 |     39.0 |     84.9 |     93.4 |   -22.9 |   -49.9 |
    |   63 | 135 | 138 | 379 | 368 |     66.8 |   -108.2 |    127.1 |   -39.3 |    63.6 |
    |   64 |  92 |  91 | 339 | 229 |    -34.5 |    989.8 |    990.4 |    20.3 |  -582.3 |
    |   65 | 412 | 207 | 122 | 393 |    169.3 |    -98.9 |    196.0 |   -99.6 |    58.2 |
    |   66 | 216 | 147 | 146 | 145 |     77.2 |   -292.2 |    302.2 |   -45.4 |   171.9 |
    |   67 | 157 | 365 | 221 | 181 |     66.1 |    -69.4 |     95.8 |   -38.9 |    40.8 |
    |   68 | 272 | 274 | 287 | 273 |    121.2 |     33.4 |    125.7 |   -71.3 |   -19.7 |
    |   69 | 107 | 113 | 114 | 110 |    255.2 |   -207.7 |    329.1 |  -150.1 |   122.2 |
    |   70 |  21 |  22 | 354 | 375 |      3.5 |    -87.2 |     87.2 |    -2.1 |    51.3 |
    |   71 | 154 | 151 | 350 | 156 |    -71.5 |   -264.7 |    274.2 |    42.1 |   155.7 |
    |   72 | 249 | 248 | 312 | 310 |    -94.1 |    -73.0 |    119.1 |    55.3 |    42.9 |
    |   73 |  59 | 377 | 283 |  58 |      3.4 |    685.6 |    685.6 |    -2.0 |  -403.3 |
    |   74 | 352 | 196 | 172 | 232 |   -104.6 |   -126.3 |    164.0 |    61.5 |    74.3 |
    |   75 | 201 | 186 | 241 | 372 |    407.6 |    402.0 |    572.5 |  -239.8 |  -236.5 |
    |   76 | 275 | 386 | 120 | 380 |    101.8 |   -160.7 |    190.2 |   -59.9 |    94.5 |
    |   77 |  32 | 126 | 367 |  31 |     60.2 |     -3.8 |     60.3 |   -35.4 |     2.3 |
    |   78 |  81 |  82 | 366 | 159 |    -89.7 |    -13.6 |     90.7 |    52.8 |     8.0 |
    |   79 | 170 | 142 | 135 | 368 |     72.6 |    -99.0 |    122.8 |   -42.7 |    58.2 |
    |   80 |  51 | 333 | 158 |  50 |     13.9 |    130.4 |    131.2 |    -8.2 |   -76.7 |
    |   81 | 136 | 130 | 140 | 139 |    226.4 |     19.9 |    227.3 |  -133.2 |   -11.7 |
    |   82 | 269 | 272 | 273 | 270 |    115.8 |     54.8 |    128.1 |   -68.1 |   -32.2 |
    |   83 | 348 | 101 |  16 |  17 |      5.9 |   -129.2 |    129.4 |    -3.5 |    76.0 |
    |   84 | 281 | 221 | 365 | 282 |     47.3 |    -63.9 |     79.5 |   -27.8 |    37.6 |
    |   85 | 173 | 186 | 201 | 174 |    573.2 |    399.8 |    698.9 |  -337.2 |  -235.2 |
    |   86 |  73 | 228 | 328 |  72 |   -706.5 |      5.0 |    706.5 |   415.6 |    -3.0 |
    |   87 | 365 | 157 | 142 | 170 |     65.8 |    -77.5 |    101.6 |   -38.7 |    45.6 |
    |   88 | 230 | 366 |  82 |  83 |    -61.5 |     -8.9 |     62.2 |    36.2 |     5.2 |
    |   89 | 168 | 167 | 154 | 156 |   -128.7 |   -260.8 |    290.9 |    75.7 |   153.4 |
    |   90 | 205 |  30 |  31 | 367 |     47.9 |     -7.3 |     48.5 |   -28.2 |     4.3 |
    |   91 | 269 | 268 | 266 | 267 |    130.1 |     95.6 |    161.4 |   -76.5 |   -56.2 |
    |   92 |  56 | 321 | 128 |  55 |     59.2 |    446.5 |    450.4 |   -34.8 |  -262.6 |
    |   93 | 130 | 381 | 182 | 140 |    223.0 |     -4.5 |    223.0 |  -131.2 |     2.6 |
    |   94 | 345 | 107 | 106 | 103 |    401.7 |   -249.8 |    473.0 |  -236.3 |   147.0 |
    |   95 | 192 | 359 | 178 | 177 |    159.0 |     14.3 |    159.6 |   -93.5 |    -8.4 |
    |   96 | 362 | 161 |  11 |  12 |    -12.1 |    -85.1 |     86.0 |     7.1 |    50.1 |
    |   97 | 284 | 233 | 210 | 224 |     40.8 |   -117.8 |    124.6 |   -24.0 |    69.3 |
    |   98 | 110 | 370 | 119 | 109 |    298.6 |   -117.5 |    320.9 |  -175.7 |    69.1 |
    |   99 | 138 | 133 | 386 | 184 |     83.2 |   -130.1 |    154.4 |   -49.0 |    76.5 |
    |  100 | 115 | 145 | 146 | 121 |    128.8 |   -256.1 |    286.7 |   -75.7 |   150.7 |
    |  101 | 186 | 173 | 315 | 419 |    553.3 |    283.9 |    621.9 |  -325.5 |  -167.0 |
    |  102 | 234 | 125 | 124 | 119 |    297.7 |    -31.9 |    299.4 |  -175.1 |    18.8 |
    |  103 | 388 | 234 | 119 | 370 |    266.7 |    -97.2 |    283.9 |  -156.9 |    57.2 |
    |  104 |  80 |  81 | 159 | 361 |   -119.1 |    -12.4 |    119.8 |    70.1 |     7.3 |
    |  105 | 103 | 102 |  88 |   6 |    786.1 |   -201.8 |    811.6 |  -462.4 |   118.7 |
    |  106 | 234 | 381 | 130 | 125 |    256.0 |    -31.0 |    257.9 |  -150.6 |    18.2 |
    |  107 | 241 | 242 | 243 | 372 |    296.5 |    280.6 |    408.3 |  -174.4 |  -165.1 |
    |  108 | 110 | 109 | 106 | 107 |    346.3 |   -201.8 |    400.8 |  -203.7 |   118.7 |
    |  109 | 196 | 248 | 249 | 202 |    -71.9 |    -93.5 |    118.0 |    42.3 |    55.0 |
    |  110 | 406 | 170 | 368 | 403 |     51.3 |    -90.8 |    104.3 |   -30.2 |    53.4 |
    |  111 | 369 | 404 | 405 | 277 |     54.3 |   -198.4 |    205.7 |   -32.0 |   116.7 |
    |  112 |  60 | 189 | 377 |  59 |     36.3 |    745.4 |    746.3 |   -21.3 |  -438.5 |
    |  113 | 393 | 131 | 394 | 373 |    132.0 |    -83.5 |    156.2 |   -77.7 |    49.1 |
    |  114 | 138 | 135 | 132 | 133 |     93.4 |   -119.0 |    151.3 |   -55.0 |    70.0 |
    |  115 | 379 | 138 | 184 | 185 |     71.9 |   -130.9 |    149.4 |   -42.3 |    77.0 |
    |  116 | 242 | 423 | 244 | 243 |    255.0 |    245.1 |    353.7 |  -150.0 |  -144.2 |
    |  117 |  95 |  94 | 217 | 325 |  -1000.0 |      6.8 |   1000.0 |   588.2 |    -4.0 |
    |  118 | 390 | 254 | 192 | 197 |    144.1 |     -2.6 |    144.1 |   -84.8 |     1.5 |
    |  119 | 113 | 115 | 116 | 114 |    198.9 |   -234.8 |    307.7 |  -117.0 |   138.1 |
    |  120 | 140 | 164 | 371 | 139 |    200.1 |     28.8 |    202.1 |  -117.7 |   -16.9 |
    |  121 | 103 | 106 | 383 | 102 |    500.4 |   -196.0 |    537.4 |  -294.3 |   115.3 |
    |  122 | 278 | 391 | 277 | 405 |     23.2 |   -173.0 |    174.6 |   -13.7 |   101.8 |
    |  123 | 351 | 149 | 148 | 147 |      9.2 |   -283.6 |    283.7 |    -5.4 |   166.8 |
    |  124 | 125 | 130 | 136 | 129 |    269.9 |     34.3 |    272.0 |  -158.7 |   -20.1 |
    |  125 | 120 | 386 | 133 | 123 |    119.1 |   -144.6 |    187.4 |   -70.1 |    85.1 |
    |  126 | 196 | 352 | 312 | 248 |    -99.0 |   -107.0 |    145.7 |    58.2 |    62.9 |
    |  127 | 347 |  54 |  55 | 128 |     23.8 |    350.5 |    351.3 |   -14.0 |  -206.2 |
    |  128 |  22 |  23 | 112 | 354 |     10.4 |    -76.5 |     77.2 |    -6.1 |    45.0 |
    |  129 | 380 | 120 | 117 | 116 |    150.9 |   -180.8 |    235.5 |   -88.7 |   106.4 |
    |  130 | 235 | 276 | 373 | 236 |    146.0 |    -57.5 |    156.9 |   -85.9 |    33.8 |
    |  131 | 281 | 226 | 219 | 221 |     48.5 |    -56.0 |     74.1 |   -28.6 |    33.0 |
    |  132 | 375 |  97 |  20 |  21 |      9.1 |   -101.2 |    101.6 |    -5.4 |    59.5 |
    |  133 | 185 | 209 | 211 | 210 |     49.5 |   -136.4 |    145.1 |   -29.1 |    80.2 |
    |  134 | 177 | 178 | 371 | 164 |    172.4 |     13.5 |    172.9 |  -101.4 |    -7.9 |
    |  135 | 350 | 151 | 150 | 149 |    -45.2 |   -264.5 |    268.3 |    26.6 |   155.6 |
    |  136 | 317 | 193 | 260 | 261 |   -280.7 |   -578.1 |    642.7 |   165.1 |   340.1 |
    |  137 | 164 | 165 | 179 | 177 |    174.0 |     -0.4 |    174.0 |  -102.3 |     0.3 |
    |  138 | 385 | 245 | 244 | 423 |    198.0 |    184.3 |    270.5 |  -116.4 |  -108.4 |
    |  139 | 223 | 212 | 391 | 278 |     27.2 |   -151.7 |    154.1 |   -16.0 |    89.3 |
    |  140 | 133 | 132 | 131 | 123 |    111.1 |   -110.4 |    156.6 |   -65.3 |    65.0 |
    |  141 | 380 | 116 | 115 | 121 |    125.1 |   -205.7 |    240.8 |   -73.6 |   121.0 |
    |  142 | 185 | 210 | 233 | 379 |     45.6 |   -119.2 |    127.7 |   -26.8 |    70.1 |
    |  143 | 171 | 237 | 402 | 163 |    -26.6 |   -161.5 |    163.7 |    15.7 |    95.0 |
    |  144 | 235 | 236 | 384 | 213 |    141.8 |    -52.3 |    151.1 |   -83.4 |    30.7 |
    |  145 | 275 | 380 | 121 | 369 |    103.3 |   -197.2 |    222.6 |   -60.8 |   116.0 |
    |  146 | 269 | 270 | 271 | 268 |    112.4 |     66.0 |    130.4 |   -66.1 |   -38.8 |
    |  147 | 109 | 119 | 124 | 118 |    352.2 |    -83.3 |    361.9 |  -207.2 |    49.0 |
    |  148 | 381 | 234 | 388 | 382 |    232.6 |    -52.8 |    238.5 |  -136.8 |    31.0 |
    |  149 | 386 | 275 | 208 | 184 |     87.4 |   -158.1 |    180.7 |   -51.4 |    93.0 |
    |  150 | 164 | 140 | 182 | 165 |    193.5 |    -11.3 |    193.9 |  -113.8 |     6.6 |
    |  151 | 358 | 383 | 118 | 215 |    469.9 |    -26.8 |    470.6 |  -276.4 |    15.8 |
    |  152 | 184 | 208 | 209 | 185 |     54.5 |   -141.6 |    151.7 |   -32.1 |    83.3 |
    |  153 | 232 | 172 | 155 | 376 |    -94.2 |   -172.1 |    196.2 |    55.4 |   101.2 |
    |  154 | 195 | 250 | 251 | 194 |   -238.3 |   -215.8 |    321.5 |   140.2 |   127.0 |
    |  155 | 109 | 118 | 383 | 106 |    419.3 |   -104.6 |    432.2 |  -246.7 |    61.5 |
    |  156 | 123 | 122 | 117 | 120 |    141.6 |   -135.2 |    195.8 |   -83.3 |    79.5 |
    |  157 | 369 | 277 | 208 | 275 |     61.7 |   -172.0 |    182.7 |   -36.3 |   101.2 |
    |  158 | 213 | 198 | 401 | 235 |    152.6 |    -35.1 |    156.6 |   -89.8 |    20.6 |
    |  159 |  19 |  20 |  97 | 374 |     -0.6 |   -109.0 |    109.0 |     0.3 |    64.1 |
    |  160 | 297 | 282 | 406 | 410 |     35.3 |    -75.6 |     83.4 |   -20.8 |    44.4 |
    |  161 | 393 | 122 | 123 | 131 |    145.4 |   -119.4 |    188.1 |   -85.5 |    70.2 |
    |  162 | 196 | 183 | 169 | 172 |    -70.5 |   -137.4 |    154.4 |    41.4 |    80.8 |
    |  163 | 221 | 219 | 218 | 181 |     59.4 |    -50.8 |     78.2 |   -35.0 |    29.9 |
    |  164 | 177 | 179 | 197 | 192 |    154.0 |     -9.3 |    154.3 |   -90.6 |     5.5 |
    |  165 | 282 | 365 | 170 | 406 |     54.0 |    -83.0 |     99.0 |   -31.8 |    48.8 |
    |  166 | 142 | 137 | 132 | 135 |     86.5 |    -93.2 |    127.2 |   -50.9 |    54.9 |
    |  167 | 203 | 303 | 306 | 302 |   -589.0 |   -375.7 |    698.6 |   346.5 |   221.0 |
    |  168 | 157 | 144 | 137 | 142 |     86.4 |    -84.1 |    120.5 |   -50.8 |    49.4 |
    |  169 | 403 | 285 | 410 | 406 |     42.2 |    -94.3 |    103.3 |   -24.8 |    55.5 |
    |  170 | 197 | 199 | 252 | 390 |    136.7 |    -19.4 |    138.1 |   -80.4 |    11.4 |
    |  171 | 376 | 155 | 153 | 154 |    -86.4 |   -199.5 |    217.4 |    50.8 |   117.3 |
    |  172 | 154 | 153 | 150 | 151 |    -59.8 |   -246.5 |    253.6 |    35.2 |   145.0 |
    |  173 | 258 | 214 | 213 | 384 |    131.4 |    -39.9 |    137.3 |   -77.3 |    23.5 |
    |  174 | 210 | 211 | 387 | 224 |     25.1 |   -125.7 |    128.2 |   -14.8 |    74.0 |
    |  175 | 181 | 180 | 144 | 157 |     78.6 |    -62.8 |    100.6 |   -46.2 |    36.9 |
    |  176 | 373 | 395 | 384 | 236 |    139.2 |    -59.0 |    151.2 |   -81.9 |    34.7 |
    |  177 | 198 | 213 | 214 | 199 |    138.7 |    -33.9 |    142.8 |   -81.6 |    19.9 |
    |  178 | 229 | 407 |  93 |  92 |     28.8 |    970.2 |    970.6 |   -16.9 |  -570.7 |
    |  179 | 279 | 255 | 219 | 226 |     43.4 |    -40.0 |     59.0 |   -25.5 |    23.5 |
    |  180 | 179 | 198 | 199 | 197 |    150.1 |    -16.3 |    151.0 |   -88.3 |     9.6 |
    |  181 | 199 | 214 | 257 | 252 |    130.6 |    -21.6 |    132.4 |   -76.8 |    12.7 |
    |  182 | 131 | 132 | 137 | 394 |    110.0 |   -100.1 |    148.7 |   -64.7 |    58.9 |
    |  183 | 149 | 150 | 152 | 148 |      1.6 |   -254.4 |    254.4 |    -0.9 |   149.6 |
    |  184 | 211 | 209 | 391 | 212 |     27.2 |   -145.7 |    148.3 |   -16.0 |    85.7 |
    |  185 | 237 | 171 | 239 | 238 |    -25.1 |   -151.5 |    153.5 |    14.8 |    89.1 |
    |  186 | 209 | 208 | 277 | 391 |     55.8 |   -161.2 |    170.6 |   -32.8 |    94.8 |
    |  187 | 220 | 180 | 181 | 218 |     77.5 |    -55.3 |     95.2 |   -45.6 |    32.5 |
    |  188 | 336 |  57 |  58 | 283 |     63.6 |    629.5 |    632.7 |   -37.4 |  -370.3 |
    |  189 | 118 | 124 | 392 | 215 |    383.0 |     17.1 |    383.4 |  -225.3 |   -10.0 |
    |  190 | 196 | 202 | 225 | 183 |    -70.7 |   -119.9 |    139.2 |    41.6 |    70.5 |
    |  191 | 172 | 169 | 163 | 155 |    -69.9 |   -154.2 |    169.3 |    41.1 |    90.7 |
    |  192 | 122 | 207 | 400 | 117 |    187.8 |   -143.2 |    236.2 |  -110.5 |    84.2 |
    |  193 | 125 | 129 | 392 | 124 |    317.9 |      7.9 |    318.0 |  -187.0 |    -4.6 |
    |  194 | 314 | 313 | 249 | 311 |    -52.3 |    -61.3 |     80.6 |    30.8 |    36.1 |
    |  195 | 432 | 301 | 240 | 300 |    -12.8 |   -133.2 |    133.9 |     7.5 |    78.4 |
    |  196 | 144 | 162 | 394 | 137 |    101.8 |    -75.8 |    126.9 |   -59.9 |    44.6 |
    |  197 | 370 | 110 | 114 | 400 |    249.2 |   -171.8 |    302.7 |  -146.6 |   101.1 |
    |  198 | 121 | 146 | 404 | 369 |     68.2 |   -221.2 |    231.4 |   -40.1 |   130.1 |
    |  199 | 116 | 117 | 400 | 114 |    184.5 |   -166.9 |    248.8 |  -108.5 |    98.2 |
    |  200 | 276 | 235 | 401 | 409 |    169.8 |    -56.3 |    178.9 |   -99.9 |    33.1 |
    |  201 | 154 | 167 | 397 | 376 |   -117.4 |   -228.9 |    257.2 |    69.1 |   134.6 |
    |  202 |  18 |  19 | 374 | 160 |      8.6 |   -119.9 |    120.2 |    -5.0 |    70.5 |
    |  203 | 238 | 239 | 300 | 240 |    -11.0 |   -135.0 |    135.4 |     6.5 |    79.4 |
    |  204 | 212 | 222 | 387 | 211 |     28.5 |   -135.6 |    138.6 |   -16.8 |    79.8 |
    |  205 | 218 | 219 | 255 | 396 |     59.9 |    -43.1 |     73.8 |   -35.2 |    25.3 |
    |  206 | 147 | 148 | 404 | 146 |     63.7 |   -259.0 |    266.7 |   -37.5 |   152.3 |
    |  207 | 381 | 382 | 409 | 182 |    207.1 |    -49.4 |    212.9 |  -121.8 |    29.1 |
    |  208 |  23 |  24 | 357 | 112 |      3.8 |    -59.7 |     59.8 |    -2.2 |    35.1 |
    |  209 | 368 | 379 | 233 | 403 |     57.0 |   -110.4 |    124.2 |   -33.5 |    64.9 |
    |  210 |  25 |  26 | 360 | 166 |      4.5 |    -33.4 |     33.7 |    -2.7 |    19.6 |
    |  211 | 212 | 223 | 280 | 222 |      8.2 |   -142.3 |    142.5 |    -4.8 |    83.7 |
    |  212 | 198 | 179 | 165 | 401 |    164.0 |    -28.5 |    166.4 |   -96.4 |    16.7 |
    |  213 | 247 | 227 | 204 | 187 |   -231.9 |    826.2 |    858.2 |   136.4 |  -486.0 |
    |  214 | 168 | 194 | 397 | 167 |   -155.3 |   -227.2 |    275.2 |    91.3 |   133.7 |
    |  215 | 373 | 394 | 162 | 395 |    119.2 |    -75.7 |    141.2 |   -70.1 |    44.5 |
    |  216 | 323 | 324 | 328 | 228 |   -733.5 |    -89.9 |    739.0 |   431.5 |    52.9 |
    |  217 | 304 | 332 | 158 | 333 |     55.6 |    122.1 |    134.2 |   -32.7 |   -71.8 |
    |  218 | 220 | 218 | 396 | 253 |     71.5 |    -37.5 |     80.7 |   -42.0 |    22.1 |
    |  219 | 302 | 260 | 193 | 203 |   -387.9 |   -522.9 |    651.1 |   228.2 |   307.6 |
    |  220 | 284 | 285 | 403 | 233 |     35.7 |   -102.2 |    108.2 |   -21.0 |    60.1 |
    |  221 | 162 | 144 | 180 | 399 |     98.4 |    -67.2 |    119.1 |   -57.9 |    39.5 |
    |  222 | 163 | 169 | 398 | 171 |    -47.7 |   -155.1 |    162.3 |    28.0 |    91.2 |
    |  223 | 165 | 182 | 409 | 401 |    184.0 |    -25.5 |    185.8 |  -108.3 |    15.0 |
    |  224 | 169 | 183 | 225 | 398 |    -52.1 |   -130.2 |    140.2 |    30.7 |    76.6 |
    |  225 | 155 | 163 | 402 | 153 |    -46.4 |   -193.0 |    198.5 |    27.3 |   113.6 |
    |  226 | 152 | 150 | 153 | 402 |    -32.4 |   -207.2 |    209.7 |    19.1 |   121.9 |
    |  227 | 405 | 411 | 413 | 278 |     16.5 |   -172.4 |    173.2 |    -9.7 |   101.4 |
    |  228 | 253 | 259 | 415 | 220 |     87.9 |    -42.2 |     97.5 |   -51.7 |    24.8 |
    |  229 | 148 | 152 | 405 | 404 |     14.8 |   -215.9 |    216.4 |    -8.7 |   127.0 |
    |  230 | 373 | 276 | 412 | 393 |    161.2 |    -85.1 |    182.3 |   -94.9 |    50.1 |
    |  231 | 276 | 409 | 382 | 412 |    184.3 |    -58.7 |    193.5 |  -108.4 |    34.5 |
    |  232 | 395 | 408 | 258 | 384 |    114.0 |    -55.2 |    126.6 |   -67.0 |    32.5 |
    |  233 | 376 | 397 | 414 | 232 |   -141.2 |   -165.8 |    217.7 |    83.0 |    97.5 |
    |  234 | 340 | 420 | 407 | 229 |    -53.7 |    931.3 |    932.8 |    31.6 |  -547.8 |
    |  235 | 402 | 411 | 405 | 152 |      4.0 |   -199.6 |    199.6 |    -2.4 |   117.4 |
    |  236 | 395 | 162 | 399 | 408 |    109.0 |    -57.6 |    123.3 |   -64.1 |    33.9 |
    |  237 | 402 | 237 | 413 | 411 |    -19.4 |   -176.6 |    177.7 |    11.4 |   103.9 |
    |  238 | 207 | 412 | 382 | 388 |    205.8 |    -94.6 |    226.5 |  -121.1 |    55.6 |
    |  239 | 194 | 251 | 414 | 397 |   -190.4 |   -199.3 |    275.6 |   112.0 |   117.2 |
    |  240 |   8 |  93 | 407 | 187 |   -175.0 |   1230.3 |   1242.7 |   102.9 |  -723.7 |
    |  241 | 292 | 296 | 414 | 251 |   -209.5 |   -134.8 |    249.2 |   123.3 |    79.3 |
    |  242 | 237 | 238 | 416 | 413 |     -4.0 |   -153.1 |    153.1 |     2.3 |    90.0 |
    |  243 | 278 | 413 | 416 | 223 |      1.8 |   -159.6 |    159.6 |    -1.1 |    93.9 |
    |  244 | 417 | 416 | 238 | 240 |    -10.6 |   -144.2 |    144.6 |     6.2 |    84.9 |
    |  245 |  85 | 317 | 100 |  86 |     13.1 |   -660.3 |    660.5 |    -7.7 |   388.4 |
    |  246 | 399 | 415 | 258 | 408 |    109.2 |    -52.3 |    121.1 |   -64.3 |    30.8 |
    |  247 | 259 | 418 | 258 | 415 |    101.6 |    -34.6 |    107.3 |   -59.8 |    20.4 |
    |  248 | 422 | 421 | 215 | 392 |    391.7 |     83.2 |    400.4 |  -230.4 |   -48.9 |
    |  249 | 180 | 220 | 415 | 399 |     89.2 |    -47.2 |    101.0 |   -52.5 |    27.8 |
    |  250 | 417 | 280 | 223 | 416 |      8.8 |   -142.4 |    142.7 |    -5.2 |    83.8 |
    |  251 | 214 | 258 | 418 | 257 |    116.7 |    -37.3 |    122.5 |   -68.6 |    21.9 |
    |  252 | 316 | 358 | 419 | 315 |    619.6 |    113.9 |    629.9 |  -364.4 |   -67.0 |
    |  253 | 293 | 247 | 420 | 308 |    -92.3 |    808.3 |    813.6 |    54.3 |  -475.5 |
    |  254 | 187 | 407 | 420 | 247 |   -193.0 |    861.9 |    883.2 |   113.6 |  -507.0 |
    |  255 | 421 | 422 | 242 | 241 |    374.7 |    215.5 |    432.3 |  -220.4 |  -126.8 |
    |  256 | 215 | 421 | 419 | 358 |    480.3 |    115.2 |    494.0 |  -282.6 |   -67.8 |
    |  257 | 129 | 136 | 424 | 143 |    262.0 |     72.4 |    271.8 |  -154.1 |   -42.6 |
    |  258 | 186 | 419 | 421 | 241 |    472.2 |    207.1 |    515.6 |  -277.8 |  -121.8 |
    |  259 | 245 | 385 | 425 | 265 |    176.0 |    157.0 |    235.9 |  -103.5 |   -92.3 |
    |  260 | 129 | 143 | 422 | 392 |    323.4 |    101.9 |    339.1 |  -190.2 |   -60.0 |
    |  261 | 242 | 422 | 143 | 423 |    290.8 |    156.8 |    330.4 |  -171.1 |   -92.2 |
    |  262 | 267 | 426 | 427 | 269 |    147.7 |     63.2 |    160.7 |   -86.9 |   -37.2 |
    |  263 | 428 | 272 | 269 | 427 |    140.6 |     46.2 |    148.0 |   -82.7 |   -27.2 |
    |  264 | 143 | 424 | 385 | 423 |    248.6 |    145.6 |    288.1 |  -146.2 |   -85.7 |
    |  265 | 427 | 178 | 359 | 428 |    149.6 |     25.7 |    151.8 |   -88.0 |   -15.1 |
    |  266 | 385 | 424 | 141 | 425 |    201.1 |    106.9 |    227.8 |  -118.3 |   -62.9 |
    |  267 | 139 | 141 | 424 | 136 |    226.3 |     76.1 |    238.8 |  -133.1 |   -44.8 |
    |  268 | 266 | 265 | 425 | 267 |    144.6 |    116.0 |    185.4 |   -85.0 |   -68.3 |
    |  269 | 139 | 371 | 426 | 141 |    187.9 |     50.9 |    194.7 |  -110.5 |   -29.9 |
    |  270 | 426 | 371 | 178 | 427 |    171.2 |     44.5 |    176.9 |  -100.7 |   -26.2 |
    |  271 | 141 | 426 | 267 | 425 |    177.9 |     94.8 |    201.6 |  -104.7 |   -55.8 |
    |  272 | 272 | 428 | 429 | 274 |    128.6 |     24.0 |    130.8 |   -75.6 |   -14.1 |
    |  273 | 256 | 429 | 428 | 359 |    142.4 |     21.2 |    143.9 |   -83.7 |   -12.5 |
    |  274 | 480 | 356 | 494 | 293 |   -114.5 |    699.4 |    708.7 |    67.4 |  -411.4 |
    |  275 | 473 | 378 | 481 | 340 |    105.0 |    816.3 |    823.0 |   -61.8 |  -480.2 |
    |  276 | 470 | 320 | 493 | 351 |    -25.4 |   -375.6 |    376.4 |    15.0 |   220.9 |
    |  277 | 307 | 469 | 347 | 128 |    124.6 |    333.5 |    356.0 |   -73.3 |  -196.2 |
    |  278 | 294 | 374 |  97 | 468 |     18.2 |   -111.4 |    112.9 |   -10.7 |    65.5 |
    |  279 |  42 | 437 | 287 | 447 |    109.8 |     11.2 |    110.3 |   -64.6 |    -6.6 |
    |  280 | 260 | 305 | 431 | 261 |   -200.3 |   -468.3 |    509.4 |   117.8 |   275.5 |
    |  281 | 229 | 339 | 473 | 340 |     98.9 |    964.0 |    969.1 |   -58.2 |  -567.1 |
    |  282 | 338 | 326 | 217 | 475 |   -863.2 |    287.1 |    909.7 |   507.8 |  -168.9 |
    |  283 | 330 | 331 | 435 | 329 |     38.6 |     61.6 |     72.7 |   -22.7 |   -36.2 |
    |  284 |  14 |  15 | 466 | 432 |      1.8 |   -126.2 |    126.2 |    -1.1 |    74.2 |
    |  285 | 441 |  39 |  40 | 438 |    125.9 |      4.0 |    126.0 |   -74.1 |    -2.4 |
    |  286 | 445 | 440 | 253 | 396 |     72.3 |    -30.7 |     78.6 |   -42.5 |    18.0 |
    |  287 | 449 | 444 | 274 | 429 |    129.2 |     21.5 |    130.9 |   -76.0 |   -12.6 |
    |  288 | 256 | 254 | 448 | 286 |    135.9 |      6.7 |    136.1 |   -80.0 |    -4.0 |
    |  289 |  35 |  98 | 452 |  34 |    100.0 |     -7.3 |    100.2 |   -58.8 |     4.3 |
    |  290 |  38 | 105 | 451 |  37 |    123.3 |      1.2 |    123.4 |   -72.6 |    -0.7 |
    |  291 | 252 | 257 | 450 | 288 |    121.8 |    -22.0 |    123.7 |   -71.6 |    13.0 |
    |  292 |  10 | 455 | 314 | 460 |    -19.4 |    -48.8 |     52.5 |    11.4 |    28.7 |
    |  293 | 320 | 318 | 261 | 431 |   -103.3 |   -484.8 |    495.7 |    60.8 |   285.2 |
    |  294 | 354 | 297 | 410 | 458 |     24.8 |    -85.2 |     88.7 |   -14.6 |    50.1 |
    |  295 | 266 | 304 | 459 | 265 |    112.9 |    143.9 |    182.9 |   -66.4 |   -84.6 |
    |  296 |  13 |  14 | 432 | 111 |     -9.0 |   -114.4 |    114.7 |     5.3 |    67.3 |
    |  297 |  63 |  64 | 433 | 176 |    -44.2 |    381.8 |    384.3 |    26.0 |  -224.6 |
    |  298 |  30 | 205 | 434 |  29 |     33.3 |     -4.7 |     33.6 |   -19.6 |     2.7 |
    |  299 |  49 | 329 | 435 |  48 |     10.2 |     63.7 |     64.6 |    -6.0 |   -37.5 |
    |  300 | 461 | 491 | 496 | 271 |     88.8 |     62.3 |    108.5 |   -52.2 |   -36.7 |
    |  301 | 438 | 286 | 448 | 441 |    130.2 |     -1.6 |    130.2 |   -76.6 |     1.0 |
    |  302 | 449 |  99 | 447 | 444 |    122.9 |      7.8 |    123.2 |   -72.3 |    -4.6 |
    |  303 | 445 | 126 | 453 | 440 |     71.1 |    -14.7 |     72.6 |   -41.8 |     8.6 |
    |  304 | 451 | 288 | 450 | 442 |    117.8 |     -6.9 |    118.0 |   -69.3 |     4.1 |
    |  305 |  42 |  43 | 134 | 437 |     95.4 |      3.9 |     95.4 |   -56.1 |    -2.3 |
    |  306 |  41 |  99 | 438 |  40 |    124.2 |     -1.2 |    124.2 |   -73.1 |     0.7 |
    |  307 | 390 | 252 | 288 | 439 |    130.1 |     -8.3 |    130.4 |   -76.5 |     4.9 |
    |  308 |  39 | 441 | 105 |  38 |    128.3 |     -3.9 |    128.3 |   -75.5 |     2.3 |
    |  309 | 259 | 253 | 440 | 289 |     84.0 |    -24.9 |     87.6 |   -49.4 |    14.7 |
    |  310 |  36 | 442 |  98 |  35 |    108.3 |     -1.3 |    108.3 |   -63.7 |     0.8 |
    |  311 | 418 | 259 | 289 | 443 |     99.7 |    -28.2 |    103.6 |   -58.7 |    16.6 |
    |  312 | 126 | 445 | 446 | 367 |     59.2 |    -18.9 |     62.2 |   -34.8 |    11.1 |
    |  313 | 396 | 255 | 446 | 445 |     56.6 |    -27.1 |     62.8 |   -33.3 |    15.9 |
    |  314 |  99 |  41 |  42 | 447 |    113.9 |      7.1 |    114.1 |   -67.0 |    -4.2 |
    |  315 | 439 | 448 | 254 | 390 |    134.3 |     -8.3 |    134.6 |   -79.0 |     4.9 |
    |  316 | 442 |  36 |  37 | 451 |    119.1 |     -6.4 |    119.3 |   -70.1 |     3.8 |
    |  317 | 443 | 450 | 257 | 418 |    110.4 |    -19.0 |    112.1 |   -65.0 |    11.2 |
    |  318 |  34 | 452 | 453 |  33 |     86.4 |     -2.9 |     86.5 |   -50.8 |     1.7 |
    |  319 | 126 |  32 |  33 | 453 |     75.2 |     -7.6 |     75.6 |   -44.2 |     4.5 |
    |  320 | 286 | 449 | 429 | 256 |    131.6 |      7.4 |    131.8 |   -77.4 |    -4.4 |
    |  321 | 282 | 297 | 467 | 281 |     37.8 |    -70.0 |     79.5 |   -22.3 |    41.2 |
    |  322 | 280 | 417 | 506 | 463 |     -4.2 |   -135.7 |    135.8 |     2.4 |    79.8 |
    |  323 | 285 | 284 | 468 | 295 |     31.2 |   -102.6 |    107.2 |   -18.3 |    60.4 |
    |  324 | 245 | 264 | 469 | 244 |    158.8 |    238.4 |    286.4 |   -93.4 |  -140.3 |
    |  325 | 147 | 216 | 470 | 351 |     59.7 |   -338.7 |    343.9 |   -35.1 |   199.2 |
    |  326 | 171 | 398 | 454 | 239 |    -28.6 |   -137.1 |    140.0 |    16.8 |    80.6 |
    |  327 | 305 | 503 | 200 | 431 |   -183.9 |   -379.7 |    421.9 |   108.1 |   223.4 |
    |  328 | 201 | 372 | 471 | 246 |    252.7 |    460.9 |    525.6 |  -148.7 |  -271.1 |
    |  329 | 222 | 298 | 472 | 387 |      8.7 |   -128.4 |    128.7 |    -5.1 |    75.5 |
    |  330 | 287 | 430 | 461 | 273 |     98.0 |     37.1 |    104.8 |   -57.6 |   -21.9 |
    |  331 | 389 | 467 | 112 | 357 |     21.1 |    -59.2 |     62.9 |   -12.4 |    34.8 |
    |  332 | 492 | 250 | 195 | 456 |   -309.3 |   -274.9 |    413.8 |   182.0 |   161.7 |
    |  333 | 309 | 477 |  77 |  78 |   -286.3 |    -42.9 |    289.5 |   168.4 |    25.2 |
    |  334 | 414 | 296 | 476 | 353 |   -187.2 |   -112.6 |    218.5 |   110.1 |    66.3 |
    |  335 | 326 | 338 | 337 | 484 |   -764.8 |    106.2 |    772.1 |   449.9 |   -62.5 |
    |  336 |  63 | 356 | 480 |  62 |    -85.9 |    566.0 |    572.4 |    50.5 |  -332.9 |
    |  337 | 318 | 319 | 486 | 100 |     42.9 |   -587.5 |    589.1 |   -25.2 |   345.6 |
    |  338 |  87 | 486 | 262 |   6 |    174.3 |   -767.8 |    787.4 |  -102.5 |   451.7 |
    |  339 |  47 | 488 |  46 |   3 |     11.7 |     11.7 |     16.6 |    -6.9 |    -6.9 |
    |  340 |  27 |   2 |  28 | 489 |      6.3 |     -6.3 |      8.9 |    -3.7 |     3.7 |
    |  341 |  65 |   4 |  66 | 487 |    -76.5 |     76.5 |    108.2 |    45.0 |   -45.0 |
    |  342 | 306 | 175 | 334 | 504 |   -519.2 |   -165.7 |    545.1 |   305.4 |    97.5 |
    |  343 |  10 |  11 | 161 | 455 |     -6.8 |    -65.2 |     65.6 |     4.0 |    38.4 |
    |  344 | 149 | 351 | 493 | 350 |    -17.8 |   -309.6 |    310.1 |    10.5 |   182.1 |
    |  345 | 194 | 168 | 503 | 195 |   -201.9 |   -278.0 |    343.6 |   118.7 |   163.5 |
    |  346 |  80 | 104 | 502 |  79 |   -172.4 |    -19.6 |    173.5 |   101.4 |    11.5 |
    |  347 | 251 | 250 | 501 | 292 |   -270.5 |   -178.7 |    324.1 |   159.1 |   105.1 |
    |  348 | 204 | 227 | 494 | 290 |   -330.0 |    760.1 |    828.6 |   194.1 |  -447.1 |
    |  349 | 313 | 363 | 202 | 249 |    -56.1 |    -92.5 |    108.2 |    33.0 |    54.4 |
    |  350 |  84 |   1 |   9 | 490 |    -11.3 |    -11.3 |     16.0 |     6.6 |     6.6 |
    |  351 | 389 | 357 | 166 | 495 |     17.1 |    -43.8 |     47.0 |   -10.1 |    25.8 |
    |  352 | 286 | 438 |  99 | 449 |    126.1 |     10.0 |    126.5 |   -74.2 |    -5.9 |
    |  353 | 105 | 439 | 288 | 451 |    126.1 |    -10.4 |    126.5 |   -74.2 |     6.1 |
    |  354 | 439 | 105 | 441 | 448 |    129.2 |      0.8 |    129.2 |   -76.0 |    -0.5 |
    |  355 | 452 | 289 | 440 | 453 |     86.3 |    -18.4 |     88.2 |   -50.8 |    10.8 |
    |  356 | 443 |  98 | 442 | 450 |    109.7 |    -16.0 |    110.8 |   -64.5 |     9.4 |
    |  357 |  98 | 443 | 289 | 452 |     96.5 |    -12.0 |     97.3 |   -56.8 |     7.1 |
    |  358 | 274 | 444 | 447 | 287 |    114.0 |     19.6 |    115.7 |   -67.1 |   -11.5 |
    |  359 | 239 | 454 | 505 | 300 |    -27.4 |   -131.5 |    134.3 |    16.1 |    77.3 |
    |  360 | 377 | 378 | 462 | 283 |    125.1 |    707.9 |    718.8 |   -73.6 |  -416.4 |
    |  361 | 298 | 463 | 101 | 348 |     -1.8 |   -128.1 |    128.1 |     1.1 |    75.4 |
    |  362 |  74 | 175 | 465 | 322 |   -630.7 |    -83.9 |    636.3 |   371.0 |    49.3 |
    |  363 | 299 | 342 | 190 | 464 |   -494.7 |    177.0 |    525.4 |   291.0 |  -104.1 |
    |  364 | 337 | 338 | 464 | 190 |   -685.6 |    202.4 |    714.8 |   403.3 |  -119.1 |
    |  365 | 246 | 336 | 283 | 462 |    103.5 |    602.9 |    611.7 |   -60.9 |  -354.7 |
    |  366 | 228 | 322 | 465 | 323 |   -671.0 |    -73.0 |    675.0 |   394.7 |    42.9 |
    |  367 | 302 | 456 | 305 | 260 |   -345.2 |   -416.8 |    541.2 |   203.1 |   245.2 |
    |  368 | 491 | 461 | 430 | 344 |     81.1 |     48.7 |     94.6 |   -47.7 |   -28.6 |
    |  369 | 304 | 266 | 268 | 457 |     93.1 |    109.0 |    143.4 |   -54.8 |   -64.1 |
    |  370 | 264 | 459 | 108 | 341 |     84.6 |    210.6 |    227.0 |   -49.7 |  -123.9 |
    |  371 | 285 | 295 | 458 | 410 |     26.3 |    -89.1 |     92.9 |   -15.5 |    52.4 |
    |  372 | 295 | 375 | 354 | 458 |     22.0 |    -87.8 |     90.5 |   -12.9 |    51.7 |
    |  373 | 245 | 265 | 459 | 264 |    119.9 |    181.6 |    217.6 |   -70.5 |  -106.8 |
    |  374 | 300 | 505 | 111 | 432 |     -9.3 |   -119.9 |    120.3 |     5.4 |    70.5 |
    |  375 | 476 | 104 | 500 | 353 |   -162.3 |    -79.7 |    180.8 |    95.5 |    46.9 |
    |  376 | 492 | 335 | 501 | 250 |   -335.2 |   -175.3 |    378.3 |   197.2 |   103.1 |
    |  377 | 436 | 434 | 205 | 497 |     32.5 |    -19.4 |     37.9 |   -19.1 |    11.4 |
    |  378 | 398 | 225 | 498 | 454 |    -44.0 |   -130.4 |    137.6 |    25.9 |    76.7 |
    |  379 | 273 | 461 | 271 | 270 |    103.8 |     57.1 |    118.4 |   -61.0 |   -33.6 |
    |  380 | 201 | 246 | 462 | 206 |    297.4 |    626.3 |    693.3 |  -174.9 |  -368.4 |
    |  381 | 222 | 280 | 463 | 298 |     13.0 |   -133.3 |    134.0 |    -7.6 |    78.4 |
    |  382 | 290 | 299 | 464 | 291 |   -535.9 |    401.5 |    669.6 |   315.2 |  -236.2 |
    |  383 | 175 | 306 | 303 | 465 |   -559.9 |   -153.9 |    580.7 |   329.4 |    90.5 |
    |  384 | 201 | 206 | 507 | 174 |    367.1 |    724.3 |    812.0 |  -215.9 |  -426.0 |
    |  385 | 290 | 291 | 509 | 204 |   -617.6 |    539.8 |    820.2 |   363.3 |  -317.5 |
    |  386 | 101 | 466 |  15 |  16 |     -4.7 |   -125.7 |    125.8 |     2.7 |    73.9 |
    |  387 | 226 | 389 | 495 | 279 |     32.8 |    -44.5 |     55.3 |   -19.3 |    26.2 |
    |  388 | 226 | 281 | 467 | 389 |     31.2 |    -53.9 |     62.3 |   -18.4 |    31.7 |
    |  389 | 202 | 363 | 498 | 225 |    -44.9 |   -104.4 |    113.6 |    26.4 |    61.4 |
    |  390 | 224 | 294 | 468 | 284 |     19.3 |   -110.5 |    112.2 |   -11.3 |    65.0 |
    |  391 | 243 | 244 | 469 | 307 |    173.0 |    282.1 |    330.9 |  -101.8 |  -165.9 |
    |  392 | 319 | 470 | 216 | 346 |     74.1 |   -412.3 |    418.9 |   -43.6 |   242.5 |
    |  393 | 318 | 320 | 470 | 319 |     18.1 |   -454.1 |    454.5 |   -10.7 |   267.1 |
    |  394 | 243 | 307 | 471 | 372 |    237.8 |    372.1 |    441.6 |  -139.9 |  -218.9 |
    |  395 | 224 | 387 | 472 | 294 |     24.8 |   -122.6 |    125.1 |   -14.6 |    72.1 |
    |  396 | 505 | 454 | 498 | 364 |    -25.9 |   -117.7 |    120.5 |    15.2 |    69.2 |
    |  397 | 320 | 431 | 200 | 493 |    -78.8 |   -385.3 |    393.3 |    46.4 |   226.7 |
    |  398 |  48 | 435 | 488 |  47 |     13.1 |     38.1 |     40.3 |    -7.7 |   -22.4 |
    |  399 |  64 |  65 | 487 | 433 |    -72.7 |    241.9 |    252.6 |    42.8 |  -142.3 |
    |  400 |  29 | 434 | 489 |  28 |     20.5 |     -7.0 |     21.7 |   -12.1 |     4.1 |
    |  401 | 495 | 436 | 497 | 279 |     30.2 |    -29.4 |     42.1 |   -17.7 |    17.3 |
    |  402 | 255 | 279 | 497 | 446 |     45.3 |    -31.2 |     55.0 |   -26.7 |    18.4 |
    |  403 | 268 | 271 | 496 | 457 |    100.3 |     86.6 |    132.5 |   -59.0 |   -51.0 |
    |  404 | 292 | 309 | 476 | 296 |   -221.6 |    -99.4 |    242.9 |   130.4 |    58.4 |
    |  405 |  76 |  77 | 477 | 127 |   -353.7 |    -18.4 |    354.1 |   208.0 |    10.8 |
    |  406 | 335 | 127 | 477 | 501 |   -359.8 |   -119.4 |    379.1 |   211.6 |    70.2 |
    |  407 | 308 | 420 | 340 | 481 |    -42.3 |    833.7 |    834.8 |    24.9 |  -490.4 |
    |  408 |  63 | 176 | 478 | 356 |    -99.6 |    486.5 |    496.6 |    58.6 |  -286.2 |
    |  409 | 479 |  61 |  62 | 480 |      2.8 |    700.1 |    700.1 |    -1.7 |  -411.8 |
    |  410 | 327 | 482 | 188 | 355 |   -978.0 |    -52.8 |    979.4 |   575.3 |    31.1 |
    |  411 | 189 |  60 |  61 | 479 |    -32.1 |    723.0 |    723.7 |    18.9 |  -425.3 |
    |  412 | 324 | 483 | 191 | 328 |   -763.1 |      5.7 |    763.2 |   448.9 |    -3.4 |
    |  413 | 483 | 484 | 337 | 191 |   -778.5 |     26.2 |    778.9 |   457.9 |   -15.4 |
    |  414 |  83 |  84 | 490 | 230 |    -38.0 |    -12.4 |     40.0 |    22.4 |     7.3 |
    |  415 |  86 | 100 | 486 |  87 |    -40.1 |   -635.9 |    637.2 |    23.6 |   374.1 |
    |  416 |  67 | 263 | 487 |  66 |   -245.3 |     82.8 |    258.9 |   144.3 |   -48.7 |
    |  417 |  45 |  46 | 488 | 231 |     36.8 |     12.2 |     38.8 |   -21.6 |    -7.2 |
    |  418 |  26 |  27 | 489 | 360 |      7.0 |    -20.6 |     21.7 |    -4.1 |    12.1 |
    |  419 | 314 | 230 | 490 | 460 |    -32.1 |    -32.2 |     45.4 |    18.9 |    18.9 |
    |  420 | 344 | 331 | 330 | 491 |     59.7 |     51.7 |     79.0 |   -35.1 |   -30.4 |
    |  421 | 463 | 506 | 466 | 101 |      5.1 |   -130.3 |    130.4 |    -3.0 |    76.7 |
    |  422 | 240 | 301 | 506 | 417 |      0.5 |   -134.0 |    134.0 |    -0.3 |    78.8 |
    |  423 | 456 | 302 | 306 | 492 |   -393.0 |   -280.1 |    482.6 |   231.2 |   164.8 |
    |  424 | 156 | 350 | 493 | 200 |    -96.3 |   -322.8 |    336.9 |    56.6 |   189.9 |
    |  425 | 193 |   5 | 508 | 203 |   -829.5 |   -726.3 |   1102.5 |   488.0 |   427.2 |
    |  426 |  94 |   8 | 509 | 217 |  -1284.5 |    394.0 |   1343.5 |   755.6 |  -231.8 |
    |  427 |  91 |   7 | 507 | 339 |    296.5 |   1212.7 |   1248.4 |  -174.4 |  -713.3 |
    |  428 |  96 | 188 | 508 |   5 |  -1201.6 |   -212.5 |   1220.3 |   706.8 |   125.0 |
    |  429 | 187 | 204 | 509 |   8 |   -677.4 |   1096.5 |   1288.9 |   398.5 |  -645.0 |
    |  430 | 173 | 174 | 507 |   7 |    813.5 |    788.2 |   1132.7 |  -478.5 |  -463.7 |
    |  431 | 247 | 293 | 494 | 227 |   -214.3 |    707.8 |    739.5 |   126.1 |  -416.3 |
    |  432 | 189 | 481 | 378 | 377 |     15.2 |    756.5 |    756.7 |    -9.0 |  -445.0 |
    |  433 | 203 | 508 | 474 | 303 |   -753.5 |   -253.2 |    794.9 |   443.3 |   149.0 |
    |  434 | 491 | 330 | 332 | 496 |     67.3 |     81.7 |    105.8 |   -39.6 |   -48.0 |
    |  435 | 500 | 361 | 310 | 312 |   -108.3 |    -62.1 |    124.9 |    63.7 |    36.5 |
    |  436 | 249 | 310 | 499 | 311 |    -78.0 |    -61.9 |     99.5 |    45.9 |    36.4 |
    |  437 | 352 | 353 | 500 | 312 |   -134.9 |    -83.7 |    158.7 |    79.4 |    49.2 |
    |  438 | 362 | 364 | 498 | 363 |    -30.1 |   -109.2 |    113.3 |    17.7 |    64.3 |
    |  439 |  80 | 361 | 500 | 104 |   -155.6 |    -34.4 |    159.3 |    91.5 |    20.3 |
    |  440 | 314 | 311 | 366 | 230 |    -54.1 |    -36.0 |     65.0 |    31.8 |    21.2 |
    |  441 | 311 | 499 | 159 | 366 |    -79.2 |    -35.8 |     86.9 |    46.6 |    21.1 |
    |  442 | 310 | 361 | 159 | 499 |   -107.0 |    -46.0 |    116.4 |    62.9 |    27.0 |
    |  443 | 309 |  78 |  79 | 502 |   -227.3 |    -13.3 |    227.7 |   133.7 |     7.8 |
    |  444 | 156 | 200 | 503 | 168 |   -131.8 |   -302.8 |    330.3 |    77.5 |   178.1 |
    |  445 | 362 | 111 | 505 | 364 |    -23.5 |   -108.3 |    110.8 |    13.8 |    63.7 |
    |  446 | 127 | 335 | 504 | 334 |   -420.0 |    -90.8 |    429.7 |   247.0 |    53.4 |
    |  447 | 485 | 433 | 487 | 263 |   -208.2 |    229.7 |    310.0 |   122.5 |  -135.1 |
    |  448 | 331 | 231 | 488 | 435 |     35.0 |     33.5 |     48.4 |   -20.6 |   -19.7 |
    |  449 | 436 | 360 | 489 | 434 |     18.1 |    -18.3 |     25.8 |   -10.7 |    10.8 |
    |  450 | 478 | 176 | 433 | 485 |   -210.2 |    365.5 |    421.6 |   123.6 |  -215.0 |
    |  451 | 360 | 436 | 495 | 166 |     19.4 |    -32.8 |     38.1 |   -11.4 |    19.3 |
    |  452 | 367 | 446 | 497 | 205 |     44.4 |    -16.2 |     47.3 |   -26.1 |     9.5 |
    |  453 | 304 | 457 | 496 | 332 |     79.4 |     99.0 |    126.9 |   -46.7 |   -58.2 |
    |  454 | 335 | 492 | 306 | 504 |   -425.4 |   -221.0 |    479.4 |   250.2 |   130.0 |
    |  455 |  10 | 460 | 490 |   9 |     -9.5 |    -34.4 |     35.7 |     5.6 |    20.2 |
    |  456 | 432 | 466 | 506 | 301 |     -7.8 |   -124.9 |    125.1 |     4.6 |    73.5 |
    |  457 | 290 | 494 | 356 | 478 |   -292.9 |    553.7 |    626.4 |   172.3 |  -325.7 |
    |  458 | 104 | 476 | 309 | 502 |   -215.9 |    -56.5 |    223.1 |   127.0 |    33.2 |
    |  459 | 206 | 473 | 339 | 507 |    300.2 |    819.2 |    872.4 |  -176.6 |  -481.9 |
    |  460 | 482 | 474 | 508 | 188 |   -855.4 |   -235.3 |    887.2 |   503.2 |   138.4 |
    |  461 | 291 | 475 | 217 | 509 |   -772.5 |    425.4 |    881.9 |   454.4 |  -250.3 |
    +------+-----+-----+-----+-----+----------+----------+----------+---------+---------+
    ```

=== "Stress problem - 3 node (plante)"

    ```
    {!stressmodel_v3_el2.txt!}
    ```

=== "Stress problem - 4 node (planqe)"

    ```
    {!stressmodel_v3_el3.txt!}
    ```




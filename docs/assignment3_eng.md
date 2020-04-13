# Program for Technical Applications - Worksheet 3

> ** Remember: ** When ** ... ** appears in the program examples, this indicates that there is no code that you yourself have to add. Variables and data structures are just examples. Depending on the type of problem you may need other data structures than those described in the code examples.

## General

In this worksheet contains the following elements:

  1. The classes implemented in sheet 2 is adapted to cope with a geometric model that describes the calculation model parametrically.
  2. Scanning Procedures must be adapted to handle the geometric model.
  2. grid generation must be implemented using gmsh ** ** and the results visualized.
  3. Visualization of the model to be made with the routines of calfem.vis ** **.

## Geometric model

In this worksheet, we will update our classes to manage a geometric model that is the basis for nätgenereringen our solver ** ** - class. Instead of defining the model in terms of nodes and elements we define it instead with parameters such as width, height and

In this worksheet we use a little more Calfem modules Add the following modules in the module where your classes are defined:

    import calfem.core as cfc
    import calfem.geometry as cfg  # <-- Geometrirutiner
    import calfem.mesh as cfm      # <-- Nätgenerering
    import calfem.vis as cfv       # <-- Visualisering
    import calfem.utils as cfu     # <-- Blandade rutiner
    
## Updating the InputData class

**Input Data** - class must be updated to handle a parametric model described instead of defining the problem at the element level. For groundwater problem, for example, parameters **w** **t** , **d**  and **H** to describe the geometry and **EP**  **kx** and **ky** to describe elements properties. Variables for loads and boundary conditions can be maintained. The difference now is that instead of degree of freedom and the value stores the boundary conditions marker and value instead.

All variables such as **coord** and **Edof**  describing the input of element level to be moved to the **Solver** - class because they will be created when the elements are generated online.

The geometry will be defined with the objects and functions of  **calfem.geometry**. To the **solver** - class always get an updated geometry at the call of **execute()** - the method we create a method in the Input Data **geometry()** **  ** which will return a **geometry** instance based on values ​​of the input variables that we defined for the geometry. This method is then called from **execute()**, so that the solver always get an updated geometry description, even if the input variablern has changed. An example of how this method might look like the following example:

    class InputData(object):
        """Klass för att definiera indata för vår modell."""
        ...
        
        def geometry(self):
            """Skapa en geometri instans baserat på definierade parametrar"""
            
            # --- Skapa en geometri-instans för att lagra vår
            #     geometribeskrivning
            
            g = cfg.Geometry()
            
            # --- Enklare att använda referenser till self.xxx
            
            w = self.w
            h = self.h
            t = self.t
            d = self.d
            
            # --- Punkter i modellen skapas med point(...) metoden 
            
            g.point([0, 0])
            g.point([w, 0])
            g.point([w, h])
            
            ...
            
            # --- Linker och kurvor skapas med spline(...) metoden
            
            g.spline([0, 1])            
            g.spline([1, 2])           
            g.spline([2, 3], marker=...) # <-- Använd marker för att
                                         #     definiera linjer som 
                                         #     skall ha laster eller
                                         #     randvillkor
            
            ...
            
            # --- Ytan på vilket nätet skall genereras definieras med 
            #     surface(...) metoden.
            
            g.surface([0,1, ... ,6,7])

            # --- Slutligen returnerar vi den skapade geometrin
            
            return g
            
## Uppdatering av Solver-klassen

In **Solver** - class, we must add the call to the mesh generator in  **calfem.mesh**, **GmshMeshGenerator** . This will give us the element coordinates, topology, and the variables that can be used for the connection between geometry and element mesh. An example of how this might look in **execute ()** - metodn shown below:

    class Solver(object):
        """Klass för att hantera lösningen av vår beräkningsmodell."""

        ...
                    
        def execute(self):
            """Metod för att utföra finita element beräkningen."""
            
            # --- Överför modell variabler till lokala referenser
            
            version = self.inputData.version
            ep = self.inputData.ep

            ...
            
            
            # --- Anropa InputData för en geomtetribeskrivning
            
            geometry = self.inputData.geometry()        
            
            # --- Nätgenerering
            
            el_type = 3      # <-- Fyrnodselement flw2i4e
            dofs_per_node = 1  # <-- Skalärt problem
            
            meshGen = cfm.GmshMeshGenerator(geometry)
            meshGen.el_size_factor = 0.5     # <-- Anger max area för element
            meshGen.el_type = el_type
            meshGen.dofs_per_node = dofs_per_node
            meshGen.return_boundary_elements = True
            
            coords, edof, dofs, bdofs, element_markers, boundary_elements = meshGen.create()
            
Element Generation and assembling does not need to be changed from worksheet 2. However, one must handling of loads and boundary conditions are updated, because we are now working load markers instead of degrees of freedom. Use the functions  **applybc(...)** and **applyforcetotal(...)**/**applyTractionLinearElement(...)** available in **calfem.utils** module.

Solving equations and elements of power calculation does not need to be changed. However, we need to store the generated variables **coord**, **EdoF**, **geometry** and other output variables needed to visualize the results.

> It may also be useful to define a variable in the **InputData** - class to set the maximum size of the generated elements such as **elSizeFactor**, which can then be assigned to **cfm.GmshGenerator** - class feature **elSizeFactor**.

## Using the parametric model

The idea of the parametric description of the problem is that a user easily be able to specify their area of concern in the code without having to go into the module for the model. This is shown in the following code:

    # -*- coding: utf-8 -*-

    import flowmodel as fm

    if __name__ == "__main__":
        
        inputData = fm.InputData()

        inputData.w = 100.0
        inputData.h = 10.0
        inputData.d = 5.0
        inputData.t = 0.5
        inputData.kx = 20.0
        inputData.ky = 20.0
        
        ...

In this way, one can also easily study the effect of, for example, increase the depth of the tongue in the ground and find out how this affects the flow. For example:

    # -*- coding: utf-8 -*-

    import flowmodel as fm
    import numpy as np

    if __name__ == "__main__":
        
        dRange = np.linspace(3.0, 7.0, 10).tolist()
        
        for d in dRange:
        
            print("-------------------------------------------")    
            print("Simulating d = ", d)
        
            inputData = fm.InputData()
        
            inputData.w = 100.0
            inputData.h = 10.0
            inputData.d = d
            inputData.t = 0.5
            inputData.kx = 20.0
            inputData.ky = 20.0
            
            outputData = fm.OutputData()
        
            solver = fm.Solver(inputData, outputData)
            solver.execute()
            
            print("Max flow = ", np.max(outputData.maxFlow))        


## Report class

In ** Report ** - class, we need to add the ability to print the geometry description.

## Visualisation with a new class Visualisation

In worksheet 2 was the only resulatet a printout from our **Report** - class. To interpret the results in text form can be difficult. We must therefore use us Calfem: routines for visualization module **calfem.vis** . These procedures are based on the established visualization package **visvis** . The visualization will be implemented in the new class **Visualization**. This class have the same input data **Result** - class ie references to instances of **InputData** and **OutputData**.

There are a number of visualization features in Calfem. In this sheet, the following visualizations are implemented:

 * Geometry - draw_geometry(...)
 * Generated network - draw_mesh(...)
 * Deformed networks - draw_mesh(...) (for example voltage)
 * Element Values ​​- draw_element_values​​(...)
 * Node values ​​- draw_nodal_values​(...)
 
Documentation for these procedures, see the user manual for [nätgeneringsrutinerna] (http://training.lunarc.lu.se/pluginfile.php/473/mod_resource/content/1/DAE_rapport_draft06.pdf).

Note that these routines are integrated in Calfem and visvis do not need to be imported explicitly. The following code from the manual:

    vv.figure()
    pcv.drawMesh(coords=coords, edof=edof, dofsPerNode=dofsPerNode, elType=elType, filled=True, title="Mesh")
    
becomes instead (med import enligt tidigare):

    cfv.figure()
    cfv.draw_mesh(coords=coords, edof=edof, dofs_per_node=dofs_per_node, el_type=el_type, filled=True, title="Mesh")
    
The following code shows how the class can be implemented with the visualization of the geometry.

    class Visualisation(object):
        def __init__(self, inputData, outputData):
            self.inputData = inputData
            self.outputData = outputData
            
        def show(self):
            
            geometry = self.outputData.geometry
            a = self.outputData.a
            max_flow = self.outputData.max_flow
            coords = self.outputData.coords
            edof = self.outputData.edof
            dofs_per_node = self.outputData.dofs_per_node
            el_type = self.outputData.el_type
            
            cfv.figure() 
            cfv.draw_geometry(geometry, title="Geometry")
            
            ...
                       
        def wait(self):
            """Denna metod ser till att fönstren hålls uppdaterade och kommer att returnera
            När sista fönstret stängs"""

            cfv.showAndWait()
            
Visualization ** ** - class is then added to the main program in the code shown:

    # -*- coding: utf-8 -*-

    import flowmodel as fm

    if __name__ == "__main__":
        
        inputData = fm.InputData()

        outputData = fm.OutputData()

        solver = fm.Solver(inputData, outputData)
        solver.execute()

        report = fm.Report(inputData, outputData)
        print(report)
        
        vis = fm.Visualisation(inputData, outputData)
        vis.show()
        vis.wait()        

**vis.wait()** must be called last in the main program, since this function does not return until the final visualization window closes.

## Submission and reporting

It is to be done in this worksheet are:

 * Change ** Input Data ** - class to describe the problem parametrically according to the examples described last in the sheet. Create a method ** geometry () ** that returns a cfg.Geometry ** ** - body with geometry defined on the basis of the parameter description.
 * Update ** ** Solver - class so that this creates an element mesh using cfm.GmshMeshGenerator ** ** class. Save also maximum flows / maxspänningar (von Mises) and store these in UDATA class.
 * Completing the implementation of Visualization ** ** - class so that it can handle visualize geometry element mesh, elements flows and node values.
 * Makes a parameter study wherein one of the parameters is varied and maximum flow / max voltage are plotted relative to the selected parameter. Create a new main program for parameter study.

The submission shall consist of a zip file (or other archive format) consisting of:

 * All the Python files. (.py files)
 * An example of a saved file JSON.
 * Printing from program execution.
 * Printing of the parameter study.

## Exempelproblem

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



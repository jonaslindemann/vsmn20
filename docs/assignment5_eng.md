# Program for Technical Applications - Worksheet 5

## General

This worksheet contains the following tasks:

 1. Add user interface controls for performing parameter studies
 1. Add input variables in the InputData-class for managing parameter studies.
 1. Implement a routine in the Solver-class to perform parameter studies.
 1. Implement a routine to export the results from a parameter study to ParaView using pyvtk.
 
## User interface controls for the parameter study

Select 2 of the parameters in the interface and add controls to be able to select the start value, end value, number of steps to be simulated.

The following picture shows an example of how this can be done:

![param_ui1](images/param_ui1.png)

The following controls and associated names have been added:

![param_ui2](images/param_ui2.png)

Set good default values on all controls so that one can perform a calculation without having to fill in all values.

Connect an event method, **on_execute_param_study** to the **Param study** button. In order to handle parameter studies in our model, we must add a number of extra variables in our **ModelPArams** class:

 * **param_d** - Flag indicating whether parameter d should be varied.
 * **param_t** - Flag indicating whether the parameter t should be varied.
 * **d_start** - Starting value of d.
 * **d_end** - Ending value of d.
 * **t_start** - Start value of t.
 * **t_end** - Ending value of t.
 * **param_filename** - Variable controlling how filenames should be formatted.
 * **param_steps** - Variable controlling the number of steps in the parameter study.
 
**on_execute_param_study** should have approximately the same structure as **onActionExecute**. At the beginning of the method, we add assigment of the new parameters based on the new controls. The following code illustrates this:

```py
def on_execute_param_study(self):
    """Exekvera parameterstudie"""

    # --- Update model from UI

    self.update_model()    

    # --- Update filename

    self.model_param.param_filename = "param_study"

    # --- Skapa en lösare

    self.solver = sm.ModelSolver(self.model_params, self.model_results)
    
    # --- Starta en tråd för att köra beräkningen, så att 
    #     gränssnittet inte fryser.
    
    self.solverThread = SolverThread(self.solver, param_study = True)        
    self.solverThread.finished.connect(self.on_solver_finished)        
    self.solverThread.start()
```

The only change is that we pass the parameter, **param_study=True**, to the **SolverThread** class constructor. The **SolverThread**-class will be updated in later sections. 
## Update the Solver-class to handle parameter studies

In the **ModelSolver** class, we now add a routine, **execute_param_study(...)**, which implements the mechanics to execute the parameter study.

```py
def execute_param_study(self):
    """Kör parameter studie"""
    
    # -- Store current values for the d and t parameters
    
    old_d = self.model_params.d
    old_t = self.model_params.t
    
    i = 1
    
    if self.model_params.param_d:
    
        # --- Create a simulation range
    
        d_range = np.linspace(self.model_params.d_start, self.model_params.d_end,
            self.model_params.param_steps)
            
        # --- Loop over the range
            
        for d in d_range:
            print("Executing for d = %g..." % d)
            
            # --- Modify parameter in self.model_params
            ...
            # --- Execute calculation
            ...
            # --- Export to VTK
            ...
            
    elif self.model_params.param_t:
        ...
            
    # --- Restore previous values of d and t
    
    self.model_params.d = old_d
    self.model_params.t = old_t
```
        
### Exporting results to VTK-files

In this worksheet, we will export the results and display them in the visualization software, ParaView. ParaView uses standardized file format with the extension .vtk. To create these files, we will use a special Python library, **pyvtk**, this library makes it easy to create these files.

First, we add the import directive at the top of our module:

```py
import pyvtk as vtk
```

In the next step, we will add a method in the **ModelSolver**-class, **export_vtk(...)** to perform the actual export.

**pyvtk** has a lot of data types. We will start from the primitive **vtk.PolyData**. This data type is well suited to managing unstructured elements that we have in this application. To be able to visualise this type of data, points and topology are needed. This information is available from the calculations. However, **pyvtk** does not handle NumPy arrays, so we have to do some tricks to convert these into the right format:

### export_vtk(...) for scalar problems

```py
def export_vtk(self, filename):
    """Export results to VTK"""

    print("Exporting results to %s." % filename)

    # --- Extract points and polygons

    points = self.model_results.coords.tolist()
    polygons = (self.model_results.edof-1).tolist()

    # --- Create point data from a

    point_data = vtk.PointData(
        vtk.Scalars(self.model_results.a.tolist(), name="pressure")
    )

    # --- Create cell data from max_flow and flow

    cell_data = vtk.CellData(
        vtk.Scalars(self.model_results.max_flow, name="max_flow"),
        vtk.Vectors(self.model_results.flow, "flow")
    )

    # --- Create structure

    structure = vtk.PolyData(points=points, polygons=polygons)

    # --- Export to vtk

    vtk_data = vtk.VtkData(structure, point_data, cell_data)
    vtk_data.tofile(filename, "ascii")

```

### export_vtk(...) for vector problems (stress)

Modify the execute method to extract the node topology, **topo**:

```py
def execute(self):
    """Metod för att utföra finita element beräkningen."""
    
    ...        
    
    # --- Mesh generation
    
    el_type = 3
    dofs_per_node = 1 
    geometry = self.model_params.geometry()        
    
    mesh = cfm.GmshMeshGenerator(geometry)
    mesh.el_size_factor = el_size_factor     # Factor that changes element sizes.
    mesh.el_type = el_type
    mesh.dofs_per_node = dofs_per_node
    
    coords, edof, dofs, bdofs, elementmarkers = mesh.create()

    self.model_results.topo = mesh.topo  # <-- ADDED
```

The VTK export routine is shown below. Modify with your own variables.

```py
def export_vtk(self, filename):
    """Export results to VTK"""        
    
    print("Exporting results to %s." % filename)

    # --- Extract points and polygons
    
    points = self.model_results.coords.tolist()
    polygons = (self.model_results.topo-1).tolist()
    
    # --- Create point data from a

    displ = np.reshape(self.model_results.a, (len(points),2)).tolist()
                        
    pointData = vtk.PointData(vtk.Vectors(displ, name="displacements"))

    # --- Create cell data from mises and principal stresses

    von_mises = np.reshape(self.model_results.evm, (self.model_results.evm.shape[0],))

    cellData = vtk.CellData(
        vtk.Scalars(von_mises, name="vonmises"),
        vtk.Vectors(self.model_results.ep1.tolist(), name="principal1"),
        vtk.Vectors(self.model_results.ep2.tolist(), name="principal2")
    )

    # --- Create structure

    structure = vtk.PolyData(points = points, polygons = polygons)

    # --- Export to VTK
    
    vtkData = vtk.VtkData(structure, cellData, pointData)
    vtkData.tofile(filename, "ascii")
```
        
ParaView can automatically manage the results of the parameter study if the files are named **param_study_01.vtk**, **param_study_02.vtk**. The file can be opened as a file **param_study** in the application.
       
## Updating the SolverThread-class

The **SolverThread** class needs to be updated to be able to run a parameter study as well. We define an additional class attribute, **self.param_study**, that indicates if a parameter study or a normal calculation should be performed:

```py
class SolverThread(QThread):
    """Background calucation thread"""
    
    def __init__(self, solver, param_study = False):
        """Constructor"""
        super().__init__()
        self.param_study = param_study # <-- ADDED
        self.solver = solver

    ...        
        
    def run(self):
        if self.param_study: # <-- UPDATED
            self.solver.execute_param_study()
        else:
            self.solver.execute()
```

## Submission and reporting

What to do in this worksheet is:

 * Implement interface controls and buttons for parameter study functionality
 * Update the **ModelSolver** class to manage parameter studies.
 * Implement the **export_vtk(...)** function in the **ModelSolver** class.
 * The method **execute_param_study(...)** for each calculation must write a vtk file with the model filename with an added number. That is, [filename]_01.vtk, [filename]_02.vtk.
 * Import the vtk files into ParaView for visualization.
  
The submission must consist of a zip file (or other archive format) consisting of:

 * All Python files. (.Py files)
 * Pictures from ParaView visualizations.
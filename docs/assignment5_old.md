# Arbetsblad 5

## Allmänt

I detta arbetsblad innehåller följande moment:

 1. Utöka gränssnittet med kontroller för att kunna göra parameterstudier.
 1. Lägga till input-parametrar i ModelParams-klassen för att kunna hantera parameterstudierna.
 1. Implementera en rutin i **ModelSolver**-klassen för att hantera parameterstudier.
 1. Implementera en rutin för att exportera resultaten från en körning till en Visualisation Toolkit (.vtk) fil med hjälp av pyvtk.
 
## Kontroller för parameterstudier

Välj ut 2 av parametrarna i gränsnittet och lägg till kontroller för att kunna välja startvärde, slutvärde, antal steg som skall simuleras.

Följande bild visar ett exempel på hur detta kan se ut:

![param_ui1](images/param_ui1.png)

Följande kontroller och tillhörande namn har lagts till:

![param_ui2](images/param_ui2.png)

Sätt bra standard värden på alla kontroller, så att det går att utföra en beräkning utan behöva fylla i alla värden.

Koppla en händelsemetod, **on_execute_param_study** till **Param study**-knappen. För att kunna hantera parameterstudier i vår modell måste vi lägga till ett antal extra inparametrar i vår **ModelParams**-klass:

 * **param_d** - Flagga som anger om parametern d skall varieras.
 * **param_t** - Flagga som anger om parametern t skall varieras.
 * **d_start** - Startvärde på d.
 * **d_end** - Slutvärde på d.
 * **t_start** - Startvärde på t.
 * **t_end** - Slutvärde på t.
 * **param_filename** - Variabel som bestämmer hur filnamnen skall namnges.
 * **param_steps** - Variable som anger antalet steg i studein.
 
 
 **on_execute_param_study** skall ha ungefär ha samma struktur som **on_action_execute**. I början av metoden lägger vi till tilldelning av de nya parametrarna utifrån de nya kontrollerna. Följande kod visar hur detta kan se ut: 


``` py
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

## Uppdatera ModelSolver-klassen för att hantera parameterstudier

I **ModelSolver**-klassen måste vi nu lägga till en rutin, **execute_param_study(...)**, som implementerar mekaniken att exekvera parameterstudien.

``` py
def execute_param_study(self):
    """Kör parameter studie"""
    
    # -- Lagra tidigare värden på d
    
    old_d = self.model_params.d
    old_t = self.model_params.t
    
    i = 1
    
    if self.model_params.param_d:
    
        # --- Skapa värden att simulera över
    
        d_range = np.linspace(self.model_params.d_start, self.model_params.d_end, self.model_params.paramSteps)
            
        # --- Starta parameterstudien
            
        for d in d_range:
            print("Executing for d = %g..." % d)
            
            # --- Sätt önskad parameter i ModelParams-instansen
            ...
            # --- Kör beräkningen 
            ...
            # --- Exportera vtk-fil
            ...
            
    elif self.model_params.param_t:
        ...
            
    # --- Återställ ursprungsvärden
    
    self.model_params.d = old_d
    self.model_params.t = old_t
```
        
## Export av resultat till VTK-filer

I detta arbetsblad kommer vi att exportera resultaten och visa dessa i ett etablerat visualiseringsverktyg, ParaView. ParaView använder sig av ett speciellt standardiserat filformat där filerna har ändelsen .vtk. För att skapa dessa filer skall vi använda ett speciellt Python-bibliotek, **pyvtk**, detta bibliotek gör det enkelt att på en hög nivå skapa dessa filer.

Först lägger vi till import-direktivet längst upp i vår modul:

``` py
import pyvtk as vtk
```

I nästa steg skall vi skapa en metod i **Solver**-klassen, **exportVtk(...)** för att utföra själa exporten.

**pyvtk** har en mängd datatyper. Vi kommer att utgå från primitiven **vtk.PolyData**. Denna datatype lämpar sig bra till att hantera ostrukturerade element som vi har i denna tillämpning. För att definiera uppritning av denna datatyp behövs punkter och topologi. Av en händelse har vi detta som ett resultat av beräkningen. **pyvtk** hanterar dock inte NumPy-arrayer, så vi får göra lite tricks för att konvertera dessa till rätt format:

### export_vkt(...) för skalära problem

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
        
ParaView kan automatiskt hantera resultaten från parameterstudien om filerna namnges **param_study_01.vtk**, **param_study_02.vtk**. Filen kan öppnas som en fil **param_study** i programmet.        

## Uppdatering av **ModelSolver**-klassen för spänningsproblemet

För att spänningsproblement skall fungera måste vi lagra ytterligare en variabel i **OutputData**-klassen:

``` py
def execute(self):
    """Metod för att utföra finita element beräkningen."""
    
    # --- Överför modell variabler till lokala referenser

    ...        
    
    # --- Nätgenerering
    
    el_type = 3
    dofs_per_node= 1 
    geometry = self.model_params.geometry()        
    
    mesh_gen = cfm.GmshMeshGenerator(geometry)
    mesh_gen.el_size_factor = el_size_factor     # Factor that changes element sizes.
    mesh_gen.el_type = el_type
    mesh_gen.dofs_per_node = dofs_per_node
    
    coords, edof, dofs, bdofs, elementmarkers = meshGen.create()
    self.model_results.topo = meshGen.topo
```
        
Topo innehåller nodtopologin, som kan användas med vtk.

## Uppdatering av SolverThread-klassen

För att kunna köra en parameterstudie måste vi anropa den tidigare definierade metode **execute_param_study(...)** istället för **execute(...)**. Detta gör vi genom att definiera en ytterligare metod-variabler **self.param_study** som anger om en parameterstudie skall köras eller en enstaka beräkning. Koden för att starta en parameterstudie visas nedan. Notera den extra parametern i **SolverThread**:s konstruktur.

``` py
def on_execute_param_study(self):
    """Exekvera parameterstudie"""
    
    ...

    # --- Starta en tråd för att köra beräkningen, så att 
    #     gränssnittet inte fryser.
    
    self.solver_thread = SolverThread(self.solver, param_study = True)        
    self.solver_thread.finished.connect(self.on_solver_finished)        
    self.solver_thread.start()
```

## Inlämning och redovisning

Det som skall göras i detta arbetsblad är:

 * Implementera gränssnittskontroller och knappar för parameterstudiefunktionaliteten
 * Uppdatera **Solver**-klassen att hantera parameterstudier.
 * Implementera funktionen **exportVtk(...)** i **Solver**-klassen.
 * Metoden **excuteParamStudy(...)** skall för varje beräkning skriva ut en vtk-fil men samma filnamn som modellen. Dvs [filnamn]_01.vtk, [filnamn]_02.vtk.
 * Importera vtk-filerna i ParaView för visualisering. 
  
Inlämningen skall bestå av en zip-fil (eller annat arkivformat) bestående av: 

 * Alla Python-filer. (.py-filer)
 * Bilder från ParaView-visualiseringar.
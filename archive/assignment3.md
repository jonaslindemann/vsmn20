# Arbetsblad 3

!!! note "Viktigt"

    När **...** visas i programexemplen anger detta att det saknas kod som ni själva måste lägga till. Variabler och datastrukturer är bara exempel. Beroende på problemtyp kan man behöva andra datastrukturer än de som är beskrivna i kodexemplen.

## Allmänt

I detta arbetsblad innehåller följande moment:

 1.	Klasserna implementerade i arbetsblad 2 skall anpassas för att hantera en geometrisk model som beskriver beräkningsmodellen parametriskt.
 1. Inläsningsrutiner måste anpassas för att hantera den geometriska modellen.
 1.	Nätgenerering skall implementeras med hjälp av **GMSH** och resultat visualiseras.
 1.	Visualisering av modellen skall göras med rutinerna i **calfem.vis**.

## Geometrisk modell

I detta arbetsblad skall vi uppdatera våra klasser för att hantera en geometrisk modell som ligger till grund för nätgenereringen i vår **Solver**-klass. Istället för att definiera modellen med avseende på noder och element definierar vi den istället med parametrar som bredd, höjd.

I detta arbetsblad skall vi använda lite fler CALFEM moduler Lägg till följande moduler i modulen där era klasser är definierade:

``` py
import calfem.core as cfc
import calfem.geometry as cfg  # <-- Geometrirutiner
import calfem.mesh as cfm      # <-- Nätgenerering
import calfem.vis_mpl as cfv   # <-- Visualisering
import calfem.utils as cfu     # <-- Blandade rutiner
```

## Uppdatering av ModelParams-klassen

**ModelParams**-klassen måset uppdateras för att hantera en parametriskt beskriven modell istället för att definiera problemet på elementnivå. För grundvattenproblemet har t ex parametrarna, **w**, **t**, **d** och **h** för att beskriva geometrin och **ep**, **kx** och **ky** för att beskriva element egenskaper. Variabler för laster och randvillkor kan bibehållas. Skillnaden nu är att vi istället för frihetsgrad och värde lagrar randvillkorsmarkör och värde istället.

Alla variabler som t ex **coord** och **edof** som beskriver indata på elementnivå skall flyttas till **ModelSolver**-klassen eftersom dessa kommer att skapas när elementnätet genereras.

Geometrin kommer att definieras med objekt och funktioner från **calfem.geometry**. För att **Solver**-klassen alltid skall få en uppdaterad geometri vid anropet av **execute()**-metoden skapar vi en metod i **ModelParams**, **geometry()** som skall returnera en **Geometry** instans utifrån värden i de indata variabler som vi definierade för geometrin. Denna metod anropas sedan från **execute()**, så att lösaren alltid får en uppdaterad geometribeskrivning även om indata variablern har ändrats. Ett exempel på hur denna metod kan se ut visas i följande exempel:

``` py
class ModelParams:
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
``` 

## Uppdatering av ModelSolver-klassen

I **ModelSolver**-klassen måste vi lägga till anrop till nätgenereraren i **calfem.mesh**, **GmshMeshGenerate**. Denna kommer att ge oss element koordinater, topologi samt variabler som kan användas för koppling mellan geometri och elementnät. Ett exempel på hur detta kan se ut i **execute()**-metodn visas nedan:

``` py
class ModelSolver:
    """Klass för att hantera lösningen av vår beräkningsmodell."""

    ...
                
    def execute(self):
        """Metod för att utföra finita element beräkningen."""
        
        # --- Överför modell variabler till lokala referenser
        
        version = self.model_params.version
        ep = self.model_params.ep

        ...
        
        
        # --- Anropa model_params för en geomtetribeskrivning
        
        geometry = self.model_params.geometry()        
        
        # --- Nätgenerering
        
        el_type = 3        # <-- Fyrnodselement flw2i4e
        dofs_per_node = 1  # <-- Skalärt problem
        
        mesh = cfm.GmshMeshGenerator(geometry)
        mesh.el_size_factor = 0.5     # <-- Anger max area för element
        mesh.el_type = el_type
        mesh.dofs_per_node = dofs_per_node
        mesh.return_boundary_elements = True
        
        coords, edof, dofs, bdofs, elementmarkers, boundaryElements = mesh.create()
``` 
            
Elementgenerering och assemblering behöver inte ändras från arbetsblad 2. Däremot måste hanteringen av laster och randvillkor uppdateras, eftersom vi nu arbetar med lastmarkörer istället för frihetsgrader. Använd funktionerna **applybc(...)** och **applyforcetotal(...)**/**applyTractionLinearElement(...)** som finns i **calfem.utils** modulen.

Lösning av ekvationssystem och elementkraftsberäkning behöver inte heller ändras. Däremot behöver vi lagra den genererade variablerna **coord**, **edof**, **geometry** och andra utdatavariabler som behövs för att visualisera resultaten.

> Det kan också vara bra att definiera en variabel i **ModelParams**-klassen för att ange max storlek på de genererade elementen t ex **el_size_factor**, som sedan kan tilldelas till **cfm.GmshGenerator**-klassens egenskap **el_size_factor**.              

## Användning av den parametriska modellen

Tanken med den parametriska problembeskrivningen är att en användare på ett enkelt sätt skall kunna specificera sitt problem område i koden utan att behöva gå in i modulen för modellen. Detta visas i följande kod:

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

På detta sätt kan man också på ett enkelt sätt studera effekten av t ex öka djupet på sponten i grundvatten och ta reda på hur detta påverkar flödet. T ex:

``` py
# -*- coding: utf-8 -*-

import flowmodel as fm
import numpy as np

if __name__ == "__main__":

    model_params = fm.ModelParams()

    model_params.w = 100.0
    model_params.h = 10.0
    model_params.d = 5.0
    model_params.t = 0.5
    model_params.kx = 20.0
    model_params.ky = 20.0

    model_results = fm.ModelResults()
    
    dRange = np.linspace(3.0, 7.0, 10).tolist()

    model_solver = fm.ModelSolver(model_params, model_results)
    
    for d in dRange:
    
        print("-------------------------------------------")    
        print("Simulating d = ", d)
    
        model_params.d = d        
        solver.execute()
        
        print("Max flow = ", np.max(model_results.maxFlow))        
``` 
## ModelReport-klassen

I **ModelReport**-klassen behöver vi lägga till utskrift för geometribeskrivning, så att denna också kommer med i rapporten.

## Visualisering med ny klass ModelVisualisation

I arbetsblad 2 var det enda resulatet en utskrift från vår **ModelReport**-klass. Att tolka resultat i textform kan vara svårt. Vi skall därför använda av oss CALFEM:rutiner för visualisering i modulen **calfem.vis**. Dessa rutiner bygger på det etablerade visualiseringspaketet **visvis**. Visualiseringen skall implementeras i den nya klassen **ModelVisualisation**. Denna klass har samma indata som **Result**-klassen, dvs referenser till instanser av **model_params** och **ModelResult**. 

Det finns ett antal visualiseringsfunktioner i CALFEM. I detta arbetsblad skall följande visualiseringar implementeras:

 * Geometri - draw_geometry(...)
 * Genererat nät - draw_mesh(...)
 * Deformerat nät - draw_mesh(...) (för spänningsexemplet)
 * Elementvärden - draw_element_values(...)
 * Nodvärden - draw_nodal_values(...)
 
Dokumentation för dessa rutiner finns i användarhandboken för [nätgeneringsrutinerna](http://training.lunarc.lu.se/pluginfile.php/473/mod_resource/content/1/DAE_rapport_draft06.pdf). 

Notera att dessa rutiner är integrerade i CALFEM och visvis behöver inte importeras explicit. Följande kod från manualen:

``` py
vv.figure()
pcv.drawMesh(coords=coords, edof=edof, dofsPerNode=dofsPerNode, elType=elType, filled=True, title="Mesh")
``` 
    
blir istället (med import enligt tidigare):

``` py
cfv.figure()
cfv.draw_mesh(coords=coords, edof=edof, dofs_per_node=dofs_per_node, el_type=el_type, filled=True, title="Mesh")
```
    
Följande kod visar hur klassen kan implementeras med visualiering av geometrin.

``` py
class ModelVisualisation:
    def __init__(self, model_params, model_result):
        self.model_params = model_params
        self.model_result = model_result
        
    def show(self):
        
        geometry = self.model_result.geometry
        a = self.model_result.a
        max_flow = self.model_result.max_flow
        coords = self.model_result.coords
        edof = self.model_result.edof
        dofs_per_node = self.model_result.dofsPerNode
        el_type = self.model_result.elType
        
        cfv.figure() 
        cfv.draw_geometry(geometry, title="Geometry")
        
        ...
                    
    def wait(self):
        """Denna metod ser till att fönstren hålls uppdaterade och kommer att returnera
        När sista fönstret stängs"""

        cfv.show_and_wait()
```
            
**ModelVisualisation**-klassen läggs sedan till i huvudprogrammet som koden nedan visar:

``` py
# -*- coding: utf-8 -*-

import flowmodel as fm

if __name__ == "__main__":
    
    model_params = fm.model_params()

    model_result = fm.OutputData()

    solver = fm.Solver(model_params, model_result)
    solver.execute()

    report = fm.Report(model_params, model_result)
    print(report)
    
    vis = fm.Visualisation(model_params, model_result)
    vis.show()
    vis.wait()
``` 

**vis.wait()** måste anropas sist i huvudprogrammet eftersom denna funktion inte returnerar förrän sista visualiseringsfönstret stängs.

## Inlämning och redovisning

Det som skall göras i detta arbetsblad är:

 * Ändra **model_params**-klassen så att den beskriver problemet parametriskt enligt de beskrivna exemplen sist i arbetsbladet. Skapa en metod **geometry()** som returnerar en **cfg.Geometry**-instans med geometri definierad utifrån parameterbeskrivningen. 
 * Uppdatera **ModelSolver**-klassen så att denna skapar ett elementnät med hjälp av **cfm.GmshMeshGenerator** klassen. Lagra också maxflöden/maxspänningar (von Mises) och lagra dessa i udata-klassen.
 * Slutföra implementeringen av **ModelVisualisation**-klassen så att den kan hantera visualisera geometri, elementnät, elementflöden och nodvärden.
 * Gör en parameter-studie där en av parametrarna varieras och maxflöde/maxspänning plottas i förhållande till den valda parametern. Skapa ett nytt huvudprogram för parameterstudien.

Inlämningen skall bestå av en zip-fil (eller annat arkivformat) bestående av: 

 * Alla Python-filer. (.py-filer)
 * Ett exempel på en sparad json-fil.
 * Utskrift från programkörning.
 * Utskrift från parameterstudien.
 
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



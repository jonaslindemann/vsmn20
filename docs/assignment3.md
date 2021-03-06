# Programutveckling för Tekniska Tillämpningar - Arbetsblad 3

> **Att tänka på:** När **...** visas i programexemplen anger detta att det saknas kod som ni själva måste lägga till. Variabler och datastrukturer är bara exempel. Beroende på problemtyp kan man behöva andra datastrukturer än de som är beskrivna i kodexemplen.

## Allmänt

I detta arbetsblad innehåller följande moment:

 1.	Klasserna implementerade i arbetsblad 2 skall anpassas för att hantera en geometrisk model som beskriver beräkningsmodellen parametriskt.
 2. Inläsningsrutiner måste anpassas för att hantera den geometriska modellen.
 2.	Nätgenerering skall implementeras med hjälp av **GMSH** och resultat visualiseras.
 3.	Visualisering av modellen skall göras med rutinerna i **calfem.vis**.

## Geometrisk modell

I detta arbetsblad skall vi uppdatera våra klasser för att hantera en geometrisk modell som ligger till grund för nätgenereringen i vår **Solver**-klass. Istället för att definiera modellen med avseende på noder och element definierar vi den istället med parametrar som bredd, höjd och  

I detta arbetsblad skall vi använda lite fler CALFEM moduler Lägg till följande moduler i modulen där era klasser är definierade:

    import calfem.core as cfc
    import calfem.geometry as cfg  # <-- Geometrirutiner
    import calfem.mesh as cfm      # <-- Nätgenerering
    import calfem.vis_mpl as cfv       # <-- Visualisering
    import calfem.utils as cfu     # <-- Blandade rutiner
    
## Uppdatering av input_data-klassen

**input_data**-klassen måset uppdateras för att hantera en parametriskt beskriven modell istället för att definiera problemet på elementnivå. För grundvattenproblemet har t ex parametrarna, **w**, **t**, **d** och **h** för att beskriva geometrin och **ep**, **kx** och **ky** för att beskriva element egenskaper. Variabler för laster och randvillkor kan bibehållas. Skillnaden nu är att vi istället för frihetsgrad och värde lagrar randvillkorsmarkör och värde istället.

Alla variabler som t ex **coord** och **edof** som beskriver indata på elementnivå skall flyttas till **Solver**-klassen eftersom dessa kommer att skapas när elementnätet genereras.

Geometrin kommer att definieras med objekt och funktioner från **calfem.geometry**. För att **Solver**-klassen alltid skall få en uppdaterad geometri vid anropet av **execute()**-metoden skapar vi en metod i **input_data**, **geometry()** som skall returnera en **Geometry** instans utifrån värden i de indata variabler som vi definierade för geometrin. Denna metod anropas sedan från **execute()**, så att lösaren alltid får en uppdaterad geometribeskrivning även om indata variablern har ändrats. Ett exempel på hur denna metod kan se ut visas i följande exempel:

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

I **Solver**-klassen måste vi lägga till anrop till nätgenereraren i **calfem.mesh**, **GmshMeshGenerate**. Denna kommer att ge oss element koordinater, topologi samt variabler som kan användas för koppling mellan geometri och elementnät. Ett exempel på hur detta kan se ut i **execute()**-metodn visas nedan:

    class Solver(object):
        """Klass för att hantera lösningen av vår beräkningsmodell."""

        ...
                    
        def execute(self):
            """Metod för att utföra finita element beräkningen."""
            
            # --- Överför modell variabler till lokala referenser
            
            version = self.input_data.version
            ep = self.input_data.ep

            ...
            
            
            # --- Anropa input_data för en geomtetribeskrivning
            
            geometry = self.input_data.geometry()        
            
            # --- Nätgenerering
            
            el_type = 3        # <-- Fyrnodselement flw2i4e
            dofs_per_node = 1  # <-- Skalärt problem
            
            mesh = cfm.GmshMeshGenerator(geometry)
            mesh.el_size_factor = 0.5     # <-- Anger max area för element
            mesh.el_type = el_type
            mesh.dofs_per_node = dofs_per_node
            mesh.return_boundary_elements = True
            
            coords, edof, dofs, bdofs, elementmarkers, boundaryElements = mesh.create()
            
Elementgenerering och assemblering behöver inte ändras från arbetsblad 2. Däremot måste hanteringen av laster och randvillkor uppdateras, eftersom vi nu arbetar med lastmarkörer istället för frihetsgrader. Använd funktionerna **applybc(...)** och **applyforcetotal(...)**/**applyTractionLinearElement(...)** som finns i **calfem.utils** modulen.

Lösning av ekvationssystem och elementkraftsberäkning behöver inte heller ändras. Däremot behöver vi lagra den genererade variablerna **coord**, **edof**, **geometry** och andra utdatavariabler som behövs för att visualisera resultaten.

> Det kan också vara bra att definiera en variabel i **InputData**-klassen för att ange max storlek på de genererade elementen t ex **el_size_factor**, som sedan kan tilldelas till **cfm.GmshGenerator**-klassens egenskap **el_size_factor**.              

## Användning av den parametriska modellen

Tanken med den parametriska problembeskrivningen är att en användare på ett enkelt sätt skall kunna specificera sitt problem område i koden utan att behöva gå in i modulen för modellen. Detta visas i följande kod:

    # -*- coding: utf-8 -*-

    import flowmodel as fm

    if __name__ == "__main__":
        
        input_data = fm.InputData()

        input_data.w = 100.0
        input_data.h = 10.0
        input_data.d = 5.0
        input_data.t = 0.5
        input_data.kx = 20.0
        input_data.ky = 20.0
        
        ...

På detta sätt kan man också på ett enkelt sätt studera effekten av t ex öka djupet på sponten i grundvatten och ta reda på hur detta påverkar flödet. T ex:

    # -*- coding: utf-8 -*-

    import flowmodel as fm
    import numpy as np

    if __name__ == "__main__":
        
        dRange = np.linspace(3.0, 7.0, 10).tolist()
        
        for d in dRange:
        
            print("-------------------------------------------")    
            print("Simulating d = ", d)
        
            input_data = fm.InputData()
        
            input_data.w = 100.0
            input_data.h = 10.0
            input_data.d = d
            input_data.t = 0.5
            input_data.kx = 20.0
            input_data.ky = 20.0
            
            output_data = fm.OutputData()
        
            solver = fm.Solver(input_data, output_data)
            solver.execute()
            
            print("Max flow = ", np.max(output_data.maxFlow))        


## Report-klassen

I **Report**-klassen behöver vi lägga till utskrift för geometribeskrivning, så att denna också kommer med. 

## Visualisering med ny klass Visualisation

I arbetsblad 2 var det enda resulatet en utskrift från vår **Report**-klass. Att tolka resultat i textform kan vara svårt. Vi skall därför använda av oss CALFEM:rutiner för visualisering i modulen **calfem.vis**. Dessa rutiner bygger på det etablerade visualiseringspaketet **visvis**. Visualiseringen skall implementeras i den nya klassen **Visualisation**. Denna klass har samma indata som **Result**-klassen, dvs referenser till instanser av **input_data** och **OutputData**. 

Det finns ett antal visualiseringsfunktioner i CALFEM. I detta arbetsblad skall följande visualiseringar implementeras:

 * Geometri - draw_geometry(...)
 * Genererat nät - draw_mesh(...)
 * Deformerat nät - draw_mesh(...) (för spänningsexemplet)
 * Elementvärden - draw_element_values(...)
 * Nodvärden - draw_nodal_values(...)
 
Dokumentation för dessa rutiner finns i användarhandboken för [nätgeneringsrutinerna](http://training.lunarc.lu.se/pluginfile.php/473/mod_resource/content/1/DAE_rapport_draft06.pdf). 

Notera att dessa rutiner är integrerade i CALFEM och visvis behöver inte importeras explicit. Följande kod från manualen:

    vv.figure()
    pcv.drawMesh(coords=coords, edof=edof, dofsPerNode=dofsPerNode, elType=elType, filled=True, title="Mesh")
    
blir istället (med import enligt tidigare):

    cfv.figure()
    cfv.draw_mesh(coords=coords, edof=edof, dofs_per_node=dofs_per_node, el_type=el_type, filled=True, title="Mesh")
    
Följande kod visar hur klassen kan implementeras med visualiering av geometrin.

    class Visualisation(object):
        def __init__(self, input_data, output_data):
            self.input_data = input_data
            self.output_data = output_data
            
        def show(self):
            
            geometry = self.output_data.geometry
            a = self.output_data.a
            max_flow = self.output_data.max_flow
            coords = self.output_data.coords
            edof = self.output_data.edof
            dofs_per_node = self.output_data.dofsPerNode
            el_type = self.output_data.elType
            
            cfv.figure() 
            cfv.draw_geometry(geometry, title="Geometry")
            
            ...
                       
        def wait(self):
            """Denna metod ser till att fönstren hålls uppdaterade och kommer att returnera
            När sista fönstret stängs"""

            cfv.show_and_wait()
            
**Visualisation**-klassen läggs sedan till i huvudprogrammet som koden nedan visar:

    # -*- coding: utf-8 -*-

    import flowmodel as fm

    if __name__ == "__main__":
        
        input_data = fm.input_data()

        output_data = fm.OutputData()

        solver = fm.Solver(input_data, output_data)
        solver.execute()

        report = fm.Report(input_data, output_data)
        print(report)
        
        vis = fm.Visualisation(input_data, output_data)
        vis.show()
        vis.wait()        

**vis.wait()** måste anropas sist i huvudprogrammet eftersom denna funktion inte returnerar förrän sista visualiseringsfönstret stängs.

## Inlämning och redovisning

Det som skall göras i detta arbetsblad är:

 * Ändra **input_data**-klassen så att den beskriver problemet parametriskt enligt de beskrivna exemplen sist i arbetsbladet. Skapa en metod **geometry()** som returnerar en **cfg.Geometry**-instans med geometri definierad utifrån parameterbeskrivningen. 
 * Uppdatera **Solver**-klassen så att denna skapar ett elementnät med hjälp av **cfm.GmshMeshGenerator** klassen. Lagra också maxflöden/maxspänningar (von Mises) och lagra dessa i udata-klassen.
 * Slutföra implementeringen av **Visualisation**-klassen så att den kan hantera visualisera geometri, elementnät, elementflöden och nodvärden.
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



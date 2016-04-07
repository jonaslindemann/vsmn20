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
    import calfem.vis as cfv       # <-- Visualisering
    import calfem.utils as cfu     # <-- Blandade rutiner
    
## Uppdatering av InputData-klassen

**InputData**-klassen måset uppdateras för att hantera en parametriskt beskriven modell istället för att definiera problemet på elementnivå. För grundvattenproblemet har t ex parametrarna, **w**, **t**, **d** och **h** för att beskriva geometrin och **ep**, **kx** och **ky** för att beskriva element egenskaper. Variabler för laster och randvillkor kan bibehållas. Skillnaden nu är att vi istället för frihetsgrad och värde lagrar randvillkorsmarkör och värde istället.

Alla variabler som t ex **coord** och **edof** som beskriver indata på elementnivå skall flyttas till **Solver**-klassen eftersom dessa kommer att skapas när elementnätet genereras.

Geometrin kommer att definieras med objekt och funktioner från **calfem.geometry**. För att **Solver**-klassen alltid skall få en uppdaterad geometri vid anropet av **execute()**-metoden skapar vi en metod i **InputData**, **geometry()** som skall returnera en **Geometry** instans utifrån värden i de indata variabler som vi definierade för geometrin. Denna metod anropas sedan från **execute()**, så att lösaren alltid får en uppdaterad geometribeskrivning även om indata variablern har ändrats. Ett exempel på hur denna metod kan se ut visas i följande exempel:

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
            
            version = self.inputData.version
            ep = self.inputData.ep

            ...
            
            
            # --- Anropa InputData för en geomtetribeskrivning
            
            geometry = self.inputData.geometry()        
            
            # --- Nätgenerering
            
            elType = 3      # <-- Fyrnodselement flw2i4e
            dofsPerNode= 1  # <-- Skalärt problem
            
            meshGen = cfm.GmshMeshGenerator(geometry)
            meshGen.elSizeFactor = 0.5     # <-- Anger max area för element
            meshGen.elType = elType
            meshGen.dofsPerNode = dofsPerNode
            
            coords, edof, dofs, bdofs, elementmarkers = meshGen.create()
            
Elementgenerering och assemblering behöver inte ändras från arbetsblad 2. Däremot måste hanteringen av laster och randvillkor uppdateras, eftersom vi nu arbetar med lastmarkörer istället för frihetsgrader. Använd funktionerna **applybc(...)** och **applyforcetotal(...)** som finns i **calfem.utils** modulen.

Lösning av ekvationssystem och elementkraftsberäkning behöver inte heller ändras. Däremot behöver vi lagra den genererade variablerna **coord**, **edof**, **geometry** och andra utdatavariabler som behövs för att visualisera resultaten.

## Report-klassen

I **Report**-klassen behöver vi lägga till utskrift för geometribeskrivning, så att denna också kommer med. 

## Visualisering med ny klass Visualisation

I arbetsblad 2 var det enda resulatet en utskrift från vår **Report**-klass. Att tolka resultat i textform kan vara svårt. Vi skall därför använda av oss CALFEM:rutiner för visualisering i modulen **calfem.vis**. Dessa rutiner bygger på det etablerade visualiseringspaketet **visvis**. Visualiseringen skall implementeras i den nya klassen **Visualisation**. Denna klass har samma indata som **Result**-klassen, dvs referenser till instanser av **InputData** och **OutputData**. 

Det finns ett antal visualiseringsfunktioner i CALFEM. I detta arbetsblad skall följande visualiseringar implementeras:

 * Geometri - drawGeometry(...)
 * Genererat nät - drawMesh(...)
 * Deformerat nät - drawMesh(...) (för spänningsexemplet)
 * Elementvärden - drawElementValues(...)
 * Nodvärden - drawNodalValues(...)
 
Dokumentation för dessa rutiner finns i användarhandboken för nätgeneringsrutinerna i 

http://training.lunarc.lu.se/pluginfile.php/473/mod_resource/content/1/DAE_rapport_draft06.pdf

Notera att dessa rutiner är integrerade i CALFEM och visvis behöver inte importeras explicit. Följande kod från manualen:

    vv.figure()
    pcv.drawMesh(coords=coords, edof=edof, dofsPerNode=dofsPerNode, elType=elType, filled=True, title="Mesh")
    
blir istället (med import enligt tidigare):

    cfv.figure()
    cfv.drawMesh(coords=coords, edof=edof, dofsPerNode=dofsPerNode, elType=elType, filled=True, title="Mesh")
    


** ---- ARBETSBLADET ÄR UNDER KONSTRUKTION --- **

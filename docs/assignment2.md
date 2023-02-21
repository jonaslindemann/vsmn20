# Arbetsblad 2

!!! note "Viktigt"

    När **...** visas i programexemplen anger detta att det saknas kod som ni själva måste lägga till. Variabler och datastrukturer är bara exempel. Beroende på problemtyp kan man behöva andra datastrukturer än de som är beskrivna i kodexemplen.

## Strukturering av datorprogram

Utveckling av datorprogram är ett tidskrävande arbete. Det är därför viktigt att programkoden utformas på sådant sätt att modifiering och vidareutveckling underlättas. Detta innebär att program-met skall vara välstrukturerat och ha god läsbarhet. En beräkningsprocess kan i regel delas in i olika logiska delar. Vid programmeringen delas programmet lämpligen upp i subrutiner. Överföring av data görs via parametrarna i deklarationen och anropet. Valet av parametrar ska vara genomtänkt, så att överflödiga parametrar inte tas med. Överföring av parametrar med globala variabler bör undvikas, eftersom dessa ofta medför att parametrar som inte används i aktuellt underprogram kommer med, vilket försämrar överskådligheten.

För att ge programmet god läsbarhet bör valet av namn på variabler och subrutiner göras med omsorg, så att namnet säger något om funktionen. För att ytterligare öka läsbarheten används kommentarsatser för att i varje subrutin beskriva syfte, parametrarnas innebörd och de olika beräk-ningsstegen. Om ett program utformas enligt ovan blir det enkelt att modifiera det eller att utnyttja delar av det i andra sammanhang än det ursprungliga.

## Utformning av finita element program

En finita element analys kan oftast delas in i tre olika steg:

 1.	Inmatning av indata
 1.	Beräkning
 1.	Presentation av resultat

Varje sådant steg utförs ofta med hjälp av ett separat program. Data överförs då mellan de olika programmen med hjälp av filer. Ibland väljer man att, istället för att dela upp analysen i tre separata program, låta två eller alla tre stegen utföras av samma program.

I början av kursen väljer vi att inrikta oss på steg 2 enligt ovan. Enklare indata definieras direkt i Python för att förenkla felsökning vid implementeringen av beräkningsdelen.

I denna inlämning skall stommen till beräkningsprogrammet skapas. En av problemtyperna:

 * Värmeledning
 * Grundvattenströmning
 * Spänningsberäkning (Plan spänning)
 
skall väljas. Det valda problemet kommer sedan att användas för alla kommande arbetsblad. Ni behöver alltså inte implementera ett program som hanterar alla problemtyperna.

Vi börjar med att definiera en python-modul, t ex flowmodel.py, vilken skall innehålla beräkningsdelen av programmet. Ge filen ett namn som passar det problemområde ni valt. I början av filen anges följande deklarationer för de moduler som skall användas:

``` py
# -*- coding: utf-8 -*-

import numpy as np
import calfem.core as cfc
```
    
För implementeringen kommer vi att använda objekt-orienterad programmering, dvs dela upp programmet i logiska objekt som kan återanvändas och möjliggöra utvidgning och återanvändning av koden. 4 logiska klasser kan identifieras för beräkningsmodellen:

  * **ModelParams** - Lagrar de in-variabler som behövs för beräkningen.
  * **ModelSolver** - Implementerar lösningsrutinen för problemet.
  * **ModelResult** - Lagra de resultat som genereras vis lösningen av problemet.
  * **ModelReport** - Hanterar genereringen av indata- och utdatarapporter för programmet.
 
## Klassen ModelParams

Klassen ModelParams skall innehålla all indata som behövs för att utföra beräkningen. En första klassdefinition kan se ut som i följande kod:

``` py
# -*- coding: utf-8 -*-

import numpy as np
import calfem.core as cfc
import json

class ModelParams:
    """Klass för att definiera indata för vår modell."""
    def __init__(self):
        
        self.version = 1
        
        self.t = 1
        self.ep = [self.t]

        # --- Elementegenskaper
        
        ...
        
        # --- Skapa indata för testexempel
        
        self.coord = np.array([
            [0.0, 0.0],
            [0.0, 0.12],
            ...
            [0.24, 0.12]
        ])

        # --- Elementtopolgi
            
        self.edof = np.array([
            [...],
            ...
            [...]
        ])

        # --- Laster

        self.loads = [
            [5, 6.0],
            [6, 6.0]
        ]

        # --- Randvillkor

        self.bcs = [
            [1, -15.0],
            [2, -15.0]
        ]
```
 
!!! note "Viktigt"
    Alla **...** anger att kod måste läggas till.
 
## Klassen ModelResult

**ModelResult**-klassen skall användas för att lagra resultaten som skapas under beräkningen. Eftersom Python är ett dynamiskt språk kan beräkningsklassen själv lägga till resultatvariablerna i resultatobjektet, men det är alltid bra att skapa tomma variabler i klassen, så att den kan fungera självständigt. Ett exempel på resultat-klass visas i följande kod:

``` py
class ModelResult:
    """Klass för att lagra resultaten från beräkningen."""
    def __init__(self):
        self.a = None
        self.r = None
        self.ed = None
        self.qs = None
        self.qt = None
```
            
!!! note "Bra att veta"
    **None** kan användas som en generisk datatyp som kan användas för att kunna skapa variabler utan innehåll. Det är också möjligt att testa om en variabel är tilldelad genom en if-sats:

        if self.a == None:
            self.a = np.array(...) # Tilldela en riktigt datatyp om self.a == None 
    
## Klassen ModelSolver

Klassen **ModelSolver** är ansvarig för att utföra själva beräkningen. Klassen kommer att ha en kontruktor, __init__(...) och en metod execute() för att utföra själva beräkningen. Kontruktorn skall ha två inparametrar, **model_params** och **model_results**, vilka är instanser av klasserna **ModelParams** och **ModelResult**. Konstruktorn får följande utseende:

``` py
class ModelSolver:
    """Klass för att hantera lösningen av vår beräkningsmodell."""
    def __init__(self, model_params, model_result):
        self.model_params = model_params
        self.model_result = model_result
```

Inparametrarna tilldelas två klassvariabler, **self.model_params** och **self.model_result**.

Själva beräkningen skall utföras i metoden execute(...). Metoden hämtar indata från **self.model_params** för att ställa upp och utföra finita element beräkningen precis som ett vanliga CALFEM-program. För att förenkla hanteringen av invariabler kan lokala referenser till indata skapas enligt som visas i följande kod:

``` py
class ModelSolver:

    ...
        
    def execute(self):
        
        # --- Överför modell variabler till lokala referenser
        
        edof = self.model_params.edof
        cond = self.model_params.cond
        coord = self.model_params.coord
        dof = self.model_params.dof
        ep = self.model_params.ep
        loads = self.model_params.loads
        bcs = self.model_params.bcs       
```

Eftersom Python hanterar alla variabler som referenser är det ingen nackdel att göra dessa tilldelningar. Inga kopior på data kommer att göras. Variablerna behöver inte heller kopieras tillbaka till **self.model_params** då de lokala variablerna pekar på samma minnesinnehåll.

När beräkningen har genomförts skall **self.model_result** tilldelas referenserna till resultatet av beräkningen. Den globala styvhetsmatrisen eller f-vektorn behöver inte lagras här. Intressanta variabler är förskjutningsvektorn, reaktionskraftsvektorn och elementkrafter. Följande kod visar hur det kan se ut:

``` py
class ModelSolver:

    ...
        
    def execute(self):
    
        # --- Överför modell variabler till lokala referenser
        
        ...

        ... Beräkningskod ...
        
        # --- Överför modell variabler till lokala referenser

        self.model_result.a = a
        self.model_result.r = r
        self.model_result.ed = ed
        self.model_result.qs = qs
        self.model_result.qt = qt
```
            
## Klassen ModelReport

När beräkingen är klar skall en rapport över indata och resultat genereras, detta görs av klassen **ModelReport**. Klassen kommer att ha samma indata som **ModelSolver**. 

För generering av rapporten kommer vi att använda Python:s inbyggda metod **__str__()**. Denna metod används för att implementera vad som skall ske när man använder **print(...)** på en instans av klassen eller när funktionen **str(...)** används för att konvertera klassens innehåll till en sträng. 

För att detta skall fungera behöver vi 2 extra metoder och en strängvariabel. Strängvariabeln kommer vi att fylla med textbeskrivningen av klassen. De extra metoderna används för att rensa och fylla strängvariabeln på innehåll. Följande kod visas implementeringen av **ModelReport** klassen:

``` py
class ModelReport:
    """Klass för presentation av indata och utdata i rapportform."""
    def __init__(self, model_params, output_data):
        self.model_params = model_params
        self.model_result = output_data
        self.report = ""
        
    def clear(self):
        self.report = ""
        
    def add_text(self, text=""):
        self.report+=str(text)+"\n"
                
    def __str__(self):
        self.clear()
        self.add_text()
        self.add_text("-------------- Model input ----------------------------------")
        ...
        self.add_text("Coordinates:")
        self.add_text()
        self.add_text(self.model_params.coord)
        ...
        return self.report
```
                           
## Huvudprogram

För att programmet skall fungera behöver vi ett huvudprogram. Följande kod visar hur man ofta brukar definiera ett huvudprogram i Python:

``` py
# -*- coding: utf-8 -*-

if __name__ == "__main__":
    print("Denna fil exekveras direkt och importeras inte.")
```

Om en Python-fil importeras med **import**-satsen kommer variabeln **__name__** innehålla modulens namn, dvs namnet på källkodsfilen. Om man däremot anropar filen direkt med en Python-tolk kommer **__name__** innehålla värdet **__main__**. På detta sätt kan man se till att bara viss kod utförs om filen startas direkt med Python-tolken och på samma gång använda filen som en modul. Detta koncept används ofta i Python för att skapa test-funktioner för moduler. 

Ett huvudprogram för vårt finita element program som använder alla våra klasser kan då se ut så här:

``` py
    # -*- coding: utf-8 -*-

    import flowmodel as fm

    if __name__ == "__main__":
        
        model_params = fm.ModelParams()
        model_result = fm.ModelResult()

        model_solver = fm.ModelSolver(model_params, output_data)
        model_solver.execute()

        model_report = fm.ModelReport(model_params, output_data)
        print(model_report)
```
        
I ovanstående huvudprogram importerar vi modulen **flowmodel** (flowmodel.py) som definierar våra klasser. Vi importerar den i namnrymden **fm**. Vi instantierar **ModelParams** och **ModelResult** objekt för att hantera in- och utdata. En **ModelSolver** instans, **model_solver** instantieras med objekten, **model_params** och **model_result** som indata. Beräkningen startas sedan genom att vi anropar metoder **model_solver.execute()**. Programmet avslutas med att vi skapar en instans av **ModelReport** som vi sedan skriver ut på skärmen med en **print()**-sats. 

Exempel på en körning av programmet visas nedan:

    Solving equation system...
    Computing element forces...

    -------------- Model input ----------------------------------

    t = 1

    Conductivty:

    [[ 1.7   1.7 ]
     [ 1.7   1.7 ]
     [ 0.04  0.04]
     [ 0.04  0.04]]

    Coordinates:

    [[ 0.    0.  ]
     [ 0.    0.12]
     [ 0.12  0.  ]
     [ 0.12  0.12]
     [ 0.24  0.  ]
     [ 0.24  0.12]]

    Coordinate dofs:

    [[1]
     [2]
     [3]
     [4]
     [5]
     [6]]

    Topology:

    [[1 4 2]
     [1 3 4]
     [3 6 4]
     [3 5 6]]

    Element coordinates X

    [[ 0.    0.12  0.  ]
     [ 0.    0.12  0.12]
     [ 0.12  0.24  0.12]
     [ 0.12  0.24  0.24]]

    Element coordinates Y

    [[ 0.    0.12  0.12]
     [ 0.    0.    0.12]
     [ 0.    0.12  0.12]
     [ 0.    0.    0.12]]

    -------------- Results --------------------------------------

    Displacements:

    [[ -15.        ]
     [ -15.        ]
     [  -7.94117647]
     [  -7.94117647]
     [ 292.05882353]
     [ 292.05882353]]

    Reactions:

    [[ -6.00000000e+00]
     [ -6.00000000e+00]
     [ -3.55271368e-15]
     [ -1.02829510e-15]
     [ -1.77635684e-15]
     [ -8.88178420e-16]]

    Element forces:

    [[-100.   0.]
     [-100.   0.]
     [-100.   0.]
     [-100.   0.]]   
     
## Spara och läsa in från fil med JSON

En viktig del av ett beräkningsprogram är att kunna läsa och skriva indata till filer. I de flest fall är problemen så stora att de inte kan definieras i programkod. I detta program kommer vi att använda JSON (Javascript Object Notation) som format på de filer vi kommer att skriva. Följande kod är ett exempel på hur en JSON fil kan se ut. 

``` json
{"employees":[
    {"firstName":"John", "lastName":"Doe"},
    {"firstName":"Anna", "lastName":"Smith"},
    {"firstName":"Peter", "lastName":"Jones"}
]}
```
    
Filen påminner mycket om den syntax som Python använder för dictionaries. Vad som gör formatet attraktivt är att vi inte själva behöver kunna skriva det till fil utan Python har ett inbyggt bibliotek för att både läsa och skriva filer av denna typ. 

För att implementera funktionaliteten läsa och skriva i vårt program lägger vi förs till följande **import**-sats i början på vår modulfil:

``` py hl_lines="5"
# -*- coding: utf-8 -*-

import numpy as np
import calfem.core as cfc
import json 
```
    
Vi börjar med att implementera skrivning till fil, eftersom vi har alla indata för detta ändamål. Vi lägger till en metod, **save(...)** i **model_params**-klassen:

``` py
class ModelParams:
    """Klass för att definiera indata för vår modell."""
    
    ...

    def save(self, filename):
        """Spara indata till fil."""
        
        ...
```

För att definiera strukturen av det som skall lagras, men också för att göra det enkelt att läsa in data, kommer vi att utgå från ett dictionary som bas för det som skall skrivas och läsas till fil. I metoden **save(...)** lägger vi till följande kod för att definiera grundstommen till vår datastruktur:

``` py
        model_params = {}
        model_params["version"] = self.version
        model_params["t"] = self.t
        model_params["ep"] = self.ep
```
                        
**model_params["version"]** kan vara bra att ha för att hålla koll på vilken version av filformatet man använder när man väl läser tillbaka filen.

Ett problem vi måste hantera är att JSON modulen i Python inte kan hantera Numpy-arrayer. Detta löses dock enkelt genom att vi konverterar våra Numpy-arrayer till listor. I följande kod konverterar vi arrayen **self.coord** till en lista med metoden **.tolist(...)**.

``` py
        model_params["coord"] = self.coord.tolist()
```
            
När **model_params** är definierad kan vi öppna en fil för skrivning och sedan skriva ut får JSON-fil med funktionen **json.dumps(...)**

``` py
        with open(filename, "w") as ofile:
            json.dump(model_params, ofile, sort_keys = True, indent = 4)
```

**sort_keys** och **indent** ser till att den skrivna filen blir snyggt formaterad.

För att läsa in en befintlig JSON-fil gör vi i omvänd ordning. Vi läser in hela filen till en teckensträng som vi sedan konverterar tillbaka till ett dictionary med funktionen **json.load(...)**

``` py
def load(self, filename):
    """Läs indata från fil."""
    
    ifile = open(filename, "r")
    model_params = json.load(ifile)
    ifile.close()

    self.version = model_params["version"]
    self.t = model_params["t"]
    self.ep = model_params["ep"]
```

För vår coord-array måste nu konvertera tillbaka denna till en Numpy-array. Detta görs med Numpy-funktionen **np.asarray(...)**            

``` py
    self.coord = np.asarray(model_params["coord"])
```
        
De kompletta skriv- och läsfunktionerna blir då:

``` py
class ModelParams:
    """Klass för att definiera indata för vår modell."""

    def save(self, filename):
        """Spara indata till fil."""

        model_params = {}
        model_params["version"] = self.version
        model_params["t"] = self.t
        model_params["ep"] = self.ep
        ...
        model_params["coord"] = self.coord.tolist()
        ...

        with open(filename, "w") as ofile:
            json.dump(model_params, ofile, sort_keys = True, indent = 4)

    def load(self, filename):
        """Läs indata från fil."""

        with open(filename, "r") as ifile:
            model_params = json.load(ifile)

        self.version = model_params["version"]
        self.t = model_params["t"]
        self.ep = model_params["ep"]
        ...
        self.coord = np.asarray(model_params["coord"])
        ...
```

## Inlämning och redovisning

Det som skall göras i detta arbetsblad är:

 * Slutföra implementeringen av **ModelParams**-klassen med all indata som krävs för att kunna lösa det valda problemet. Rutiner för att spara och läsa från JSON-filer skall också implementeras för alla indatavariabler.
 * Slutföra implementeringen av **ModelSolver**-klassen med en finita element lösare implementerad med de metoder som finns beskriva i CALFEM.
 * Slutföra implementeringen av **ModelReport**-klassen med en komplett utskrift av indata- och utdata variabler med beskrivande texter.

Inlämningen skall bestå av en zip-fil (eller annat arkivformat) bestående av: 

 * Alla Python-filer. (.py-filer)
 * Ett exempel på en sparad json-fil.
 * Utskrift från programkörning.
 * Jämförande beräkning i CALFEM för MATLAB.

## Elementtyper

### Plan skiva

![eltype1](images/eltype1.png)

 * u = förskjutning
 * x1,y1,x2,y2,x3,y3 = koordinater
 * E = elasticitetsmodul
 * v = Poissons tal
 * CALFEM för Python element : **plante/plants**

### Tvådimensionell värmeledning

![eltype2](images/eltype2.png)

 * T = temperatur
 * x1,y1,x2,y2,x3,y3 = koordinater
 * lambda,x, lambda,y = värmekonduktivitet
 * CALFEM för Python element : **flw2te/flw2te**

### Grundvattenströmning

![eltype3](images/eltype3.png)

 * phi = tryckhöjd
 * x1,y1,x2,y2,x3,y3 = koordinater
 * kx, ky = permeabiliteter
 * CALFEM för Python element : **flw2te/flw2ts**

## Testexempel

### Plan skiva

![case1](images/case1.png)

 * E = 20.8 GPa
 * t = 0.01 m
 * R_6,2 = -10.0 kN
 * u_1,1 = u_1,2 = u_2,1 = u_2,2 = 0.0 
 * Plan spänning
 * u_i,j  där i är nodnummer och j lokal frihetsgrad

### Tvådimensionell värmeledning

![case2](images/case2.png)

 * lambda_x1 = lambda_y1 = 1.7 W/mC
 * lambda_x2 = lambda_y2 = 0.04 W/mC
 * q_5 = q_6 = 6.0 W/m
 * T_1 = T_2 = -15.0 0C
 * t = 0.2 m

### Grundvattenströmning

![case3](images/case3.png)

 * k_x = k_y = 50 m/dag
 * q_6 = -400 m^2/dag 
 * phi_2 = phi_4 = 60.0 m
 * t = 1.0 m


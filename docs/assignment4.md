# Arbetsblad 4


!!! note "Viktigt"
    
    När **...** visas i programexemplen anger detta att det saknas kod som ni själva måste lägga till. Variabler och datastrukturer är bara exempel. Beroende på problemtyp kan man behöva andra datastrukturer än de som är beskrivna i kodexemplen.

## Allmänt

I detta arbetsblad innehåller följande moment:

 1. Skapa gränssnitt i Qt Designer.
 1. Skapa ett huvudprogram och klass för grafiskt gränssnitt.
 1. Skapa en trådklass för att kunna hantera beräkningar i bakgrunden.
 1. Uppdatera visualiseringsklassen för att visa enstaka fönster, samt att kunna stänga alla öppnade fönster.
 
## Grafiskt gränssnitt i Qt Designer
 
Det grafiska gränssnittet skapas i programmet Qt Designer. I detta program skapas en beskrivning av gränssnittet i XML som kommer att läsas in av vårt program. 

Qt Designer kan startas direkt från Spyder genom att klicka på **Tools/External tools/Qt Designer** i menyn. Det går också att starta Qt Designer från kommandoprompten genom att ange kommandot **designer**. Programmet ser ut som i följande figur:

![qt_designer_1](images/qt_designer1.png)

När programmet startats visas en dialogruta för att välja vilken typ av formulär som skall skapas. För vårt huvudfönster väljer vi **Main Window** och klickar på **Create**.

Mitt i fönstret böde det nu finnas ett fönster med namnet **MainWindow - untitled** enligt följande figur:

![qt_designer_1](images/qt_designer2.png)

Qt Designer är uppdelat i 5 huvudsakliga vyer:

 * **Widget Box** till vänster i fönstret visar alla tillgängliga kontroller vi kan använda för att designa gränssnittet.
 * **Designytan** mitten i fönstret används för att välja och redigera de skapade kontrollerna.
 * **Object Inspector** visar ett hierarkiskt träd över alla kontroller i projektet. Detta träd motsvarar också den objektstruktur som kommer att skapas när beskrivningen läses in i Python-koden.
 * **Property Editor** visar egenskaperna för en vald kontroll i gränssnittet. Här kan utseende och andra egenskaper för kontrollerna styras.
 * **Signal/Slot Editor** Hanterar hur kontrollerna rent händelsemässigt är sammankopplade. I denna uppgift kommer vi _inte_  att använda denna, då denna sammankoppling kommer att göras i vår Python-kod.
 
### Menyrad

Det första vi skall göra är att skapa en menyrad till vårt program. I menyraden lägger vi funktioner som:

 1. Skapa ny modell - **File/New**
 1. Öppna existerande modell - **File/Open**
 1. Spara modell - **File/Save**
 1. Spara modellen med ett annat filnamn - **File/Save as...**
 1. Avsluta programmet - **File/Exit**
 1. Starta en beräkning - **Calc/Execute**
 
Menyer skapas genom att klicka på **Type here** i huvudfönstret och ange namnet på huvudmenyn. I följande exempel definieras huvudmenyn **File**.
 
![qt_designer_1](images/qt_designer3.png)

![qt_designer_1](images/qt_designer4.png)

En undermeny till en huvudmeny skapas kommer Qt Designer att automatiskt skapa en s.k. Action. Dessa visas i **Action Editor** längst ned till höger i fönstret.

![qt_designer_1](images/qt_designer5.png)

Namnen på en **Action** kan ändras genom att man väljer en action in editorn och ändrar på namnet i **Property Editor** enligt följande figur:

![qt_designer_1](images/qt_designer6.png)

Namnen som ges här motsvarar de namn som kommer att användas i Python. Försök skapa namn utan mellanrum. 

Färdigställ hela menyn enligt den tidigare listan med menyfunktioner.

### Kontroller för inmatning av parametrar

För att kunna redigera vår modell måste vi skapa kontroller för detta i Qt Designer. De kontroller kan använda är **QLineEdit** och **QLabel**. QLineEdit används för att kunna ange värden i textutor på skärmen. QLabel använder vi för att beskriva vad textutorna beskriver för parameter.

Kontroller skapas genom att man drar kontrollen från **Widget Box** och släpper dem på formulärfönstret. Följande bild visar ett antal kontroller skapade på detta sätt med tillhörande objektnamn.

![qt_designer_1](images/qt_designer7.png)

![qt_designer_1](images/qt_designer8.png)

Texten för **QLabel** kontrollerna ändras genom att välja kontrollen och ändra egenskapen **text** i **Property Editor** enligt följande figur:

![qt_designer_1](images/qt_designer9.png)

!!! note "Att tänka på"

    Ett bra sätt att namnge kontroller in Qt Designer är att ge ett suffix som beskriver vad det är för typ av kontroll. T ex en **QLineEdit**-kontroll för variabeln **b** kan ges namnet **b_edit**. En motsvarande **QLabel**-kontroll kan ges namnet **b_label**.

### Knappar för visualisering

För att kunna visa visualiseringarna behöver vi också ett antal knappar för detta. Skapa följande knappar (**QPushButton**) till höger om de tidigare kontrollerna (Bry er inte om exakt placering. Räcker med ungefärlig placering.):

 * text: **Geometry** - namn: **show_geometry_button**  
 * text: **Mesh** - namn: **show_mesh_button**  
 * text: **Nodal values** - namn: **show_nodal_values_button**  
 * text: **Element values** - namn: **show_element_values_button**
 
Följande figur visar ungefärligt utseende:
 
![qt_designer_1](images/qt_designer10.png)

### Textruta för rapport

För att visa rapporten kommer vi att använda en **QPlainTextEdit**-kontroll. Denna kontroll kan hantera text bestående av flera rader. Skapa en sådan kontroll med namnet **report_edit**. 

Det färdiga fönstret bör nu se ut som i följande figur:

![qt_designer_1](images/qt_designer11.png)

### Ordning och reda på kontroller

Fram till nu har vi placera ut kontrollerna ungefärligt. För att se till att kontrollerna placeras på ett mer ordentligt och skalbart sätt skall vi använda verktyget för grid-layout och radlayout.

Först skapar vi en grid-layout av etiketter, textutor och knappar. Markera dessa kontroller i Qt Designer:

![qt_designer_1](images/qt_designer12.png)

Klicka sedan på grid-verktyget:

![qt_designer_1](images/qt_designer13.png)

Nu skapar Qt Designer automatiskt en grid-layout av de valda kontrollerna:

![qt_designer_1](images/qt_designer14.png)

Det är lite trångt mellan textutorna och knapparna. Lägg in en "Horizontal Spacer" enligt figuren nedan:

![qt_designer_1](images/qt_designer15.png)

För att placera ut kontrollerna så att de fyller ut hela fönstret, markerar huvudfönstret och trycker på **Layout vertically**.

![qt_designer_1](images/qt_designer16.png)

Huvudfönstret bör nu se ut som i följande bild:

![qt_designer_1](images/qt_designer17.png)

Objektstrukturen bör ha följande struktur och namngivning (Måste dock anpassas efter problemområde och egna idéer).

![qt_designer_1](images/qt_designer18.png)

Vi har nu en komplett beskrivning av det grafiska gränssnittet. Spara filen som "mainwindow.ui". 

## Huvudprogram och klass för huvudfönster

För att programmet skall visa vårt gränssnitt måste huvudprogrammet modifieras. Enklast är det om ni skapar en ny Python-fil och börjar från början.

### Moduler som behöver importeras

För att implementera det grafiska gränssnittet måste vi importera ett antal Python-moduler. 

 * qtpy modulerna **QtGui** och **QtCore**. Dessa används för att skapa gränsnittet.
 * CALFEM modulen **calfem.ui**. Denna innehåller en del specialkod för att integrera våra visualiseringrutiner och PyQt.
 * Er egen modul för ert problemområde. I detta exempel använder vi modulen **flowmodel**.
 
Programmets import instruktioner blir då:

``` py 
# -*- coding: utf-8 -*-

import os, sys

from qtpy.QtCore import QThread
from qtpy.QtWidgets import QApplication, QDialog, QWidget, QMainWindow, QFileDialog
from qtpy import uic

import flowmodel as fm
``` 

!!! note "Varför använder vi qtpy istället för PyQt?"

    Det finns idag ett antal olika Qt implementeringar för Python. T ex **PyQt**, **PySide** och **Qt for Python**. För att göra programmet mindre beroende på en viss implementering använder vi modulen **qtpy** som automatiskt väljer den implementering som är tillgänglig. 

### Huvudfönsterklass MainWindow

Vårt huvudfönster implemeterar vi enklast i en egen klass **MainWindow**. Klassens uppgift är att ladda gränssnittsbeskrivningen och implementera de händelsemetoder som krävs för att programmet skall fungera. Det mest grundläggande är en **__init__**-metod som initierar klassen och läser beskrivningen samt visar fönstret på skärmen. Stommen för detta visas i följande exempel:

``` py
class MainWindow(QMainWindow):
    """MainWindow-klass som hanterar vårt huvudfönster"""

    def __init__(self):
        """Constructor"""
        super(QMainWindow, self).__init__()
                  
        # --- Läs in gränssnitt från fil
        
        uic.loadUi('mainwindow.ui', self)
        
        # --- Se till att visa fönstret
        
        self.show()
        self.raise_()
```
            
Funktionen **uic.loadUi()** läser in och skapar objekten som är beskrivna i ui-filen direkt mot **MainWindow**-instansen. T ex ett objekt som har namnet **a_text** i ui-filen kan nås direkt genom klass-variabeln, **self.a_text**.
            
### Ett nytt huvuduprogram

Program som använder fönster har ofta ett annorlunda huvudprogram än t ex beräkningsprogram. Ett fönsterbaserade program använder ofta en s.k. händelse-loop som väntar på händelser från operativsystemet. Händelserna skickar loopen vidare till de underliggande klasserna som sedan hantera dessa. 

Vårt nya huvudprogram visas i nedanstående kod:

``` py
if __name__ == '__main__':

    # --- Skapa applikationsinstans

    app = QApplication(sys.argv)

    # --- Skapa och visa huvudfönster

    window = MainWindow()
    window.show()

    # --- Starta händelseloopen

    sys.exit(app.exec_())
```

Metoden **app.exec_()** returnerar när alla programmets fönster har stängts. 

Programmet vi skapat har nu all kod som krävs för att visa vårt grafiska gränssnitt på skärmen. Under Windows kan gränssnittet se ut som i följande bild när programkoden körs:

![qt_designer_1](images/qt_designer19.png)

## Koppling av händelser till metoder

För att programmet skall kunna köra beräkningar, öppna och spara filer måste vi koppla händelser från kontrollerna till metoder i **MainWindow**-klassen. Att koppla händelser för kontroller till metoder görs med metoden **.connect(...)** som finns definierade för alla händelser en kontroll kan hantera. 

### Koppling av menyhändelser

De första händelserna vi kopplar ihop är menyhändelserna. Menyhändelserna i ui-filen angavs med namn som **new_action** och **open_action**. För att skapa en koppling lägger vi först till en metod som skall hantera själva händelsen:

``` py
class MainWindow:
    ...
    def on_new_action(self):
        """Skapa en ny modell"""
        print("on_new_action")
```
            
Vi lämnar implementeringen av denna till användaren. Kopplingen av metoden gör vi nu i **__init__(...)**

``` py
class MainWindow:
    ...
    def __init__(self):

        ...
        
        # --- Läs in gränssnitt från fil
        
        uic.loadUi("mainwindow.ui", self)
        
        # --- Koppla kontroller till händelsemetoder
        
        self.new_action.triggered.connect(self.on_new_action)
```
            
För menyhändelser är det händelsen **triggered** som skall kopplas.            

### Koppling av händelser för knappar

För att koppla knappar är det händelsen **clicked** som skall kopplas. Följande kod visar ett exempel på detta:

``` py
class MainWindow:
    ...
    def __init__(self, app):
    
        ...
        
        # --- Koppla kontroller till händelsemetoder
        
        self.new_action.triggered.connect(self.on_new_action)
        ...
        self.show_geometry_button.clicked.connect(self.on_show_geometry) # <---
        
    ...
    
    def on_show_geometry(self):
        """Visa geometrifönster"""
        
        print("on_show_geometry")
```

## Integrering av beräkningsmodul

I det förra arbetsbladet skapade vi våra **ModelParams**-, **ModelResults**- och **ModelSolver**-objekt i vårt huvudprogram. I det modifierade programmet är det **MainWindow** som häger alla referenser till dessa objekt. För att hantera modellen och uppdatera kontrollerna implementeras lämpligen följande metoder:

 * **init_model(...)** - Skapar de nödvändiga objekten som behövs för indata, utdata och lösning av problemet. Sätter också standardvärden på de ingående parametrarna i modellen.
 * **update_controls(...)** - Tar värden från ett **ModelParams**-objekt och tilldelar kontrollerna dessa värden.
 * **update_model(...)** - Läser av angivna värden i kontrollerna och tilldelar dessa till **ModelParams**-objektet.
 
För att tilldela värden till kontroller används metoden **setText(...)** på textkontrollerna. Ett exempel på hur detta görs visas i följande kod:

``` py
def update_controls(self):
    """Fyll kontrollerna med värden från modellen"""
    
    self.w_edit.setText(str(self.model_params.w))
    ...
```

> Tänk på att **self.inputData** lagrar värden av typen **float** och alltså måste konverteras till teckensträngar innan **setText(...)** anropas. Detta görs i ovanstående exempel med metoden **str(...)**

För att hämta värden från kontrollerna används metoden **text()** på textkontrollen. Ett exempel på hur detta kan implementeras visas i följande kod:

``` py
def update_model(self):
    """Hämta värden från kontroller och uppdatera modellen"""
    
    self.model_params.w = float(self.w_edit.text())
    ...
```
        
> Vi har den omvända problematiken här, dvs vi måste konvertera från teckensträng från kontrollen till ett **float**-värde genom att använda funktionen **float(...)**.    

## Öppna och spara modeller från filer

Beräkningsmodellen som implementerades i arbetsblad 2 och 3 innehåller metoderna **load(...)** och **save(...)**. Dessa skall nu användas för att implementera metoder för att öppna och spara våra modeller till disk.

### Öppna fil från disk

För att öppna en redan existerande fil från disk, måste vi först fråga användaren om vilken fil som skall öppnas. Detta kan göras med funktionen **QFileDialog.getOpenFileName(...)**. Funktionen visar en stanadard fildialogruta där användaren kan välja en existerande fil. Hur den används visas i följande exempel:

``` py
    def on_open_action(self):
        """Öppna in indata fil"""
        
        filename, _ = QFileDialog.getOpenFileName(self, 
            "Öppna modell", "", "Modell filer (*.json *.jpg *.bmp)")
        
        if filename!="":
            self.filename = filename

            # --- Öppna ModelParams instans

            ...
```

Om användaren avbrutit valet av filnamn returneras en tom sträng. Det är alltid bra att alltid använda en if-sats för att kontrollera att en fil verkligen valts.

Rutinen **load(...)** kan sedan användas för att läsa in modellen från disk med det angivna filnamnet.

### Spara fil till disk

Om användaren vill spara en modell till disk, måste vi på samma sätt först fråga användaren om en plats och ett filnamn. För detta ändamål använder vi istället funktionen **QtGui.QFileDialog.getSaveFileName(...)**. Denna funktion visar en standard fildialogruta som frågar om ett filnamn och en katalog där filen skall sparas. Följande kod visar hur detta sker i metoden **actionSave**:

``` py
def on_save_action(self):
    """Spara modell"""
    
    self.update_model()
    
    if self.filename == "":
        filename, _  = QFileDialog.getSaveFileName(self, 
            "Spara modell", "", "Modell filer (*.json)")

        if filename!="":
            self.filename = filename

    # --- Spara ModelParams instans

    ...    
```
            
## Exekvera beräkningsmodellen

Den enklaste modellen för att exekvera beräkningsmodellen är att helt enkelt anropa **solver.execute()** i en händelsemetod. Detta har dock ett stort problem. Om beräkningsmodellen tar lång tid att exekvera kommer programmet att stå kvar i händelsemetoden och händelse-loopen kommer ej att kunna fortästt förren metoden avslutas. För en användare ser det ut som programmet låst sig, vilket inte är långt ifrån sanningen.

För att lösa denne problematik placerar vi beräkningskoden i en s.k. tråd. En tråd är en parallell exekvering av en given kod. Denna exekvering ligger utanför händelse-loopen, så den inte kommer att blockera denna. 

Problemet med att använda trådar är att vi måste synkronisera exekveringen av dessa, så att vi vet när beräkningen är klar. Detta görs dock enkelt i PyQt:s trådimplementering. 

För att implementera vår beräkning i en tråd måste vi först skapa en speciell trådklass för vår beräkning. Lägg till följande kod längst upp i modulen:

``` py
# -*- coding: utf-8 -*-

import os, sys

from qtpy.QtCore import QThread
from qtpy.QtWidgets import QApplication, QDialog, QWidget, QMainWindow, QFileDialog
from qtpy import uic

import flowmodel as fm

class SolverThread(QThread):
    """Klass för att hantera beräkning i bakgrunden"""
    
    def __init__(self, solver, param_study = False):
        """Klasskonstruktor"""
        QThread.__init__(self)
        self.solver = solver
        self.param_study = param_study
        
    def __del__(self):
        self.wait()
        
    def run(self):
        self.solver.execute()

class MainWindow:
    ...
```

Under metoden **run(...)** skall läggs själva anropet till att starta beräkningen in.  

För att starta beräkningen när man väljer **Calc/Execute** i menyn kan händelsemetoden se ut på följande sätt:

``` py
class MainWindow:
    ...
    def on_execute_action(self):
        """Kör beräkningen"""
        
        # --- Avaktivera gränssnitt under beräkningen.        
        
        self.setEnabled(False)
        
        # --- Uppdatera värden från kontroller
        
        self.update_model()
        
        # --- Skapa en lösare
        
        self.solver = fm.ModelSolver(self.model_params, self.model_results)
        
        # --- Starta en tråd för att köra beräkningen, så att 
        #     gränssnittet inte fryser.
        
        self.solver_thread = SolverThread(self.solver)        
        self.solver_thread.start()
```
      
Denna metod kommer då att starta lösaren som en separat tråd som inte påverkar händelseloopen. 

För att veta när beräkningstråden avslutas måste koppla en metod till händelse **finished** på vår trådklass. Vi skapar först metoden **on_solver_finished(...)**:

``` py
class MainWindow:
    ...
    def on_solver_finished(self):
        """Anropas när beräkningstråden avslutas"""
        
        # --- Aktivera gränssnitt igen        
        
        self.setEnabled(True)
        
        # --- Generera resulatrapport.        

        ...
```
            
Metoden kopplas sedan till trådobjektet med **connect(...)** ungefär på samma sätt som för kontrollerna:

``` py
class MainWindow:
    ...
    def on_execute_action(self):
    
        ...
                    
        self.solver_thread = SolverThread(self.solver)
        self.solver_thread.finished.connect(self.on_solver_finished)   
        self.solver_thread.start()
```
                        
När beräkningen avslutas kommer trådobjektet att automatiskt anropa metoden **self.on_solver_finished**.

## Uppdatering av **ModelVisualisation**-klassen

I det tidigare arbetsbladet 3 implementerade visualiseringen i klassen **ModelVisualisation**. Vi kommer nu att utöka klassen med metoder för att selektivt anropa de olika visualiseringsvarianterna och kopplas dessa till händelsemetoder i vår **MainWindow**-klass.

För att få lite bättre kontroll över de visualiseringsfönster som skall visas implemeteras en metod för varje visualiseringstyp. T ex:

 * **show_geometry()** - Visar geometridefinitionen för problemet.
 * **show_mesh()** - Visar beräkningsnätet som genererats med GMSH.
 * **show_nodal_values()** - Visar beräknade nodvärden.
 * **show_element_values()** - Visar beräknade elementvärden.
 
För att visualiseringsklassen skall kunna hålla reda på vilka fönster som är öppna skapas 4 klassvariabler som skall lagra referenser till de visade figurerna.

``` py
class ModelVisualisation(object):
    """Klass för visualisering av resulat"""

    def __init__(self, model_params, model_results):
        """Konstruktor"""
        
        self.model_params = model_params
        self.model_results = model_results
        
        # --- Variabler som lagrar referenser till öppnade figurer
        
        self.geom_fig = None
        self.mesh_fig = None
        self.el_value_fig = None
        self.node_value_fig = None
```
            
Vi sätter variablerna till **None** så att vi kan särskilja dem från tilldelade variabler.

Ett exempel på hur detta kan användas visas i följande metod:

``` py
class ModelVisualisation(object):
    ...
    def show_geometry(self):
        """Visa geometri visualisering"""
        
        geometry = self.model_results.geometry
        
        self.geom_fig = cfv.figure(self.geom_fig)
        cfv.clf()            
        cfv.draw_geometry(geometry, title="Geometry")
```

Funktionen **cfv.figure(...)** tar en existerande figurreferens som indata. Beroende på om den är tilldelad eller inte skapas returneras den existerande eller så skapas automatiskt en ny figur. **clf()** rensar innehållet i figurfönstret. 

**Visualisation**-klassen skall också imeplementera en metod **closeAll(...)** som stänger alla öppna fönster och nollställer figurvariablerna.            

## Inlämning och redovisning

Det som skall göras i detta arbetsblad är:

 * Implementera _alla_ händelsemetoder för de kontroller som används i gränssnittet.
 * Implementera grundläggande funktioner som ny modell, spara modell, spara som och avsluta programmet.
 * Implementera kontroller för att se till att visualiseringsmetoderna inte anropas om en beräknings inte är utförd. Använd en flagga **self.calcDone** för att ange beräkningsstatus.
 * Slutför implementeringen av **ModelVisualisation**-klassen, så den kan hantera alla visualiseringsfall. Skapa objektet av denna klass på nytt efter varje slutförd beräkning. Skapa en tom variabel in **MainWindow** konstruktorn, så att if-satser kan användas för att testa om det finns en aktuell instans till **ModelVisualisation**
 * Efter beräkningen skall **repoert_edit**-kontrollen tilldelas utdata från **ModelReport**-klassen. **report_edit** har en metod **setPlainText(...)** just för detta ändamål. Aktuellt innehåll i kontrollen kan rensas med metoden **clear()**.
 * Programmet skall vid detta arbetsblad slut vara ett komplett självständigt beräkningsprogram.

Inlämningen skall bestå av en zip-fil (eller annat arkivformat) bestående av: 

 * Alla Python-filer. (.py-filer)
 * Ett exempel på en sparad json-fil.
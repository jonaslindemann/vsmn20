# Programutveckling för Tekniska Tillämpningar - Arbetsblad 1

Detta arbetsblad består av ett antal uppgifter som skall lösas med hjälp av Python och Numpy. För att lösa uppgifter kan följande länkar vara av nytta:

 * How to Think Like a Computer Scientist
 * http://interactivepython.org/courselib/static/thinkcspy/toc.html
 * http://docs.python.org/tutorial/  
 * http://www.scipy.org/Tentative_NumPy_Tutorial

Gå gärna igenom avsnitten fram till “Recursion” i online boken ”How To Think Like a Computer Scientist” innan du börjar med uppgifterna.

Alla uppgifterna sparas som separata python-filer (filer med ändelsen .py).

## Uppgift 1: Skriv ett ”hello, world” program

Alla programmeringsböcker startar med ett enkelt program som skriver ut texten ”Hello, world!”. Skriv ett sådant program i Python.

## Uppgift 2: Enhetskonvertering

Skriv ett program som konverterar en given längd i meter till tum, fot, yards och miles. En tum = 2.54 cm, en fot = 12 tum, en yard = 3 fot och en brittisk mil = 1760 yards. För verifikation, 640 meter är ekvivalent med 25196.85 tum, 2099.74 fot, 699.91 yards eller 0.3977 miles.

## Uppgift 3: Ränteberäkning

Låt p vara bankens årliga ränta. Ett insatt belopp, A, har då vuxit till: 

<img src="images/interest.png" width="150">

Efter n år. Skriv ett program som beräknar vad 1000 kr har vuxit till efter tre år med 5 % ränta.

## Uppgift 4: Fahrenheit till Celsius konverterings tabell

Skriv ett program som skriver ut en tabell med Fahrenheit grader 0, 10, 20, …, 100 i den första kolumnen och motsvarande grader Celsius i den andra kolumnen. För att beräkna grader Celsius används följande formel.

<img src="images/fahrenheit.png" width="150">

## Uppgift 5: Beräkna summan av tal i en fil

Läs in tal från en fil och beräkna den totala summan av dessa. Filen innehåller en siffra per rad.

## Uppgift 6: Plotta en funktion

Plotta funktionen

<img src="images/sinx.png" width="150">

Inom intervallet [0,2π]

För denna uppgift måste import deklarationer göras:

    import numpy as np
    import pylab as pl

## Uppgift 7: Beräkna matriser

Definiera följande matriser in numpy:

<img src="images/matrix.png" width="150"><br>

<img src="images/matrix_b.png" width="90">

Beräkna och skriv ut följande uttryck:

<img src="images/matrix_formulas.png" width="60">

Lös därefter ekvationsystemet 

<img src="images/eqsys.png" width="80">

En linjär ekvationslösare finns som np.linalg.solve(A, b). Följande modul importer måste 

göras för denna uppgift:

    import numpy as np

[img1]: "images/interest.png"
[img2]: "images/fahrenheit.png"
[img3]: "images/sinx.png"
[img4]: "images/matrix.png"
[img5]: "images/matrix_b.png"
[img6]: "images/matrix_formulas.png"
[img7]: "images/eqsys.png"

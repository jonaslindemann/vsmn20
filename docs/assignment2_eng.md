# Program for Technical Applications - Worksheet 2

> ** Remember: ** When ** ... ** appears in the program examples, this indicates that there is no code that you yourselves must add. Variables and data structures are only examples. Depending on the type of problem you may need other data structures than those described in the code examples.

## Structuring of code

Software development is a time consuming job. It is therefore important that the program code out-shaped such that modification and further development facilitated. This means that software program must be structured and have good readability. A calculation process can generally be divided into different logical parts. When programming, the program is assigned appropriately divided into subroutines. Transfer of data is done via the parameters in the declaration and call. The choice of parameters to be thought out, so that redundant parameters are not taken. Transfer of parameters to global variables should be avoided, as these often causes the parameters not used in the current program comes with, which impairs transparency.

To give the program readability, the choice of names for variables and subroutines done with care, so that the name says something about the function. To further improve readability used comment kits to each subroutine describe the purpose, meaning and parameters of the various computational steps. If a program is designed in accordance with the above, it is easy to modify it or to use parts of it for other purposes than the original.

## Strucuture of a finite element application

A finite element analysis can usually be divided into three different steps:

 1. Input of input
 1. Calculation
 1. Presentation of results

Each such step is often performed by a separate program. The data is then transferred between the programs by using the files. Sometimes they choose to, instead of dividing the analysis into three separate applications, allowing two or all three of the steps performed by the same program.

At the beginning of the course we have chosen to focus on step 2 above. Easier input are defined directly in Python to simplify troubleshooting in the implementation of the calculation part.

In this submission, the backbone of the calculation program is created. A problem of the types:

 * Thermal Conductivity
 * Groundwater Flow
 * Stress calculation (Plane stress)
 
is selected. The selected problem will then be used for all future work sheets. You do not need to implement a program that handles all problem types.

We begin by defining a python-mode, for example flowmodel.py, which will include estimating portion of the program. Give the file a name that fits the problem area you have chosen. At the beginning of the file, the following declarations for the modules to be used:

    # -*- coding: utf-8 -*-
    
    import numpy as np
    import calfem.core as cfc
    
For the implementation, we will use object-oriented programming, ie, divide the program into logical objects that can be reused and allow for expansion and reuse of code. 4 logical classes can be identified for the calculation model:

  1. Input Data - Stores the input variables required for calculation.
  1. Solver - Implements routine solution for the problem.
  1. Output Data - Storing the results generated thus solving the problem.
  1. Report - Manages the generation of input and output reports for the program.

## The class InputData

The class input data must contain all inputs required to perform the calculation. A first class definition may look like the following code:

    # -*- coding: utf-8 -*-

    import numpy as np
    import calfem.core as cfc
    import json

    class InputData(object):
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
 
> All ** ... ** indicates that the code must be added.

## The class OutputData

Output data class to be used to store the results generated during the calculation. Because Python is a dynamic language can calculate the class itself add to the result variables in the result object, but it's always good to create empty variables in the class, so that it can function independently. An example of results-class is shown in the following code:

    class OutputData(object):
        """Klass för att lagra resultaten från beräkningen."""
        def __init__(self):
            self.a = None
            self.r = None
            self.ed = None
            self.qs = None
            self.qt = None
            
** None ** can be used as a generic data type that can be used to create variables without content. It is also possible to test if a variable is assigned by an if statement:

    if self.a == None:
        self.a = np.array(...) # Tilldela en riktigt datatyp om self.a == None 
    
## Klassen Solver

The class solver is responsible for performing the actual calculation. The class will have a kontruktor, __init__(...) and a method execute() to perform the actual calculation. Constructor must have two input parameters, **input_data** and **output_data**, which are instances of the classes **InputData** and **OutputData**. The constructor will look like the following:

    class Solver(object):
        """Klass för att hantera lösningen av vår beräkningsmodell."""
        def __init__(self, input_data, output_data):
            self.input_data = input_data
            self.output_data = output_data

The input parameters are assigned two class variables, self.inputData ** ** and ** ** self.outputData.

The calculation to be performed in the execute (...). The method retrieves input from self.inputData ** ** to set up and perform finite element calculations just like a regular Calfem programs. To simplify management of input variables can be local references to the input data is created according to the following code:    class Solver(object):

        ...
            
        def execute(self):
            
            # --- Överför modell variabler till lokala referenser
            
            edof = self.input_data.edof
            cond = self.input_data.cond
            coord = self.input_data.coord
            dof = self.input_data.dof
            ep = self.input_data.ep
            loads = self.input_data.loads
            bcs = self.input_data.bcs       

Because Python handles all the variables that references it is no disadvantage to make these assignments. No copies of the data will be made. The variables need not be copied back to **self.input_data** then the local variables pointing to the same memory contents.

After the calculation is completed, the **self.output_data** is assigned the result of the calculation. The global stiffness matrix, **K** or the load vector **f** need not be stored here. Hot variables is the displacement vector, the reaction force vector and the element forces. The following code shows how this might look like:

    class Solver(object):

        ...
            
        def execute(self):
        
            # --- Överför modell variabler till lokala referenser
            
            ...

            ... Beräkningskod ...
            
            # --- Överför modell variabler till lokala referenser

            self.output_data.a = a
            self.output_data.r = r
            self.output_data.ed = ed
            self.output_data.qs = qs
            self.output_data.qt = qt
            
## The class Report

When beräkingen is completed, a report of input data and results generated, this is done by the class ** Report **. The class will have the same input data ** ** solver.

For the generation of the report, we will use Python's built-in method ** __ str __ () **. This method is used to implement what should happen when using print ** (...) ** on an instance of the class or the function str ** (...) ** is used to convert the class content to a string .

For this to work, we need two additional methods and a string variable. The string variable we will fill with the text description of the class. The extra methods used to clean and fill the string of content. The following code shows the implementation of ** Report ** class:

    class Report(object):
        """Klass för presentation av indata och utdata i rapportform."""
        def __init__(self, input_data, output_data):
            self.input_data = input_data
            self.output_data = output_data
            self.report = ""
            
        def clear(self):
            self.report = ""
            
        def addText(self, text=""):
            self.report+=str(text)+"\n"
                    
        def __str__(self):
            self.clear()
            self.addText()
            self.addText("-------------- Model input ----------------------------------")
            ...
            self.addText("Coordinates:")
            self.addText()
            self.addText(self.input_data.coord)
            ...
            return self.report
                           
## Main program

For the program to work, we need a main program. The following code shows how often do define a main program in Python:

    # -*- coding: utf-8 -*-

    if __name__ == "__main__":
        print("Denna fil exekveras direkt och importeras inte.

If a Python file is imported with the import ** ** - the kit will variable ** __ name __ ** include the module name, ie the name of the source file. However, if you call the file directly with a Python interpreter is ** __ name __ ** contain the value ** __ main __ **. In this way you can ensure that only certain code is executed if the file is started directly with the Python interpreter and at the same time use the file as a module. This concept is often used in Python to create test functions for modules.

A main program of our finite element program that uses all of our classes can then look like this:

    # -*- coding: utf-8 -*-

    import flowmodel as fm

    if __name__ == "__main__":
        
        input_data = fm.InputData()
        output_data = fm.OutputData()

        solver = fm.Solver(input_data, output_data)
        solver.execute()

        report = fm.Report(input_data, output_data)
        print(report)
        
In the above main program, we import the module ** ** Flow model (flowmodel.py) that define our classes. We import the namespace ** sc **. We instantiates ** Input Data ** and ** ** Output Data Objects to manage input and output. A ** ** solver instance, the solver ** ** instantiated with the items, input data ** ** and ** ** output data as input. The calculation is then started by calling the methods we solver.execute ** () **. The program ends with that we create an instance of ** Report ** which we then print on the screen with a ** print () ** - kit.

Example of a running program is shown below:

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
     
## Reading and writing to a file using JSON

An important part of a calculation program is being able to read and write input files. In most cases, the problems are so large that they can not be defined in the program code. In this program, we will use JSON (JavaScript Object Notation) as the format of the files we will write. The following code is an example of how a JSON file might look like.

    {"employees":[
        {"firstName":"John", "lastName":"Doe"},
        {"firstName":"Anna", "lastName":"Smith"},
        {"firstName":"Peter", "lastName":"Jones"}
    ]}
    
The file is very similar to the syntax that Python uses dictionaries. What makes the format attractive is that we ourselves do not need to write it to the file without Python has a built-in library to read and write files of this type.

To implement the functionality to read and write in our program, we conducted the following imports ** ** - set at the beginning of our module file:

    # -*- coding: utf-8 -*-

    import numpy as np
    import calfem.core as cfc
    import json # <--- Denna rad
    
We begin to implement writing to a file, because we have all the input data for this purpose. We add a method, save ** (...) ** of ** Input Data ** - class:

    class InputData(object):
        """Klass för att definiera indata för vår modell."""
        
        ...

        def save(self, filename):
            """Spara indata till fil."""
            
            ...

To define the structure of that which is to be stored, but also to make it easy to read data, we will start from a dictionary as a base for what should be written and read to File. The method ** save (...) ** we add the following code to define the backbone of our data structure:

            inputData = {}
            inputData["version"] = self.version
            inputData["t"] = self.t
            inputData["ep"] = self.ep
                        
** Input data ['version'] ** can be good to have to keep track of which version of the file format to use once you read the file back.

One problem we have to manage is the JSON module in Python can not handle numpy arrays. This is solved, however, simply because we convert our numpy arrays to lists. In the following code we convert the array self.coord ** ** to a list of method **. Tolist (...) **.

            inputData["coord"] = self.coord.tolist()
            
When the input data ** ** is defined, we can open a file for writing and then printing may JSON file with the function json.dumps ** (...) **

            ofile = open(filename, "w")
            json.dump(inputData, ofile, sort_keys = True, indent = 4)
            ofile.close()

** Sort_keys ** and ** ** indent ensures that the file is written nicely formatted.

To load an existing JSON file we do in reverse. We read the entire file into a string of characters which we then convert back to a dictionary function json.load ** (...) **

    def load(self, filename):
        """Läs indata från fil."""
        
        ifile = open(filename, "r")
        inputData = json.load(ifile)
        ifile.close()

        self.version = inputData["version"]
        self.t = inputData["t"]
        self.ep = inputData["ep"]

For our coord array must now convert this back to a numpy array. This is done with numpy function ** np.asarray (...) **

        self.coord = np.asarray(inputData["coord"])
        
The complete writing and reading functions becomes:

    class InputData(object):
        """Klass för att definiera indata för vår modell."""

        def save(self, filename):
            """Spara indata till fil."""

            inputData = {}
            inputData["version"] = self.version
            inputData["t"] = self.t
            inputData["ep"] = self.ep
            ...
            inputData["coord"] = self.coord.tolist()
            ...

            ofile = open(filename, "w")
            json.dump(inputData, ofile, sort_keys = True, indent = 4)
            ofile.close()

        def load(self, filename):
            """Läs indata från fil."""
            
            ifile = open(filename, "r")
            inputData = json.load(ifile)
            ifile.close()

            self.version = inputData["version"]
            self.t = inputData["t"]
            self.ep = inputData["ep"]
            ...
            self.coord = np.asarray(inputData["coord"])
            ...

## Submission and accounting

It is to be done in this worksheet are:

  * Completing the implementation of the Data Input ** ** - class with all the required input to solve the chosen problem. Procedures to save and read from JSON files will also be implemented for all input variables.
  * Completing the implementation of the solver ** ** - class with a finite element solver implemented with the methods that are described in Calfem.
  * Completing the implementation of the ** Report ** - class with a complete transcript of the input and output variables with descriptive texts.

The submission shall consist of a zip file (or other archive format) consisting of:

  * All the Python files. (.py Files)
  * An example of a saved file JSON.
  * Printing from applications run.
  * Comparative calculation of Calfem for MATLAB.

## Element types

### Plate element

![eltype1](images/eltype1.png)

 * u = förskjutning
 * x1,y1,x2,y2,x3,y3 = koordinater
 * E = elasticitetsmodul
 * v = Poissons tal
 * CALFEM för Python element : **plante/plants**

### 2-dimensional heat flow

![eltype2](images/eltype2.png)

 * T = temperatur
 * x1,y1,x2,y2,x3,y3 = koordinater
 * lambda,x, lambda,y = värmekonduktivitet
 * CALFEM för Python element : **flw2te/flw2te**

### Groundwater flow

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


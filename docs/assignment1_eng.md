# Software development for Technical Applications - Worksheet 1

This worksheet consists of a number of tasks to be solved using Python and NumPy. To solve tasks may find the following links useful:

 * How to Think Like a Computer Scientist
 * http://interactivepython.org/courselib/static/thinkcspy/toc.html
 * http://docs.python.org/tutorial/  
 * http://www.scipy.org/Tentative_NumPy_Tutorial

Please go through the sections until the "Recursion" in the online book "How To Think Like a Computer Scientist" before you start with the data.

All data are stored as separate python files (files with the extension .py).

## Task 1: Write a "hello, world" program

All programming books start with a simple program that prints the text "Hello, world!". Write such a program in Python.

## Task 2: Unit conversion  

Write a program that converts a given length in meters to inches, feet, yards and miles. One inch = 2:54 cm, one foot = 12 inches, one yard = 3 feet and a British mil = 1760 yards. For verification, 640 meters is equivalent to 25196.85 inch, 2099.74 ft, 699.91 yards or 0.3977 miles.

## Task 3: Interest Calculation

Let p be the bank's annual interest rate. An amount deposited, A, has since grown to:

<img src="../images/interest.png" width="150">

After n years. Write a program that calculates what 1000 SEK has grown to three years with 5% interest.

## Task 4: Fahrenheit to Celsius conversion table

Write a program that prints a table of Fahrenheit degrees 0, 10, 20, ..., 100 in the first column and the corresponding Celsius in the second column. To calculate degrees Celsius, the following formula.

<img src="../images/fahrenheit.png" width="150">

## Task 5: Calculate the sum of the numbers in a file

Read the speech from a file and calculate the total sum of these. The file contains one number per line.

## Task 6: Graphing a Function

Plot function

<img src="../images/sinx.png" width="150">

Within the range [0,2π]

For this task, the import declarations made:

    import numpy as np
    import pylab as pl

## Task 7: Calculate the matrices

Define the following matrices in numpy:

<img src="../images/matrix.png" width="150"><br>

<img src="../images/matrix_b.png" width="90">

Calculate and print the following expression:

<img src="../images/matrix_formulas.png" width="60">

Solve then the system of equations

<img src="../images/eqsys.png" width="80">

A linear equation solver is as np.linalg.solve (A, B). The following module imports must made for this task:

    import numpy as np


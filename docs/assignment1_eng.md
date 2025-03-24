# Worksheet 1

This worksheet consists of a number of tasks to be solved using Python and NumPy. The links provided [here](links.md) may be helpful for completing the tasks.

Please go through the sections up to "Recursion" in the online book ["How To Think Like a Computer Scientist"](https://runestone.academy/ns/books/published/thinkcspy/index.html) before starting the worksheet.

All source files should be stored as separate Python files (files with the extension .py).

## Task 1: Write a "hello, world" program

All programming books start with a simple program that prints the text "Hello, world!". Write such a program in Python and save it in a file named `hello_world.py`.

## Task 2: Unit conversion  

Write a program that converts a given length in meters to inches, feet, yards and miles. One inch = 2.54 cm, one foot = 12 inches, one yard = 3 feet and a British mil = 1760 yards. For verification, 640 meters is equivalent to 25196.85 inch, 2099.74 ft, 699.91 yards or 0.3977 miles.

## Task 3: Interest Calculation

The amount of money in a bank account increases each year by a certain percentage of the amount present at the beginning of the year. This is called interest. The formula for calculating the amount after one year is:

$$A = P \left( 1 + \frac{r}{100}\right)^n$$

Where:

- A is the amount after n years,
- P is the principal amount, 
- r is the annual interest rate,
- n is the number of years (3 years in this case).

Write a program that calculates what 1000 SEK has grown to after three years with 5% interest.

## Task 4: Fahrenheit to Celsius conversion table

Write a program that prints a table of Fahrenheit degrees 0, 10, 20, ..., 100 in the first column and the corresponding Celsius in the second column. To calculate degrees Celsius, use the formula:

$$C = \frac{5}{9} (F - 32)$$

## Task 5: Calculate the sum of the numbers in a file

Read the numbers from a file and calculate the total sum of these. The file contains one number per line. Ensure the program handles errors such as missing files or invalid data gracefully by using appropriate exception handling (e.g., `try-except` blocks).

## Task 6: Graphing a Function

Plot the function

$$f(x) = \sin x^2$$

Within the range [0,2π]

For this task, the import declarations made:

``` py
import numpy as np
import matplotlib.pyplot as plt
```

!!! tip
    A function can be plotted using the `plt.plot()` function. The x-values are generated using `np.linspace(start, stop, num)` and the y-values are calculated by applying the function to the x-values. The plot is displayed using `plt.show()`.
    
    A range of values can be generated using `np.linspace(start, stop, num)`. 


## Task 7: Calculate the matrices

Define the following matrices in numpy:

$$\mathbf{A} = \begin{bmatrix} 
1 & 2 & 3 \\ 
3 & 7 & 4 \\ 
2 & 5 & 3 \end{bmatrix}$$

$$\mathbf{b} = \begin{bmatrix}
1 \\
2 \\
3 \end{bmatrix}$$

Calculate and print the following expression:

$$\mathbf{A}^T$$

$$\mathbf{Ab}$$

$$\mathbf{A}^{-1}$$

Solve then the system of equations

$$\mathbf{Ax} = \mathbf{b}$$

The following module imports must made for this task:

``` py
import numpy as np
```

!!! tip

    A linear equation solver is as np.linalg.solve (A, B). The inverse of a matrix can be calculated using np.linalg.inv(A). The transpose of a matrix can be calculated using np.transpose(A) or A.T.
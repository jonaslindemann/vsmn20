# How to present data in tables

In many applications it is important to be able to create readable tables in text based reports. You could of course implement this yourself in Python, but there is an easier way. The Python-module tabulate can take any list or array based structure and present it in a text readable form. The module is installed by the following command:

```py
pip install tabulate
```

The module is imported in Python using the following import statement:

```py
import tabulate as tbl
```

A simple example of printing an array in a table with headers is shown in the following examples:

```py
A = np.random.rand(10,5)

A_table = tbl.tabulate(
    A,
    headers=["A", "B", "C", "D", "E"],
    numalign="right",
    floatfmt=".4f",
    tablefmt="psql",
)

print(A_table)
```

The resulting output will be:

    +--------+--------+--------+--------+--------+
    |      A |      B |      C |      D |      E |
    |--------+--------+--------+--------+--------|
    | 0.3636 | 0.1814 | 0.9069 | 0.3731 | 0.9131 |
    | 0.1123 | 0.2457 | 0.3626 | 0.2613 | 0.5432 |
    | 0.6862 | 0.1532 | 0.0058 | 0.4078 | 0.1499 |
    | 0.4528 | 0.0012 | 0.6174 | 0.4762 | 0.7446 |
    | 0.9884 | 0.4961 | 0.1079 | 0.8766 | 0.5033 |
    | 0.4039 | 0.7538 | 0.6479 | 0.5291 | 0.6946 |
    | 0.1390 | 0.8729 | 0.5255 | 0.6712 | 0.6944 |
    | 0.7284 | 0.8625 | 0.7105 | 0.4617 | 0.9684 |
    | 0.5844 | 0.5775 | 0.0851 | 0.1950 | 0.6114 |
    | 0.4152 | 0.7463 | 0.3412 | 0.7114 | 0.7328 |
    +--------+--------+--------+--------+--------+

The tabulate function always returns a string with table, which make it suitable for writing to files or to text controls in a user interface. 

It is also possible to add an index to the left side of the table using the showindex keyword:

```py
B = np.arange(50).reshape(10,5)

B_table = tbl.tabulate(
    B,
    headers=["A", "B", "C", "D", "E"],
    numalign="right",
    floatfmt=".4f",
    tablefmt="psql",
    showindex=range(1, len(A) + 1)
)

print(B_table)
```

This results in the following output:

    +----+-----+-----+-----+-----+-----+
    |    |   A |   B |   C |   D |   E |
    |----+-----+-----+-----+-----+-----|
    |  1 |   0 |   1 |   2 |   3 |   4 |
    |  2 |   5 |   6 |   7 |   8 |   9 |
    |  3 |  10 |  11 |  12 |  13 |  14 |
    |  4 |  15 |  16 |  17 |  18 |  19 |
    |  5 |  20 |  21 |  22 |  23 |  24 |
    |  6 |  25 |  26 |  27 |  28 |  29 |
    |  7 |  30 |  31 |  32 |  33 |  34 |
    |  8 |  35 |  36 |  37 |  38 |  39 |
    |  9 |  40 |  41 |  42 |  43 |  44 |
    | 10 |  45 |  46 |  47 |  48 |  49 |
    +----+-----+-----+-----+-----+-----+
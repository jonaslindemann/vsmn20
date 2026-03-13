# Worksheet 6 — Final Submission

This worksheet is the culmination of Worksheets 1–5. You will submit your complete finite element application together with a written report that documents the work.

## Checklist before submitting

!!! warning "Before you submit"
    - All code comments must be in English and correctly describe the code.
    - The program must run without errors on the example input files you provide.
    - Verify your numerical results against the reference values from the earlier worksheets.

## Deliverables

### 1. Written report

The report should be a self-contained document describing your application. It must include:

1. **Problem description** — the physical problem you chose (thermal, groundwater, or plane stress) and a brief overview of the underlying theory.
2. **Program structure** — a description of each class (`ModelParams`, `ModelSolver`, `ModelResult`, `ModelReport`, `ModelVisualization`) and how they interact.
3. **Verification** — a reasonableness check of your results against the reference values provided in Worksheet 2 and Worksheet 3.
4. **User manual** — step-by-step instructions for running the program, including:
    - How to create a new model.
    - How to open and save a model file.
    - How to run a calculation and view results.
    - How to run a parameter study and open the output in ParaView.
5. **Source code appendix** — all `.py` files reproduced in full.

### 2. Code archive

A zip file containing everything required to run the application:

* All Python source files (`.py`)
* At least one example model file (`.json`)
* Example VTK output files from a parameter study (`param_study_01.vtk`, …)
* Any additional resource files (`.ui` files, images, etc.)

## Grading criteria

The submission is assessed on:

| Criterion | Description |
|-----------|-------------|
| Correctness | Program produces numerically correct results |
| Code quality | Clear naming, comments, and class structure |
| Report quality | Clear writing, correct theory, and useful user manual |
| Completeness | All features from WS 1–5 are implemented and working |

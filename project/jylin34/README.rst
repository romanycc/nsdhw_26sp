=====================================================================
Proposal: FastGA - A High-Performance Hybrid Genetic Algorithm Engine
=====================================================================

Basic Information
=================

**Repository:**


Problem to Solve
================

Genetic Algorithms (GA) are a class of heurstic search and optimization
techniques inspired by the process of natural selection. They are used to solve
complex optimization problems by evolving a population of candidate solutions
using operators such as selection, crossover, and mutation. As a stochastic,
population-based approach, GA is particularly effictive for navigating
high-dimensional or non-convex search spaces where traditional gradient-based
methods may fail.

Specifically, FastGA focuses on optimization problems where candidate solutions 
are represented as **1D vectors of double-precision floating-point numbers**. 
While this representation is versatile—covering everything from financial 
portfolio weights to physical control parameters—it introduces significant 
computational challenges when implemented in pure Python:

1. **Overhead of 1D Vector Processing**:
   As the dimension of the 1D vector (chromosome) increases, performing element-
   wise genetic operations in Python becomes extremely slow due to interpreter 
   loop overhead and dynamic type-checking.

2. **The "Flexibility vs. Performance" Trade-off**:
   Users often require **custom, user-defined fitness functions** to evaluate 
   their specific problems. In existing libraries like PyGAD, calling these 
   custom Python functions within nested C-style loops creates a massive 
   bottleneck. There is a lack of tools that provide the flexibility of 
   Python-defined logic with the execution speed of a compiled C++ engine.

3. **Memory and Parallelism Inefficiency**:
   Large-scale 1D vector populations require contiguous memory layouts to 
   benefit from CPU cache prefetching and SIMD instructions. Furthermore, 
   Python's Global Interpreter Lock (GIL) prevents true multi-core utilization 
   during the fitness evaluation of these vectors.

FastGA aims to solve these problems by providing a high-performance C++ core 
that manages contiguous 1D arrays while offering a seamless "Batch Callback" 
interface for user-defined fitness functions in Python.

Prospective Users
=================

FastGA is desgined for users who require both the rapid prototyping capabilities
of Python and the computational performance of C++. The potential user includes:

1. **Scientific Researchers**
2. **Quantitative Financial Analysts**
3. **Machined Learning Engineer**
4. **Students and Educators**

System Architecture
===================

FastGA implements a three-layered hybrid architecture to decouple high-level 
modeling from low-level computational execution, ensuring zero-copy data flow.


1. **Python API Layer (User Interface)**:
   Provides a high-level interface for evolution control. It supports **Batch 
   Evaluation**, where the entire population is passed to the Python-defined 
   fitness function as a single NumPy array, reducing the cross-language 
   callback overhead from $O(N)$ to $O(1)$ per generation.

2. **Interoperability Layer (pybind11)**:
   Acts as a transparent bridge utilizing the **NumPy Buffer Protocol**. It 
   maps C++ memory buffers directly to NumPy ndarrays, enabling **Zero-copy** 
   data sharing. This allows the C++ engine and Python scripts to operate on 
   the same memory address.

3. **C++ Core Engine (Performance Core)**:
   * **Data Representation**: Stores the population in a **strictly contiguous 
     1D array** of doubles (Row-major). This layout maximizes spatial locality 
     for CPU cache prefetching.
   * **Parallelism**: Genetic operators are optimized with **SIMD** instructions. 
     During fitness evaluation, the engine **releases the GIL** and utilizes 
     **OpenMP** for true multi-core parallel execution.

Workflow and Data Flow:
-----------------------

.. code-block:: text

   [ Python ] -> User defines Fitness(NumPy_Matrix)
      |
   [ pybind11 ] -> Zero-copy mapping (NumPy <-> double*)
      |
   [ C++ Core ] -> 1. Crossover/Mutation (SIMD Optimized)
                -> 2. Release GIL & OpenMP Parallel Evaluation
                -> 3. Selection & Memory Management

Key C++ Components:
-------------------

.. code-block:: cpp

   class Population {
   public:
       // Contiguous buffer: size = pop_size * genome_size
       std::vector<double> m_data;
       double* data_ptr() { return m_data.data(); }
   };

   class GAEngine {
   public:
       // Main loop with OpenMP and GIL release
       void run_evolution(Population& pop, int generations);
       // Interface for vectorized Python callbacks
       void set_batch_fitness(py::function func);
   };

API Description
===============

A user interacts with FastGA by defining a **Vectorized Fitness Function**. 
The function receives the entire population as a NumPy array, allowing the 
user to use optimized NumPy expressions.

**Example Script:**

.. code-block:: python

   import fastga
   import numpy as np

   # User defines the logic in Python
   def objective_function(population):
       # population is a 2D NumPy view of the internal C++ buffer
       # Optimized vectorized calculation: f(x) = sum(x^2)
       return -np.sum(population**2, axis=1)

   # 1. Setup Population (100 individuals, each with 50 variables)
   pop = fastga.Population(pop_size=100, genome_size=50)

   # 2. Setup C++ Engine
   engine = fastga.GAEngine(mut_rate=0.01, cross_rate=0.8)
   engine.set_batch_fitness(objective_function)

   # 3. Execute High-performance Evolution
   engine.evolve(pop, generations=1000)

   # 4. Result Retrieval (Zero-copy)
   best_result = pop.get_best()


Engineering Infrastructure
==========================



Schedule
========



References
==========


# Goal

This project provides several algorithms implemented in python to solve linear programs of the form

![latex:\large $\mathbf{x}^*=argmin_\mathbf{x} \mathbf{c}^t\mathbf{x} ~  s.t.~  A_e\mathbf{x}=\mathbf{b_e},A_i\mathbf{x}\leq\mathbf{ b_i}, \mathbf{l}\leq \mathbf{x}\leq \mathbf{u}$ ](https://rawgithub.com/martinResearch/PySparseLP/master/images/LPproblem.svg)

where Ae and Ai are sparse matrices

The different algorithms that are implemented here are documented in the [pdf](./latex/SparseLinearProgramming.pdf): 

* a dual coordinate ascent method with exact line search 
* a dual gradient ascent with exact line search
* a first order primal-dual algorithm adapted from chambolle pock [2]
* three methods based on the Alternating Direction Method of Multipliers [3]

**Note** These methods are not meant to be efficient methods to solve generic linear programs. They are simple and quite naive methods I  implemented while exploring different possibilities to solve very large sparse linear programs that are too big to be solved using the standard simplex method or standard interior point methods.

This project also provides: 

* a python implementation of Mehrotra's Predictor-Corrector Pimal-Dual Interior Point method.
* a python class *SparseLP* (in SparseLP.py) that makes it easier to build linear programs from python 
* methods to convert between the different common forms of linear programs (slack form, standard form etc), 
* methods to import and export the linear program from and to standard file formats (MPS), It is used here to run [netlib](http://www.netlib.org/lp/data/) LP problems. Using mps file one can upload and solve LP on the [neos](https://neos-server.org/neos/) servers.
* a simple constraint propagation method with back-tracking to find feasible integer values solutions (for integer programs)
* interfaces to other solvers (SCS, ECOS, CVXOPT) through CVXPY
* interfaces to other LP and MILP solvers ([CLP](https://www.coin-or.org/download/binary/Clp/), [CBC](https://www.coin-or.org/download/binary/Cbc/), [MIPLC](http://mipcl-cpp.appspot.com/), [GLPSOL](https://sourceforge.net/projects/winglpk/), [QSOPT](http://www.math.uwaterloo.ca/~bico/qsopt/downloads/downloads.htm)) using mps text files

# Build and test status

![Python package](https://github.com/martinResearch/PySparseLP/workflows/Python%20package/badge.svg)

# Installation

## Compiling from source

using pip

	sudo pip install git+git://github.com/martinResearch/PySparseLP.git
	
otherwise you can dowload it, decompress it and compile it locally using 

	python setup.py build_ext --inplace

If you want to be able to run exernal solvers using mps files in windows then download the following executables and copy them in the solvers\windows subfolder

* [clp.exe](https://www.coin-or.org/download/binary/Clp/) 
* [cbc.exe](https://www.coin-or.org/download/binary/Cbc/) 
* [mps_mipcl.exe](http://mipcl-cpp.appspot.com/)
* [glpsol.exe and glpk_4_63.dll](https://sourceforge.net/projects/winglpk/)
* [qssolver.exe](http://www.math.uwaterloo.ca/~bico/qsopt/downloads/downloads.htm)

# LP problem modeling

This library provides a python class *SparseLP* (in SparseLP.py) that aims at making it easier to build linear programs from python. It is easy to derive a specialize class from it and add specialized constraints creations methods (see pott penalization in example 1). 
SparseLP is written in python and relies on scipy sparse matrices and numpy matrices to represent constraint internally and for its interface. There is no variables class binding to c++ objects. This makes it potentially easier to interface with the python scientific stack. 

## Debuging

Constructing a LP problem is often error prone. If we can generate a valid solution before constructing the LP we can check that the constraints are not violated as we add them to the LP using the method *check_solution*. This make it easier to pin down which constraint is causing problem. We could add a debug flag so that this check is automatic done as we add constraints.

## Other modeling tools
Other libraries provide linear program modeling tools, but they have some limitations:

* [CyLP](https://github.com/coin-or/CyLP): use operator oveloading so that we can use notation that are close to mathetmatical notations. But variables are defined as 1D vectors
* [GLOP](https://developers.google.com/optimization/lp/glop): tha variables are defined one by one as scalars which make the creation of large LPs very slow in python.
* [PuLP](https://github.com/coin-or/pulp). Variables are added as scalars, one at a time, instead of using arrays, which make the creation of large LPs very slow in python.
* [Pyomo](http://www.pyomo.org/)

The approach taken here is lower level than this tools (no *variable* class and no operator overloading to define the constraints) but provide more control and flexibility on how to define the constraints and the objective function. It is made easy by using numpy arrays to store variables indices.

# Examples

## Image segmentation
We consider the image segmentation problem with Potts regularisation:

![latex: \large $min_s c^ts + \sum_{(i,j)\in E} |s_i-s_j| ~s.t. ~0 \leq s\leq 1$](https://rawgithub.com/martinResearch/PySparseLP/master/images/segmentation.svg)

with *E* the list of indices of pairs of neighbouring pixels and *c* a cost vector that is obtain from color distribution models of the two regions.
This problem can be rewritten as a linear progam by adding an auxiliary variable *d_ij* for each edge with the constraints

![latex: \large $min_s c^ts + \sum_{(i,j)\in E} d_{ij} ~s.t. ~0 \leq s\leq 1, ~d_{ij}\geq s_j-s_j, ~d_{ij}\geq s_i-s_i $](https://rawgithub.com/martinResearch/PySparseLP/master/images/segmentation_lp.svg)
 
This problem can be more efficiently solved using graph-cuts than with a generic linear program solver but it is still interesting to compare the different generic LP solvers on this problem. 


	from pysparselp.example1 import run
	run()

Here are the resulting segmentations obtain with the various LP solvers, with the same random data term with the optimisations limited to 15 seconds for each solver.
![curves](https://rawgithub.com/martinResearch/PySparseLP/master/images/potts_results.png)
convergence curves
![curves](./images/potts_curves.png)

Note that instead of using a simple Potts model we could try to solve the LP from [5]

## Sparse inverse convariance matrix 
 
The Sparse Inverse Covariance Estimation problem aims to find a sparse matrix B that approximate the inverse of Covariance matrix A.

![latex:\large $B^*=argmin_B \|B\|_1~ s.t.~ \|A B-I_d\|_\infty\leq \lambda$](https://rawgithub.com/martinResearch/PySparseLP/master/images/sparse_inv_covariance.svg)

Let denote *f* the fonction that take a matrix as an input an yield the vector of coefficient of the matrix in row-major order.
Let *b=f(B)* we have *f(AB)=Mb* with *M=kron(A,I_d)*
The problem rewrites

![latex: \large $ min_{b,c} \sum_i c_i ~s.t.~ -b\leq c,~b\leq c,~-\lambda\leq M b-f(I_d)\leq \lambda$](https://rawgithub.com/martinResearch/PySparseLP/master/images/lp_sparse_inv_covariance.svg)

We take inspiration from this scikit-learn example [here](http://scikit-learn.org/stable/auto_examples/covariance/plot_sparse_cov.html) to generate samples of a gaussian with a sparse inverse covariance (precision) matrix. From the sample we compute the empirical covariance A and the we estimate a sparse inverse covariance (precision) matrix B from that empirical covariance using the LP formulation above.

	from pysparselp.example2 import run
	run()

![curves](./images/sparse_precision_matrix.png)

## L1 regularised multi-class SVM

Given *n* examples of vector-class pairs *(x_i,y_i)*, with *x_i* a vector of size m and *y_i* an integer representing the class, we aim at estimating a matrix *W* of size *k* by *m* that allows to discriminate the right class, with *k* the number of classes. We assume that the last component of *x_i* is a one in order to represent the offset constants in *W*. we denote *W_k* the *k*th line of the matrix *W*

![latex:\large $W^*=argmin_W min_{\epsilon}\|W\|_1+\sum_{i=1}^n \epsilon_i\\ s.t.~ W_{y_i}x_i-W_kx_i>1-\epsilon_i \forall\{(i,k)|k\neq y_i\}$](https://rawgithub.com/martinResearch/PySparseLP/master/images/l1svm.svg)

By adding auxiliary variables in a matrix *S* of the same size as the matrix *W* we can rewrite the absolute value as follow:
![latex:\large $\|W\|_1=min_S \sum_{ij}S_{ij} \\ s.t.~ W_{ij}<S_{ij}, -W_{ij}<S_{ij} \forall(ij)$](https://rawgithub.com/martinResearch/PySparseLP/master/images/abstolp.svg)

We obtain the LP formulation:

![latex:\large $W^*=argmin_{W} min_{\epsilon,S} \sum_{ij}S_{ij} +\sum_{i=1}^n \epsilon_i\\s.t.~W_{y_i}x_i-W_kx_i>1-\epsilon_i \forall\{(i,k)|k\neq y_i\},W_{ij}<S_{ij}, -W_{ij}<S_{ij} \forall(ij)$](https://rawgithub.com/martinResearch/PySparseLP/master/images/l1svmLP.svg)


The example can be executed using the following line in python

	from pysparselp.example3 import run
	run()


The support vectors are represented by black circles.

![classification result with support points](https://rawgithub.com/martinResearch/PySparseLP/master/images/l1svmClassification.svg)

## Bipartite matching 

Bipartite matching can be reformulated as an integer linear program:

![latex: $$ max \sum_{ij\in \{1,\dots,n\}^2} M_{ij} C_{i,j} ~ s.t~ M_{ij}\in\{0,1\}, \sum_j M_{ij}\leq 1 \sum_i M_{ij}\leq 1 $$](./images/bipartite.svg)

We relax it into an continuous variables LP.

	from pysparselp.example4 import run
	run()



## K-medians

Given *n* points we want to cluster them into *k* set by minimizing

![latex: $min_ {C \subset \{1,\dots,n\}} \sum_i min_{j\in C}d_{ij}~ s.t~ card(C)\leq k$](./images/kmedians1.svg)
with *d_ij* the distance between point *i* and point *j*
This can be reformulated as an integer program:

![latex: $$ min \sum_{ij\in \{1,\dots,n\}^2} L_{ij} d_{ij} ~ s.t~ L_{ij}\in\{0,1\}, \sum_j L_{ij}=1 \forall i, L_{ij}<u_i \forall (i,j),\sum_i u_i\leq k $$](./images/kmedians2.svg)
 
We relax it into a continuous variabels LP using 

![latex: $$ L_{ij}\in[0,1]$$](./images/kmedians2_relax.svg)
 
	from pysparselp.example5 import run
	run()

![kmedians result](./images/kmedians.svg)


## Netlib LP problems 

We have an interface to easily test the various solvers on netlib problems from [netlib](http://www.netlib.org/lp/data/). 
The uncompressed files are downloaded from [here](ftp://ftp.numerical.rl.ac.uk/pub/cuter/netlib). 
In order to monitor convergence rates, the exact solutions of these problems are found [here](http://www.zib.de/koch/perplex/data/netlib/txt/)

	from pysparselp.test_netlib import test_netlib
	test_netlib('SC50A')

![curves](./images/libnetSC50A.png)
Note: since august 2017, numpy files containing the netlib examples are provided with scipy [here] (https://github.com/scipy/scipy/tree/master/benchmarks/benchmarks/linprog_benchmark_files)

## Random problems 

Random sparse LP problem can be generate using code in *randomLP.py*. The approach used to generate random problem is very simple and could be improved in order to generate harder sparse LPs. We could implement the approach used in section 6.2.1 in https://arxiv.org/pdf/1404.6770v3.pdf to  generate random problems with the matlab code available [here](https://github.com/YimingYAN/pipm-lp/tree/master/Tests/Ultilities)

# To Do

* improve the API by removing redundant functions
* add OSQP[11] as an available solver
* translate from Matlab ot pyton the ADMM methods from [https://github.com/nmchaves/admm-for-lp](https://github.com/nmchaves/admm-for-lp)
* add automatic constraint checking if we provide a feasible solution from the begining. It will help debugging constraints.
* convert to python the matlab implementation of the LP solver based on improved version of champolle-pock called [Adaptive Primal-Dual Hybrid Gradient Methods](https://arxiv.org/abs/1305.0546) available [here](https://www.cs.umd.edu/~tomg/projects/pdhg/)
* create a cython binding for LPsparse [1] using scipy.sparse matrices for the interface and adding the possibility to compute the convergence curve by providing the problem known solution to the solver or by adding the possibility to define a callback to a python function.
* implement method [4]
* implement method in [5]
* add interface to [8] once the code is online.
* try to get more meaningfull convergence curves for scipy.linprog, or maybe those are the expected curves ? 
* we provide an implementation of Mehrotra's Predictor-Corrector Pimal-Dual Interior Point method translated to python from  [Yiming yan's matlab code](https://github.com/YimingYAN/mpc). We could add other interior point methods by translating into python the code 
	* https://github.com/YimingYAN/pathfollow (matlab)
	* https://github.com/YimingYAN/pipm-lp (matlab)
	* http://www.cs.ubc.ca/~pcarbo/convexprog.html
	* https://github.com/YimingYAN/cppipm (c++)
	* https://github.com/pkhuong/cholesky-is-magic (lisp) described here https://www.pvk.ca/Blog/2013/12/19/so-you-want-to-write-an-lp-solver/	
* implement some presolve methods to avoid singular matrices in the interior point methods (for example http://www.davi.ws/doc/gondzio94presolve.pdf). For example detect constraints on singletons, duplicated rows etc.
* add basis pursuite example using [9] .
* add non negative matrix factorization example using [10]

# Alternatives

## Linear Program solvers with a python interface
* Scipy's [linprog](http://docs.scipy.org/doc/scipy/reference/generated/scipy.optimize.linprog.html). It provides three different solvers (Simplex, revised Simplex and an interior point method). it is possible to call this solvers from within our code using *method="scipy_simplex"* or *method=""scipy_interior_point"* when callign the *solve* method. The simplex method is implemented in python with many loops and is very slow for problems that involve more than a hundred variables. 
* OSQP. Operator Splitting Quadratic programming [11]. It support support linear programming (with all zeros hessian matrix)
  [python interface](https://github.com/oxfordcontrol/osqp-python)
* GPU implementation of OSQP (can be 2 order of magnitude faster)[here](https://github.com/oxfordcontrol/cuosqp)
* Python bindings for GLPK [here](https://en.wikibooks.org/wiki/GLPK/Python) . Might not be adapted to very large sparse problems as it use simplex or interior point methods. The installation is a bit tedious. The licence is GPL which makes it unsuited for use in commercial products.
* [GLOP](https://developers.google.com/optimization/lp/glop), Google's linear programming system has a python interface [pywraplp](https://developers.google.com/optimization/introduction/using#python). 
* [CyLP](http://mpy.github.io/CyLPdoc/index.html) . Python interface to Coin-Or solvers CLP, CBC, and CGL. We can use the first two solvers using mps files using my code. Isntalling CyLP involves quite a few steps. CyLP also provide LP modeling tools.
* [CVXOPT](http://cvxopt.org/), provides linear program cone program solvers and also provides interfaces to GLPK,Mosek,DSPD. 
* [CVXPY](http://www.cvxpy.org/en/latest/) Python-embedded modeling language for convex optimization problems. It provide interface to cvxopt solvers and to SCS
* [SCS](https://pypi.python.org/pypi/scs), [github](https://github.com/cvxgrp/scs) Solves convex cone programs via operator splitting. Can solve in particular linear programs. 
## No python interface

* [LIPSOL](http://www.caam.rice.edu/~zhang/lipsol/). matlab code. Seems to be adequate for sparse problems. Part of the code in fortran. licence GPL
* [LPsolve](https://sourceforge.net/projects/lpsolve/) licence LGPL. Python wrapper [here](http://lpsolve.sourceforge.net/5.5/,Python.htm#Install_the_lpsolve_driver). I cannot find  in the windows installer the command line executable mentioned in the documentation that could be executed with mps files.
* [Joptimize](http://www.joptimizer.com/linearProgramming.html) implemented in Java. Appache licence
* [PCx](http://pages.cs.wisc.edu/~swright/PCx/) PCx is an interior-point predictor-corrector linear programming package. Code available here https://github.com/lpoo/PCx. Free but to public domain. Binaries provided for Linux only.
* [DSDP](http://www.mcs.anl.gov/hs/software/DSDP/) solve semidefinite programs, which are more general than linear programs. It uses the sparsity of the problem and might still be competitive to solve sparse linear programs. Can be called from python through [cvxopt](http://cvxopt.org/)

# References

[1] *Sparse Linear Programming via Primal and Dual Augmented Coordinate Descent* Ian En-Hsu Yen, Kai Zhong, Cho-Jui Hsieh, Pradeep K Ravikumar, Inderjit S Dhillon , NIPS 2015. [code](http://ianyen.site/LPsparse/)

[2] *Diagonal preconditioning for first order primal-dual algorithms in convex optimization* T. Pock and A.Chambolle  ICCV 2011

[3] *Distributed Optimization and Statistical Learning via the Alternating Direction Method of Multipliers* Stephen Boyd  Foundations and Trends in Machine Learning 2010

[4] *Augmented
Lagrangian method for large-scale linear programming problems* Yu G Evtushenko, A I Golikov, and N Mollaverdy. Optimization Method and Software 2005.

[5] *Alternating Direction Method of Multipliers for Linear Programming*. He Bingsheng and Yuan Xiaoming. 2015. Paper [here](http://www.optimization-online.org/DB_FILE/2015/06/4951.pdf) 

[6] *Local Linear Convergence of the Alternating Direction Method of Multipliers on Quadratic or Linear Programs*. Daniel Boley. SIAM Journal on Optimization. 2013

[7] *Multiblock ADMM Methods for Linear Programming*. Nico Chaves, Junjie (Jason) Zhu. 2016. report and matlab code [here](https://github.com/nmchaves/admm-for-lp)

[8] *A New Alternating Direction Method for Linear Programming*. Sinong Wang, Ness Shroff. NIPS 2017
paper [here](http://papers.nips.cc/paper/6746-a-new-alternating-direction-method-for-linear-programming.pdf)

[9] *Equivalence of Linear Programming and Basis Pursuit*. paper [here](http://onlinelibrary.wiley.com/doi/10.1002/pamm.201510351/pdf)

[10] *Factoring nonnegative matrices with linear programs*
Victor Bittorf, Benjamin Recht, Christopher Re, Joel A. Tropp. 2012 
paper [here](https://arxiv.org/abs/1206.1270)

[11] *OSQP: An Operator Splitting Solver for Quadratic Programs*. B.Stellato,  G. Banjac,  P. Goulart,   A. Bemporad and S. Boyd. ArXiv e-prints 2017 



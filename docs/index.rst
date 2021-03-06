The JiTCDDE module
==================

Introduction
------------

JiTCDDE (just-in-time compilation for delay differential equations) is a standalone Python implementation of the DDE integration method proposed by Shampine and Thompson [ST01]_, which in turn employs the Bogacki–Shampine Runge–Kutta pair [BS89]_.
JiTCDDE is designed in analogy to `JiTCODE`_ (which is handled very similarly to `SciPy’s ODE`_ (`scipy.integrate.ode`)):
It takes an iterable (or generator function) of `SymPy <http://www.sympy.org/>`_ expressions, translates them to C code, compiles them and an integrator wrapped around them on the fly, and allows you to operate this integrator from Python.

This approach has the following advantages:

*	**Speed boost through compilation**
	Evaluating the derivative and the core operations of the Runge–Kutta integration happen in compiled C code and thus very efficiently.
  
*	**Speed boost through symbolic optimisation**
	If your derivative is automatically generated by some routine, you can simplify it symbolically to boost the speed.
	In fact, blatant optimisations such as :math:`y·(x-x)=0` are done on the fly by SymPy.
	This is for example interesting if you want to simulate dynamics on a sparse network, as non-existing links are not taken into account when calculating the derivative when integrating.
	Moreover, multiple delay terms with the same delay can be handled efficiently, requiring only one look-up (see below).

*	**Automatically calculated Lyapunov exponents**
	As the derivative is provided symbolically, SymPy’s automatic derivation routines can be employed to calculate the Jacobian required for the DDE for the tangent vector required for calculating the Lyapunov expontents (see `lyapunov`).

*	**SymPy interface**
	You can enter your differential equations almost like you would on paper.
	Also, if you are working with SymPy anyway – e.g., to calculate fixed points –, you do not need to bother much with translating your equations.

If compilation fails to work for whatever reason, pure Python functions can be employed as a fallback (which is much slower, however).

For installation instructions, if you face issues with the compiler, want to optimise the speed, or wish to integrate network dynamics, take a look at the `common JiTC*DE documentation`_.

A brief mathematic background
-----------------------------

This documentation assumes that the delay differential equation (DDE) you want to solve is:

.. math::

	\dot{y} = f(t, y, y(t-τ_1), y(t-τ_2), …)

The gist of Shampine’s and Thompson’s method [ST01]_ is this:
The differential equation is integrated adaptively with the Bogacki–Shampine pair [BS89]_, like an ODE.
After every successful integration step, the state and derivative of the integration (which is an automatic by-product) are stored.
Whenever the derivative :math:`(f)` is evaluated, the required past states :math:`\left ( y(t-τ_1), y(t-τ_2), … \right )` are obtained through piece-wise cubic `Hermite interpolation <http://en.wikipedia.org/wiki/Hermite_interpolation>`_, using previously stored pairs of state and derivative (“anchor”).
In some extreme cases, they may also be extrapolated.
Note that unlike most other DDE softwares, JiTCDDE requires you to initiate the past in exactly this way, i.e., you have to give at least two such anchor points.

.. _example:

A simple example
----------------

.. automodule:: mackey_glass


.. _discontinuities:

Dealing with initial discontinuities
------------------------------------

As already examplified in `example`, :math:`\dot{y}` will usually be discontinuous at the start of the integration:
Before that time, it is directly defined by an Hermite interpolation of the anchors that you supply; afterwards, it is determined via evaluating :math:`f`.
As future values of :math:`f` depend on the past via the delay terms, it is also non-smooth at other times, namely :math:`τ_1, τ_2, …, 2τ_1, τ_1 + τ_2, 2τ_2, …`.
If an integration step contains one of these points, this may violate the conditions of Runge–Kutta integrations (for a low-order discontinuity) and makes the error estimate be very high, no matter the step size.
Fortunately, the discontinuities are quickly “smoothed out” (i.e., reduced in order) with time evolution and can then be ignored.
To make this happen, you have three options:

* `step_on_discontinuities` – This chooses the integration steps such that they fall on the discontinuities. In most cases, this is the easiest and fastest solution to this problem.

* `integrate_blindly` – This integrates the system for some time with a fixed step size, ignoring the error estimate. You have to take care that all parameters are reasonable. This is a good choice if you have a lot of different delays or time- or state-dependent delays.

* Carefully chosen initial conditions – of course, you can define the past such that the derivative for the last anchor is identical to the value of :math:`f` as determined with the anchors. To find such initial conditions, you usally have to solve a non-linear equation system. If you are not interested in the general dynamics of the system, but the evolution of a very particular initial condition, this may be given by default (otherwise your model is probably worthless).

Delays within the step
----------------------

If the delay becomes shorter than the step size, we need a delayed state to evaluate `f` before we have a final result for the required interpolation anchors.
With other words, the intergration step depends on its own result.

JiTCDDE addresses this problem mainly in the same manner as Shampine and Thompson [ST01]_:

* If reducing the step size by a small factor (`pws_factor`) makes it smaller than the delay, this is done.

* Otherwise, the result of an intergration step is calculated iteratively as follows:
	
	1. Attempt an integration step and **extrapolate** the required delayed states from the existing results.
	
	2. Attempt the same step again and **interpolate** the required delayed states using the result of the previous attempt.
	
	3. If the results of the last two attempts are identical within an absolute tolerance of `pws_atol` and relative tolerance of `pws_rtol`, accept the result of the last attempt. Otherwise go to step 2. If no such convergence has happened within `pws_max_iterations`, reduce the step size by `pws_factor`.

A problem of this approach is that as soon as it reduces the step size, the error estimates from the adaptive Runge–Kutta routines are not directly useful anymore since they almost always insist on increasing the step size.
Ignoring this may lead to useless integration steps (and thus wasted time) due to the step size being adapted back and forth.
Moreover, throttling step size increases (which is generally reasonable) may result in the step size being “trapped” at an unnecessary low value.
As far as I can tell, Shampine and Thompson [ST01]_ offer no solution to this.

To address this issue, JiTCDDE employs the following criteria for increasing the step size when the recommended step size (from the adaptive Runge–Kutta method) is larger than the current one:

* If the shortest delay is larger than the recommended step size, the step size is increased.

* If the calculating the next step took less than `pws_factor` iterations and the recommended step size is bigger than `pws_factor` times the shortest delay, the step size is increased.

* In all other cases, the step size is increased with a chance of `pws_base_increase_chance`.

To be precise, the above sharp criteria are intentionally blurred such that the probability to increase the step size continuously depends on the mentioned factors.
Finally, the parameter `pws_fuzzy_increase` determines whether the increase is actually depends on chance or is deterministic (which may be useful for some applications).
This parameter and the others mentioned above can be controlled with `set_integration_parameters`.

Time- and state-dependent delays
--------------------------------

There is nothing in JiTCDDE’s implementation that keeps you from making delays time- or state-dependent.
However, the error estimate is not accurate anymore as it does not take into account the inaccuracy caused by the changing delay.
This should not be a problem if your delays change sufficiently slowly.

Networks and large equations
----------------------------

JiTCDDE is specifically designed to be able to handle large delay differential equations, as they arise, e.g., in networks.
The caveats, tools, and tricks when doing this are the same as for JiTCODE; so please refer to its documentation, in particular the sections:

* `Handling very large differential equations <http://jitcode.readthedocs.io/en/latest/#handling-very-large-differential-equations>`_
* `A more complicated example <http://jitcode.readthedocs.io/en/latest/#module-SW_of_Roesslers>`_

.. _lyapunov:

Calculating Lyapunov exponents with `jitcdde_lyap`
--------------------------------------------------

`jitcdde_lyap` is a simple extension of `jitcdde` that almost automatically handles calculating Lyapunov exponents by evolving separation functions.
It works just like `jitcdde`, except that it generates and integrates additional differential equations for the separation functions.
After every call of `integrate`, the separation functions are orthonormalised, and the “local” Lyapunov exponents for this integration step are returned alongside with the system’s state.
These can then be further processed to obtain the Lyapunov exponents.
The separation functions are intialised with random data, and you have to take care of the preiterations that the separation functions require to align themselves.

The method employed here is similar to Farmer’s [F82]_, which in turn is an adaption of the method described by Benettin et al. [BGGS80]_ to delayed systems.
As the state of delayed systems is also defined by their recent past, one has to consider the past of tangent vectors (as used in Benettin et. al.) as well, which are called separation functions.
Farmer approximates these separation functions by fine equidistantly sampled recordings of the past on which he applies the standard scalar product for purposes of computing norms and orthonormalisation.
This approach does not translate well to adaptive step sizes as JiTCDDE employs.
Instead, JiTCDDE employs as a scalar product between two separation functions :math:`g` and :math:`h`:

.. math::

	\int_{t-τ_\text{max}}^t
	\mathcal{H}_f(\mathfrak{t}) \;
	\mathcal{H}_g(\mathfrak{t}) \;
	\mathrm{d} \mathfrak{t},

where :math:`\mathcal{H}` denotes the piecewise cubic Hermite interpolant (which is also used for obtaining past states).
The matrix induced by this scalar product can largely be calculated beforehand and thus the scalar product itself can be evaluated efficiently.
Note that for the limit of an infinitely fine sampling, this yields the same result as Farmer’s approach.

.. automodule:: mackey_glass_lyap

As the Lyapunov vectors (separation functions) are quite difficult to interpret, they are not returned as of now (if you need them, please `make a feature request <http://github.com/neurophysik/jitcdde/issues>`_).
There also is a class (`jitcdde_restricted_lyap`) that allows to calculate Lyapunov exponents for the dynamics transversal to some manifold (such as a synchronisation manifold).



Command reference
-----------------

.. automodule:: _jitcdde
	:members:
	:exclude-members: jitcdde, jitcdde_lyap, jitcdde_restricted_lyap

The main class
^^^^^^^^^^^^^^

.. autoclass:: jitcdde
	:members:
	:inherited-members:

Lyapunov exponents
^^^^^^^^^^^^^^^^^^

.. autoclass:: jitcdde_lyap
	:members:

.. autoclass:: jitcdde_restricted_lyap
	:members:

References
----------

.. _common JiTC*DE documentation: https://jitcde-common.readthedocs.io

.. [ST01] L.F. Shampine, S. Thompson: Solving DDEs in Matlab, Applied Numerical Mathematics 37, pp. 441–458 (2001), `10.1016/S0168-9274(00)00055-6 <http://dx.doi.org/10.1016/S0168-9274(00)00055-6>`_.

.. [BS89] P. Bogacki, L.F. Shampine: A 3(2) pair of Runge–Kutta formulas, Applied Mathematics Letters 2, pp. 321–325 (1989), `10.1016/0893-9659(89)90079-7 <http://dx.doi.org/10.1016/0893-9659(89)90079-7>`_.

.. [F82] J.D. Farmer: Chaotic attractors of an infinite-dimensional dynamical system, Physica D 4, pp. 366–393 (1982), `10.1016/0167-2789(82)90042-2 <http://dx.doi.org/10.1016/0167-2789(82)90042-2>`_.

.. [BGGS80]  G. Benettin, L. Galgani, A. Giorgilli, and J.-M. Strelcyn: Lyapunov Characteristic Exponents for smooth dynamical systems and for Hamiltonian systems; A method for computing all of them. Meccanica 15, pp. 9–30 (1980), `10.1007/BF02128236 <http://dx.doi.org/10.1007/BF02128236>`_.

.. _JiTCODE: http://github.com/neurophysik/jitcode

.. _JiTCODE documentation: http://jitcode.readthedocs.io

.. _SciPy’s ODE: http://docs.scipy.org/doc/scipy/reference/generated/scipy.integrate.ode.html

.. _SymPy Issue 4596: https://github.com/sympy/sympy/issues/4596

.. _SymPy Issue 8997: https://github.com/sympy/sympy/issues/8997


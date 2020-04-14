:tocdepth: 1

.. Please do not modify tocdepth; will be fixed when a new Sphinx theme is shipped.

.. note::
  This document describes the procedure used to measure the look-up
  table for the Auxiliary Telescope primary mirror support pressure
  as a function of elevation. We start with a description of the
  support system and the software used to control it. The data
  acquisition process is described along with the fit to the data
  and loading the results into the active optics system.

.. sectnum::

Introduction
============

The Auxiliary Telescope (AT) primary mirror (M1) is supported by 3 hard points
plus 21 pneumatic actuators. The actuators are connected to a pneumatic system
that regulates pressure on the line, which is uniformly distributed between
each element.

When pressure in the pneumatic system is turned off the mirror sits in the
3 hard points. One of the hard points (a.k.a. the load cell) contain a "digital
scale" that measures the load (how much weight) it supports.
With the mirror supported by the 3 hard points distort the shape of the mirror
resulting in considerable image aberration. The pneumatic system can provide
additional support to the mirror, spreading the load over its back surface,
mitigating the problem.

With different elevation angles, the normal support force on the mirror change,
decreasing as elevation decreases. This means that the pressure on the
actuators need to be adjusted with elevation to compensate for that effect.

To first order one expects that this will be a simple relation with the normal
vector. In a real case scenario, secondary effects in the system can cause
the pressure to deviate somewhat for a simple linear relation requiring
higher-order terms to account for systematics in the hardware.

There are 3 components that work together to account for this effect in the
Auxiliary Telescope (AT); ATPneumatics, AT Active Optics System (ATAOS) and
AT Mount Control System (ATMCS).

The ATPneumatics is the component is charge of regulating the pressure for the
pneumatics actuators. There are 3 valves involved when it comes to setting the
M1 support pressure: master air supply, instrument air supply and M1 air
valves. Once all 3 valves are opened there will a small back pressure applied
to M1 even if it is commanded to zero. It is important to have that in mind
since it will impact the way we measure the optimum pressure.

The ATAOS is the component in charge of storing the look-up table and setting
the M1 pressure in the ATPneumatics depending on the elevation. It works by
listening to the position of telescope, published by the ATMCS, and the current
M1 support pressure, published by the ATPneumatics. The elevation is used to
calculate the expected pressure, according to a polynomial equation. If the
computed value changes by more than a configurable threshold compared to the
current pressure, it will send a command to the ATPneumatics to update the M1
support pressure.

This document describes the procedure to measure the optimum M1 pressure for
different telescope elevations and fit the data with a polynomial equation
to obtain the coefficients required by the ATAOS component. We also describe
how to load the new coefficients into the ATAOS.


Measuring optimum M1 support pressure for different elevations
==============================================================

.. important::
   During this process it is important to make sure one does not lift the
   mirror from the hard points. This will happen if the value measured by
   the load cell gets close to zero. In this case, it is important to lower
   the mirror all the way back, setting the pressure to zero, close the
   M1 valve in the ATPneumatics, slew the telescope to the park position
   (:math:`\mathrm{elevation} = 86^\circ`), and restart the process.

As mentioned above, the support system contains 3 hard points and 21 actuators,
all have the same contact area with the mirror. The total mirror weight is
:math:`800lb` (:math:`\sim 362.9kg`). Nevertheless, as it is possible to note
furthermore, we developed a procedure that does not require knowing the total
weight of the mirror, using the load cell to measure the supporting weight.

In principle, to minimize distortion in the mirror shape we must make sure
the load is evenly distributed between all the 24 actuators. The first step in
the process is determining what it expected "optimum load", as measured by the
hard point load scale mentioned above. When there is no pressure in the
pneumatic system all the weight of the mirror will be supported by the 3 hard
points. That means the "total load" for a given elevation is 3x the one
measured by the load cell. Consequently, the "optimum load" will be the "total
load" divided by 24, the number of actuators.

.. math:: \mathrm{optimum\_load} = \frac{\mathrm{load\_cell} \times 3}{24}
   :label: eq-optimum-load

Once the optimum load is defined for a given elevation we need to determine
what is the corresponding pressure set point to the ATPneumatics needed.
In order to obtain the value for the optimum pressure we then perform a
binary search inside a sensible range.

.. note::
   As the pressure increases in the M1 support system the load decreases.

Assuming a lower and upper limit for the pressure of :math:`P_{min}` and
:math:`P_{max}`, the binary search for the optimum pressure is as follows:

   1. Set pressure to zero (:math:`P=0`) and measure load to determine
      :math:`\mathrm{optimum\_load}` using equation :eq:`eq-optimum-load`.

   2. Set pressure to :math:`P=P_{min}` and measure the load in the load cell.

      - If the load is lower than the optimum load, it means the minimum
        pressure is larger than the optimum value. The process is terminated.

   3. Set pressure to :math:`P=P_{max}` and measure the load in the load cell.

      - If the load is larger than the optimum load, it means the maximum
        pressure is lower than the optimum value, e.g. maximum was not correctly
        measured. The process is terminated.

   4. Compute the mid-pressure point :math:`P_{set}=(P_{min}+P_{max})/2.`.

   5. Set the pressure in the ATPneumatics to the mid-pressure point.

   6. Measure the mid-pressure load.

      - If the mid-pressure load is larger than the optimum load,
        set :math:`P_{max} = P_{set}`.
      - If the mid-pressure load is lower than the optimum load,
        set :math:`P_{min} = P_{set}`.

   7. Repeat steps 4, 5 and 6 until the relative difference between the
      mid-pressure load and the optimum load is smaller then 1%.

Before running this process for a range of elevations we need to define a
reliable procedure to determine the initial minimum and maximum pressures. For
the minimum pressure :math:`P_{min} = 0.` was taken as the starting point.

For the maximum pressure it is key that the procedure gives a higher enough
value that guarantees the optimum load will be larger than the load at
maximum pressure or the procedure will fail in step 3. It also must not be too
high that will cause the mirror to lift from the hard points.

To determine a safe maximum pressure we selected the lowest elevation in the
grid as a reference point (:math:`\mathrm{elevation} = 20^\circ`). Then, the
pressure was manually set until the measured load was lower than the optimum
load but still did not caused the mirror to lift. Assuming pressure is zero for
an :math:`\mathrm{elevation} = 0^\circ`, we then compute the linear coefficient
with the normal angle.

.. math:: c_{max} = \frac{P_{max}(20^\circ)}{\sin(20^\circ)}
   :label: eq-max-pressure-coeff

Then, the maximum pressure for each elevation is computed as;

.. math:: P_{max}(\mathrm{el}) = c_{max} \times \sin(\mathrm{el})
   :label: eq-max-pressure-eq

With this we proceed to measure the optimum pressure for 20 different
elevations, linearly spaced between :math:`20^\circ` and :math:`85^\circ`.

The result is shown alongside with the polynomial fit
in :numref:`fig-pressure-elevation`.

.. figure:: /_static/press_el.png
   :name: fig-pressure-elevation
   :target: ../_images/press_el.jpg
   :alt: pressure-elevation

   **Upper panel:** Data for the optimum M1 support pressure vs. elevation
   (blue dots) alongside a linear and a 7th-order polynomial fit with the
   normal gravity vector. **Lower panel:** Residuals when using the linear
   (solid blue line) and the 7th-order (solid orange line) polynomial fit.

In the bottom panel of :numref:`fig-pressure-elevation` we show the residue of
linear and 7th-order (solid orange line) polynomial fit. It is clear that there
are systematic residuals in the linear fit. We increased the order of the fit
until the residual was satisfactorily. Since there are no apparent small scale
structures we are not particularly worried about overfitting the data.

The output of the fit is shown in :numref:`table-fit-coeff`.

.. _table-fit-coeff:

.. table:: Coefficients for the linear and 7th-order polynomial fit.

   +-------------+--------------------+-----------------------+
   | Coefficient |  value - linear fit|  value - 7th order fit|
   +=============+====================+=======================+
   |a7           |                    |  -23093764.326        |
   +-------------+--------------------+-----------------------+
   |a6           |                    | +102195277.543        |
   +-------------+--------------------+-----------------------+
   |a5           |                    | -189824540.398        |
   +-------------+--------------------+-----------------------+
   |a4           |                    | +191664153.592        |
   +-------------+--------------------+-----------------------+
   |a3           |                    | -113529679.693        |
   +-------------+--------------------+-----------------------+
   |a2           |                    |  +39429444.370        |
   +-------------+--------------------+-----------------------+
   |a1           |         +127308.191|   -7299398.287        |
   +-------------+--------------------+-----------------------+
   |a0           |           -5780.049|    +577871.121        |
   +-------------+--------------------+-----------------------+

Loading values into ATAOS
=========================

The process to load a new configuration into the ATAOS is pretty standard.
It starts by cloning the
`configuration repo <https://github.com/lsst-ts/ts_config_attcs.git>` locally.


.. prompt:: bash

   git clone https://github.com/lsst-ts/ts_config_attcs.git


Create a ticket branch in the repo and create the file `hex_m1_hex_202003.yaml`
to host the configuration in the ATAOS configuration host in `ATAOS/v2`. Here
we choose to add only year and month to the name of the configuration file.
In case multiple configurations are generated over the course of a run, one
can think about appending a day and even an index to the file name.
Add the data in the third column of :numref:`table-fit-coeff` into the m1 session
of the file.

It is also important to edit the `_labels.yaml` file in the package and make
sure the new configuration is mapped to the `current` (or the first entry in
the file) tag. This will make sure the high level software will select the
configuration by default.

Once these changes are implemented, proceed with commit, push and open a PR to
the configuration package.

The final result can be found
`here <https://github.com/lsst-ts/ts_config_attcs/blob/8455be5887175cd4453fd95df7fe70565d50430b/ATAOS/v2/hex_m1_hex_202003.yaml#L3-L11>`__,
for the configuration file, and
`here <https://github.com/lsst-ts/ts_config_attcs/blob/v0.5.0/ATAOS/v2/_labels.yaml>`__,
for the labels.


.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :style: lsst_aa

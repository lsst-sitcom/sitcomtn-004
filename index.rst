..
  Technote content.

  See https://developer.lsst.io/restructuredtext/style.html
  for a guide to reStructuredText writing.

  Do not put the title, authors or other metadata in this document;
  those are automatically added.

  Use the following syntax for sections:

  Sections
  ========

  and

  Subsections
  -----------

  and

  Subsubsections
  ^^^^^^^^^^^^^^

  To add images, add the image file (png, svg or jpeg preferred) to the
  _static/ directory. The reST syntax for adding the image is

  .. figure:: /_static/filename.ext
     :name: fig-label

     Caption text.

   Run: ``make html`` and ``open _build/html/index.html`` to preview your work.
   See the README at https://github.com/lsst-sqre/lsst-technote-bootstrap or
   this repo's README for more info.

   Feel free to delete this instructional comment.

:tocdepth: 1

.. Please do not modify tocdepth; will be fixed when a new Sphinx theme is shipped.

.. sectnum::

.. TODO: Delete the note below before merging new content to the master branch.

.. note::

   **This technote is not yet published.**

   This tech note describes the implementation of the various Look-Up Tables (LUTs) for the Rubin Observatory Active Optics System (AOS). The subsystems involved are the M1M3, M2, the M2 hexapod, and the camera hexapod. Our objective is to make the various LUTs as consistent as possible in terms of functional forms, units, coordinate systems (CS), etc., because confusions over these later will be very costly.

   This note is meant to be a guide for the subsystem software developers, and for the scientists and engineers who will be tuning up the AOS and updating the LUTs, both during commissioning and operations.

.. Add content here.
.. Do not include the document title (it's automatically added from metadata.yaml).

############
Introduction
############

The LUTs of the various Active Optics System (AOS) components are of vital importance to the image quality of the Rubin Observatory.
It is expected that most of the on-sky commissioning time will be spent calibrating these LUTs.
And the improvements to the LUTs will be a long-term process lasting well into operations.
We will use the engineering data from initial science observations for trend analyses so that the various LUT components can be characterized more and more accurately over time.

The following subsystems need LUTs to offset the impacts of changing operational parameters,

- M1M3
- M2
- M2 hexapod
- Camera hexapod

Each subsystem LUT has various components, depending on the input variables,

- Elevation
- Static component
- Actuator weight
- Temperature (bulk, gradients)
- Azimuth
- Camera rotator angle

.. note::
    Here we are only concerned with the LUTs that are used during exposures. The M1M3 also has a dynamic LUT that are used during slewing, but it only affects the image quality indirectly (if the system doesn’t settle completely before exposure starts).

#########
Objective
#########

The implementation details we cover in this tech note are mostly about conventions, i.e., if we make different choices, it probably won't matter much to the performance of the AOS.
The main objective of this tech note is to make the implementation of the LUTs as consistent as possible across the components, while accommodating the unique design and functionalities of each.
This is crucial for avoiding confusions later, both during commissioning of the AOS, and in operations.

.. Important::

  It is important to acknowledge that the personel who would be using this system will have a diverse background, ranging from scientists, engineers, to software developers and telescope operators. As the result, people from different background may have different opinions on what is better and more intuitive.
  What we do in this tech note is that we try to balance the different perspectives.
  The more important thing is to have a standard, not what standard.


###############################
Elevation angle vs zenith angle
###############################

We should use elevation angle for elevation, instead of zenith angle.
The elevation angle is zero when the telescope points at horizon, and 90 deg when the telescope points at zenith.

It is OK if a subsystem or an analysis notebook uses zenith angle internally, if the variable name is self-explanatory. The input variable to the gravity component of the LUTs should be elevation angle. The output of data analysis, which would be passed to software developers to implement, should also use elevation angle.

The gravity components of the LUTs have different elevation ranges,

- [-8.5, 98.5] deg for M1M3. Occasionally M1M3 may go out of the [0, 90] deg range, and we want to make sure the glass mirror can still be supported properly. Hence the 8.5 deg margin on both sides.
- [-270, 90] deg for M2. In normal operations, M2 elevation will never go beyond the [0, 90] deg range. But M2 has a cart. For testing and engineering purposes, it needs to be able to be rotated by 360 deg. Beyond the [0, 90] deg range, glass safety is still the concern, but not image quality.
- [0, 90] deg for the M2 and camera hexapods. The hexapods may go out of the [0, 90] deg range. But we won't be imaging outside of that range. Unlike the mirrors, where outside of [0, 90] deg proper LUT forces are still needed to counter gravity, for hexapods, we do not have to compensate position offset due to gravity. If the elevation angle goes smaller than 0 deg, we will use LUT values for 0 deg. If it goes larger than 90 deg we will use LUT values for 90 deg.


.. _sec-p5:

#########################################
Functional Form of the gravity components
#########################################

We should use 5th order standard polynomials of the elevation angle (:math:`\theta_e`) for all the gravity components of the LUTs.

.. math::
    F_G = C_0 + C_1 \theta_e + C_2 \theta_e^2 + C_3 \theta_e^3 + C_4 \theta_e^4 + C_5 \theta_e^5

To avoid confusion, when the coefficients are output to a configuration file, they should always follow the order of

.. math::
  [C_0, C_1, C_2, C_3, C_4, C_5]

Other functional form for the gravity component which we also considered include:

- Nth order standard polynomials of the :math:`cosine` of the elevation angle (:math:`\cos(\theta_e)`),
- Fourier series up to N cycles in the [0, 90] deg range, and
- piecewise linear interpolation with input data at 5 deg increment.

In choosing which functional form to use, our main considerations are:

#. Consistency between the LUTs. If the needs of each component is drastically different, it is OK to use different functional forms for them. However, unless there are good reasons for doing so, we prefer to keep the functional form consistent between the LUTs. This will make future work for fine-tuning the LUTs using real measurements and updating the LUTs much more straightforward and less error-prone.
#. Simplicity. There may be many functional forms that can meet our accuracy requirements. Here the criteria is to make sure the error due to finite number of coefficients is much less than the non-repeating error of the corresponding degree of freedom (DOF). Document-36395 :cite:`Document-36395` shows an analysis that demostrates that the 5th order standard polynomials are accurate enough in all cases. And it is the simplest form among the above.
#. Various requirement documents (for example, LTS-88 :cite:`LTS-88` and LTS-206 :cite:`LTS-206`) specify that 5th order standard polynomials be used. We do not want to go through the change request process unless it is necessary.

##########################
Actuator weight components
##########################

For M1M3 and M2, the actuators have to supply the forces that can support their own weights first. Additional forces are then used to support and shape the glass. The LUTs therefore need to have actuator weight components. These forces can be calculated analytically, although they might not be as straightforward as one might first expect.

The calculations for M1M3 actuators (both single-axis and dual-axis actuators) are given in Document-32192 :cite:`Document-32192`. Measurements were also performed for standalone M1M3 actuators on the test bench in Tucson :cite:`Document-34907`. The results were consistent with the analytical calculations within measurement errors. We therefore use the results from the calculations in the final actuator weight component of the M1M3 LUT.

The M2 LUT actuator weight component was supplied by the M2 vendor Harris. The Rubin team plans to recalculate these forces to crosscheck the Harris results.

The M2 and the camera hexapods do not have actuator weight components in their LUTs because the output of the hexapod LUTs are positions instead of forces.

#################
Static components
#################

The static component of the LUT doesn't vary with external conditions. For the mirrors, these are the forces that are needed to bend out the low spatiall frequency factory figuring error. These were supplied by the vendors during factory acceptance testings. We will not change these components during commissioning and operations, unless somehow it can be proven that the figuring errors are different from what were determined at the factories.

As for the hexapods, the :math:`C_0` defined in Sec. :ref:`sec-p5` is the static component. For now, all six coefficients for the 5th order standard polynomial for each hexapod have been determined using results from FEA analyses. Once we have the hexapods mounted on the telescope mount assembly (TMA), we will use Laser Trackers (LTs) to calibrate the LUTs for both hexapods. It is expected that the calibrated values of :math:`C_0` will be quite different from the FEA values, because they depend on the actual installations of the hexapods on the TMA.

##################
Thermal components
##################

The software engineering user interfaces (EUIs) need to enable the users to toggle the thermal components of the LUTs on and off.
The thermal compensations are often hard to determine with good accuracy, especially for the mirrors.
This functionality will enable the users to easily determine whether the thermal LUTs are helping with the performance or not.

For now, the thermal LUTs only use the bulk temperature as the input variable. There is no plan to utilize the thermal gradients.
The functional form of the thermal compensations will be the 5th order standard polynomials, to comply with
LTS-88 :cite:`LTS-88` and LTS-206 :cite:`LTS-206`). All the thermal coefficients are set to zeros before we have good measurements of the thermal commpensations.

#############
Azimuth angle
#############

All the LUTs should have an azimuth component where the azimuth angle of the telescope is the input variable, to comply with
LTS-88 :cite:`LTS-88` and LTS-206 :cite:`LTS-206`.
The functional form of the azimuth angle dependence should also use a 5th order standard polynomial.
It is understood that a Fourier series will have the advantage of being continuous at 0/360 deg.
However, it is expected that the azimuth corrections will be small and not worth the complexity.

#############
Rotator angle
#############

Only the camera hexapod LUT needs to have a rotator angle component. This is due to the asymmetry in the camera mass distribution around the optical axis.

########################
Units, DOF names, and CS
########################

The outputs of the M1M3 and M2 LUTs are forces for individual actuators. The units should always be Newtons.
The CSs should be M1M3 CS and M2 CS, respectively. See SITCOMTN-003 :cite:`SITCOMTN-003` for definitions of these CSs.

The outputs of the hexapod LUTs are x, y, and z displacements and rotations around the x, y, and z axes.
The rotations follow the right-hand rule.
The units shoulld be microns for displacements and arcseconds for rotations.
Even though the Data Management (DM) standard for rotations are in degrees, we decide to make an exception here because we will be dealing with small angle all the time.\ [#label1]_

.. [#label1] By the same logic, DM uses arcsecond for small quantities like Point-Spread-Function size and platescale.

In the XML interface, we need to keep uniform naming to avoid confusions.
The x, y, and z displacements should have parameter names of dx, dy, and dz.
The rotations around the x, y, and z axes should have parameter names of rx, ry, and rz.
The parameter names having "d" and "r" in them indiciate that these are offset commands, not the new positions for the commanded components.
For clarity, these are required even when the topic name already indicates it is an offset command, since they are not much longer than x, y, z, u, v, and w.
The offsets are always in M2 CS :cite:`SITCOMTN-003` for the M2 hexapod, and CCS :cite:`SITCOMTN-003` for the camera hexapod.

We realize that the parameter names for mirror positions are not as consistent as one may wish.
Right now the M1M3 positions use units of meters and degrees, while M2 positions uses microns and arcseconds.
The parameter names are xPosition, yPosition, zPosition, xRotation, yRotation, and zRotation for M1M3.
For M2 they are x, y, z, xRot, yRot, and zRot.
Since these are only used in engineering modes, and not controlled by the AOS, they are less likely to cause confusions.
To reduce the amount of work for the developers we choose not to change these.

###########
Future work
###########

Things we need to do before the next round of testing:

- finish up FEA analysis on M1M3 gravity LUT, and make sure we account for the weights of all the interface plates and cups correctly; also revise Document-34898 :cite:`Document-34898` accordingly;
- perform FEA analysis on M2 gravity LUT;
- determine M2 actuator weight component and compare against Harris results;
- perform analysis to determine if M2 static forces from Harris make sense;
- change M2 gravity functional form to 5th order polynomial;
- remove Harris M2 LUT dependence on thermal gradients;
- all thermal components in the form of 5th order polynomial;
- check and ensure that we use the following everywhere in the XML

  - elevation angle instead of zenith angle (also revise SITCOMTN-003 :cite:`SITCOMTN-003` accordingly);
  - parameter names for offset commands: dx, dy, dz, rx, ry, rz;
  - units for hexapod offsets: microns and arcseconds.

Future milestones for LUT updates:

- M3 summit testing;
- LT testing of the M2 and camera hexapods on the TMA;
- Initial Optical Testing Assembly (IOTA) (if we eventually do get a time winidow);
- Commissioning Camera (ComCam);
- LSSTCam Full-Array Mode (FAM);
- LSSTCam normal operation mode (using four corner wavefront sensors).

#########################
Appendix A - Useful links
#########################

M1M3
####

gravity:

- https://github.com/lsst-ts/ts_m1m3support/blob/master/SettingFiles/Tables/ElevationXTable.csv
- https://github.com/lsst-ts/ts_m1m3support/blob/master/SettingFiles/Tables/ElevationYTable.csv
- https://github.com/lsst-ts/ts_m1m3support/blob/master/SettingFiles/Tables/ElevationZTable.csv

Azimuth (place holder for now):

- https://github.com/lsst-ts/ts_m1m3support/blob/master/SettingFiles/Tables/AzimuthXTable.csv
- https://github.com/lsst-ts/ts_m1m3support/blob/master/SettingFiles/Tables/AzimuthYTable.csv
- https://github.com/lsst-ts/ts_m1m3support/blob/master/SettingFiles/Tables/AzimuthZTable.csv

Thermal (place holder for now):

- https://github.com/lsst-ts/ts_m1m3support/blob/master/SettingFiles/Tables/ThermalXTable.csv
- https://github.com/lsst-ts/ts_m1m3support/blob/master/SettingFiles/Tables/ThermalYTable.csv
- https://github.com/lsst-ts/ts_m1m3support/blob/master/SettingFiles/Tables/ThermalZTable.csv

static:

- https://github.com/lsst-ts/ts_m1m3support/blob/master/SettingFiles/Tables/StaticXTable.csv
- https://github.com/lsst-ts/ts_m1m3support/blob/master/SettingFiles/Tables/StaticYTable.csv
- https://github.com/lsst-ts/ts_m1m3support/blob/master/SettingFiles/Tables/StaticZTable.csv

M2
##

Piecewise interpolation (at 5 deg increment) implemented by Harris:

- https://github.com/lsst-ts/ts_mtm2_cell/tree/master/configuration/lsst-m2/config/parameter_files/luts/FinalHandlingLUTs
- https://github.com/lsst-ts/ts_mtm2_cell/tree/master/configuration/lsst-m2/config/parameter_files/luts/FinalOpticalLUTs

Hexapods
########

Configurations:

- https://github.com/lsst-ts/ts_config_mttcs/tree/develop/Hexapod/v1

Fitter:

- https://github.com/lsst-ts/ts_hexapod/tree/develop/fitter



.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
    :style: lsst_aa

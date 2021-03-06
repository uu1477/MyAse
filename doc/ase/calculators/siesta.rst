.. module:: ase.calculators.siesta

======
SIESTA
======

Introduction
============

SIESTA_ is a density-functional theory code for very large systems
based on atomic orbital (LCAO) basis sets.


.. _SIESTA: http://www.uam.es/siesta/



Environment variables
=====================

The environment variable :envvar:`SIESTA_COMMAND` must hold the command
to invoke the siesta calculation. The variable must be a python format 
string with exactly two string fields for the input and output files.
Examples: "siesta < %s > %s", "mpirun -np 4 /bin/siesta3.2 < %s > %s".

A default directory holding pseudopotential files :file:`.vps/.psf` can be
defined to avoid defining this every time the calculator is used.
This directory can be set by the environment variable
:envvar:`SIESTA_PP_PATH`.

Set both environment variables in your shell configuration file:

.. highlight:: bash

::

  $ export SIESTA_COMMAND="siesta < ./%s > ./%s
  $ export SIESTA_PP_PATH=$HOME/mypps

.. highlight:: python

Both of these environment variables can be overwritten by setting:

===================== ========= ============= =====================================
keyword               type      default value description
===================== ========= ============= =====================================
``pseudo_path``       ``str``   ``None``      Directory for pseudopotentials to use
                                              None means using $SIESTA_PP_PATH
``siesta_executable`` ``str``   ``None``      Path to a Siesta executable. None
                                              means using the $SIESTA variable
===================== ========= ============= =====================================

For parallel calculations with four nodes set the keyword ``n_nodes=4``.


SIESTA Calculator
=================

The default parameters are very close to those that the SIESTA Fortran
code uses.  These are the exceptions:

==================== ========= ============= =====================================
keyword              type      default value description
==================== ========= ============= =====================================
``label``            ``str``   ``'siesta'``  Name of the output file
``mesh_cutoff``      ``float`` ``200*Ry``    Mesh cut-off energy in eV
``xc``               ``str``   ``'LDA'``     Exchange-correlation functional.
                                             Corresponds to either XC.functional
                                             or XC.authors keyword in SIESTA
``energy_shift``     ``float`` ``100 meV``   Energy shift for determining cutoff
                                             radii
``kpts``             ``list``  ``[1,1,1]``   Monkhorst-Pack k-point sampling
``basis_set``        ``str``   ``DZP``       Type of basis set ('SZ', 'DZ', 'SZP',
                                             'DZP')
``spin``             ``float`` ``COLLINEAR`` The spin approximation used, must be
                                             either ``UNPOLARIZED``, ``COLLINEAR``
                                             or ``FULL``
``species``          ``list``  ``[]``        A method for specifying a specific
                                             description for some atoms.
``pseudo_qualifier`` ``str``   ``None``      String for picking out specific type
                                             type of pseudopotentials. Giving
                                             ``example`` means that
                                             ``H.example.psf`` or
                                             ``H.example.vps`` will be used. None
                                             means that the XC.functional keyword
                                             is used, i.e. ``H.lda.psf``
==================== ========= ============= =====================================


Defining Custom Species
=======================
Standard basis sets can be set by the keyword ``basis_set`` directly, but for
anything more complex than one standard basis size for all elements,
a list of ``species`` must be defined. Each specie is identified by atomic
element and the tag set on the atom.

For instance if we wish to investigate a H2 and put ghost atom in the middle
with a special type of basis you would write:
    
>>> from ase.calculators.siesta.parameters import Specie, PAOBasisBlock
>>> from ase import Atoms
>>> from ase.calculators.siesta import Siesta
>>> atoms = Atoms(
>>>     '3H',
>>>     [(0.0, 0.0, 0.0),
>>>      (0.0, 0.0, 0.5),
>>>      (0.0, 0.0, 1.0)],
>>>     cell=[10, 10, 10])
>>> atoms.set_tags([0, 1, 0])
>>>
>>> basis_set = PAOBasisBlock(
>>> """1
>>> 0  2 S 0.2
>>> 0.0 0.0""")
>>>
>>> siesta = Siesta(
>>>     species=[
>>>         Specie(symbol='H', tag=None, basis_set='SZ'),
>>>         Specie(symbol='H', tag=1, basis_set=basis_set, ghost=True)])
>>>
>>> atoms.set_calculator(siesta)
 
The priority order of which description is used is that species
defined with a tag has the highest priority. Then general species
with ``tag=None`` has a lower priority. Finally, if no species apply
to an atom, the general calculator keywords are used.

Species can also be used to specify pseudopotentials:

>>>         Specie(symbol='H', pseudopotential='H.example.psf'),

Both absolute and relative paths can be given.
Relative paths are considered relative to the default pseudopotential
path.


Extra FDF parameters
====================

The SIESTA code reads the input parameters for any calculation from a
:file:`.fdf` file. This means that you can set parameters by manually setting
entries in this input :file:`.fdf` file. This is done by the argument:

>>> Siesta(fdf_arguments={'variable_name': value, 'other_name': other_value})

For example, the ``DM.MixingWeight`` can be set using

>>> Siesta(fdf_arguments={'DM.MixingWeight': 0.01})

The explicit fdf arguments will always override those given by other
keywords, even if it will break calculator functionality.
The complete list of the FDF entries can be found in the official `SIESTA
manual`_.

.. _SIESTA manual: http://departments.icmab.es/leem/siesta/Documentation/Manuals/manuals.html


Pseudopotentials
================

Pseudopotential files in the ``.psf`` or ``.vps`` formats are needed.
Pseudopotentials generated from the ABINIT code and converted to
the SIESTA format are available in the `SIESTA`_ website . A database of user
contributed pseudopotentials is also available there.

You can also find an on-line pseudopotential generator_ from the
OCTOPUS code.

.. _generator: http://www.tddft.org/programs/octopus/wiki/index.php/Pseudopotentials


Example
=======

Here is an example of how to calculate the total energy for bulk Silicon,
using a double-zeta basis generated by specifying a given energy-shift:

>>> from ase import Atoms
>>> from ase.calculators.siesta import Siesta
>>> from ase.units import Ry
>>>
>>> a0 = 5.43
>>> bulk = Atoms('Si2', [(0, 0, 0),
>>>                      (0.25, 0.25, 0.25)],
>>>              pbc=True)
>>> b = a0 / 2
>>> bulk.set_cell([(0, b, b),
>>>                (b, 0, b),
>>>                (b, b, 0)], scale_atoms=True)
>>>
>>> calc = Siesta(label='Si',
>>>               xc='PBE',
>>>               mesh_cutoff=200 * Ry,
>>>               energy_shift=0.01 * Ry,
>>>               basis_set='DZ',
>>>               kpts=[10, 10, 10],
>>>               fdf_arguments={'DM.MixingWeight': 0.1,
>>>                              'MaxSCFIterations': 100},
>>>               )
>>> bulk.set_calculator(calc)
>>> e = bulk.get_potential_energy()

Here, the only input information on the basis set is, that it should
be double-zeta (``basis='DZP'``) and that the confinement potential
should result in an energy shift of 0.01 Rydberg (the
``energy_shift=0.01 * Ry`` keyword). Sometimes it can be necessary to specify
more information on the basis set.


Restarting from an old Calculation
==================================

If you want to rerun an old SIESTA calculation, made using the ASE
interface or not, you can set the keyword ``restart`` to the siesta ``.XV``
files. The keyword ``ignore_bad_restart`` (True/False) will decide whether
a broken file will result in an error(False) or the whether the calculator
will simply continue without the restart file.


Further Examples
================
See also ``ase/test/calculators/siesta/test_scripts`` for further examples
on how the calculator can be used.

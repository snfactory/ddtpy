CubeFit
=======

[![Join the chat at https://gitter.im/snfactory/cubefit](https://badges.gitter.im/Join%20Chat.svg)](https://gitter.im/snfactory/cubefit?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)

Fit supernova + galaxy model on a Nearby Supernova Factory spectral data cube.


Installation
------------

To install a release version:

```
pip install http://github.com/snfactory/cubefit/archive/v0.2.0.tar.gz
```

Release versions are listed
[here](http://github.com/snfactory/cubefit/releases). CubeFit has the
following dependencies:

- Python 2.7 or 3.3+
- numpy
- scipy (for optimization)
- [fitsio](https://github.com/esheldon/fitsio) (for FITS file I/O)
- [pyfftw](http://hgomersall.github.io/pyFFTW) (for fast FFTs)


Usage
-----

Fit the model, write output to given file:

```
cubefit config.json output.fits
```

Producing subtracted data cubes is a separate step:

```
cubefit-subtract config.json output.fits
```

This reads the input and output filenames listed in the configuration file, and
the results of the fit saved in `output.fits`.

With either command, you can run with `-h` or `--help` options to see all the
optional arguments: `cubefit --help`.

Input format
------------

CubeFit requires the following keys in the input JSON file. Keys are
case-sensitive. Additional keys are simply ignored.

| Parameter        | Type   | Description                           |
| ---------------- | ------ | ------------------------------------- |
| `"filenames"`    | *list* | list of FITS data cube files
| `"airmasses"`    | *list* | airmass for each epoch
| `"pressures"`    | *list* | pressure for each epoch **[mmHg]**
| `"temperatures"` | *list* | temperature for each epoch **[deg Celcius]**
| `"xcenters"`     | *list* | x position of MLA center relative to SN **[spaxels]**
| `"ycenters"`     | *list* | y position of MLA center relative to SN **[spaxels]**
| `"ha"`           | *list* | HA (?) of instrument **[degrees]**
| `"dec"`          | *list* | declination of the target **[degrees]**
| `"psf_params"`   | *list of lists* | PSF parameters for each epoch
| `"refs"`         | *list* | index of final refs in lists **[0-based indexing]**
| `"master_ref"`   |        | index of "master" final ref **[0-based indexing]**
| `"mla_tilt"`     |        | MLA tilt **[radians]** (?)


Parameter only used in `cubefit-subtract`:

| Parameter        | Type   | Description                           |
| ---------------- | ------ | ------------------------------------- |
| `"outnames"`     | *list* | Target filenames for galaxy-subtracted cubes


**NOTE:** the meaning of `xcenters` and `ycenters` is different than in DDT
config files!

Output format
-------------

CubeFit outputs a FITS file with two extensions.

- **HDU 1:** (image) Full galaxy model. E.g., a 32 x 32 x 779 array.

- **HDU 2:** (binary table) Per-epoch results. Columns:
  - `yctr` Fitted y position of data in model frame
  - `xctr` fitted x position of data in model frame
  - `sn` (1-d array) fitted SN spectrum in this epoch
  - `sky` (1-d array) fitted sky spectrum in this epoch
  - `galeval` (3-d array) Galaxy model evaluated on this epoch
    (e.g., 15 x 15 x 779)
  - `sneval` (3-d array) SN model evaluated on this epoch (e.g., 15 x 15 x 779)


- The **HDU 1 header** contains the wavelength solution in the "CRPIX3",
  "CRVAL3", "CDELT3" keywords. These are simply propagated from input
  data cubes.

- The **HDU 2 header** contains the fitted SN position in the model frame in the
  "SNX" and "SNY" kewords.

Here is an example demonstrating how to reproduce the full model of the scene
for the first epoch:

```python
>>> import fitsio
>>> f = fitsio.FITS("PTF09fox_B.fits")
>>> f[1]

  file: PTF09fox_B.fits
  extension: 1
  type: BINARY_TBL
  extname: epochs
  rows: 17
  column info:
    yctr                f8  
    xctr                f8  
    sn                  f4  array[779]
    sky                 f4  array[779]
    galeval             f4  array[15,15,779]
    sneval              f4  array[15,15,779]

>>> epochs = f[1].read()
>>> i = 0
>>> scene = epochs["sky"][i, :, None, None] + epochs["galeval"][i] + epochs["sneval"][i]
```

What it does
------------

**The data**

The data are N 3-d cubes, where N are observations at different times
(typically N ~ 10-30).  Each cube has two spatial directions and one
wavelength direction (typical dimensions 15 x 15 x 800). Each cube has
both a data array and a corresponding weight (error) array.

**The model**

The model parameters consist of:

- 3-d galaxy model ("unconvolved"). Default size is approximately 32 x
  32 x 800, where the pixel size of the model matches that of the data
  (both spatially and in wavelength) but the model extends past the
  data spatially.
- N 1-d sky spectra
- N 1-d supernova spectra
- position of each of the N data cubes relative to a "master final
  ref": 2 x (N-1) parameters
- position of the SN in the "master final ref" frame of reference: 2
  parameters

Additionally there are inputs that describe "atmospheric
conditions". These are necessary for propagating the model to the
data. They are derived externally and are not varied in the fit:

- Amount and direction of atmospheric differential refraction in each
  observation.
- wavelength-dependent PSF model for each observation.

Finally, there are two regularization parameters that determine the
penalty on the galaxy model for being "rougher" (having a high
pixel-to-pixel variation).

**The Procedure**

These steps are carried out in ``cubefit.main`` (which is called from
the command-line script).

1. **Initialization**

   - Read in datacubes
   - Read in parameters that determine the PSF and ADR and initialize the
     "atmospheric model" that contains these for each epoch.
   - Initialize all model parameters to zero, except sky. For the sky, we
     make an initial heuristic guess based on only the data and a
     sigma-clipping process.
   - Initialize the regularization, based on the input regularization
     parameters and a rough guess at the average galaxy spectrum.

2. **Fit the galaxy model to just the master final ref**

   The model is defined in the frame of the master final ref, so there
   is no position fit here. Also, the sky of the master final ref is
   fixed to the initial guess (as the galaxy model and sky level are
   degenerate).

3. **Fit the position and sky of the remaining final refs**

4. **Re-fit galaxy model using all final refs**

5. **Fit position, sky and SN position of all the non-final refs**


Internal API documentation
--------------------------

|                |                  |
| -------------- | ---------------- |
| `cubefit.main` | Main entry point |


**Data structure and I/O**

|                         |                                          |
| ----------------------- | ---------------------------------------- |
| `cubefit.DataCube`      | Container for data and weight arrays     |
| `cubefit.read_datacube` | Read a two-HDU FITS file into a DataCube |


**ADR and PSF model**

|                                    |                                          |
| ---------------------------------- | ---------------------------------------- |
| `cubefit.paralactic_angle`         | Return paralactic angle in radians, including MLA tilt |
| `cubefit.gaussian_plus_moffat_psf` | Evaluate a gaussian+moffat function on a 2-d grid |
| `cubefit.psf_3d_from_params`       | Create a wavelength-dependent Gaussian+Moffat PSF from given parameters |
| `cubefit.AtmModel`                 |  Atmospheric conditions (PSF and ADR) model for a single observation |

*Additionally, the ADR class from the SNfactory Toolbox package is used.*

**Fitting**

|                                     |                                          |
| ----------------------------------- | ---------------------------------------- |
| `cubefit.RegularizationPenalty`     | Callable that returns the penalty and gradient on it. |
| `cubefit.guess_sky`                 | Guess sky based on lower signal spaxels compatible with variance |
| `cubefit.fit_galaxy_single`         | Fit the galaxy model to a single epoch of data. |
| `cubefit.fit_galaxy_sky_multi`      | Fit the galaxy model to multiple data cubes. |
| `cubefit.fit_position_sky`          | Fit data position and sky for a single epoch (fixed galaxy model). |
| `cubefit.fit_position_sky_sn_multi` | Fit data pointing (nepochs), SN position (in model frame), SN amplitude (nepochs), and sky level (nepochs). |



**Utilities**

|                               |                                          |
| ----------------------------- | ---------------------------------------- |
| `cubefit.fft_shift_phasor_2d` | phasor array used to shift an array (in real space) by multiplication in fourier space. |
| `cubefit.plot_timeseries`     | Return a figure showing data and model. |


Developer Documentation
-----------------------

**Running Tests:**

If you've clone the repository (rather than using pip to install), you
can run tests with `./test.py`. Requires the `pytest` package
(available via pip or conda).


License
-------

CubeFit is released under the MIT license. See LICENSE.md.

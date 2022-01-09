# Background

This data-cube contains the centroid, amplitude, sigma and skew values of the spectral lines retrieved using NEID Solar Data from December 2020 to August 2021.

## Structure
The architecture of the cube, a Numpy nested dictionary, is as follows:
```python
# DATA CUBE FORMAT
# For each spectral order from Order 40 - 110,
# 
# File Name (Date-Time)
#    Order
#       Line  
#          centroid (in angstroms)
#          centroid_pixval (in pixels)
#          norm_amplitude (normalized depth)
#          elec_amplitude (depth in electron flux)
#          sigma 
#          skew
```

## Imports
Since the data-cube is a Numpy nested dictionary, we need to import Numpy to work with the values. Matplotlib is also imported for visualizations:
```python
import numpy as np
import matplotlib.dates as mdates
import matplotlib.pyplot as plt
%matplotlib inline
```

## Using the Data
```python
# Reading the data cube (Replace 'x' with latest version, '2.0.0' for the noon data cube)
data = np.load('data_cube_vx.npy',allow_pickle='TRUE').item()

# Returns the entire cube
print(data)

# Working with Values
# The example below would give you the centroid value for the 3rd Line in Order 50 for May 9, 2021 @ 19:52:26
centroid = data['Time 20210509T195226')]['Order 50']['Line 3']['centroid']
```

## Working with Date-times and Plots
The datetime values from the cube need to be converted to a format that is 'plotting-friendly' in order to be visualized correctly:
```python
# Organize datetimes
datetimes = []
for x in data:
    x = x.replace('Time ', '')
    datetimes.append(x)
datetimes.sort()


# Make datetimes 'readable' for visualization
import datetime
formatted = []
np_datetimes = np.array(datetimes)
for value in np_datetimes:
    d = datetime.datetime.strptime(value, '%Y%m%dT%H%M%S')
    formatted.append(d)
dates = mdates.date2num(formatted) # <- This would be your x-axis when plotting 
```

A plot may therefore be formatted as:
```python
plt.plot_date(dates, y, color='green', label='Values over Time')
```

# Methods used for Aquiring the Data

Here I will explain the methods for which I have arrived at the values in this data-cube. More information regarding this process can be found in the Main.ipynb notebook on GitHub.

## Skewed Gaussian Model
A wave chunk within a 15-pixel range is created using a list of reference line values. The wave chunk along with the normalized upside values are passed into a Skewed Gaussian Model. This model was used from the following library:
```python
# Import Models for Fitting
from lmfit.models import GaussianModel, SkewedGaussianModel, VoigtModel
```
```python
wchunk = wave[order,(rounded_ref_centroid_pixval-15):(rounded_ref_centroid_pixval+15)]
xchunk = np.arange(len(schunk))

mod = SkewedGaussianModel()
pars = mod.guess(upside, x=wchunk)
try:
    out = mod.fit(upside, pars, x=wchunk)
except ValueError:
    continue
```

## Centroid Values
The centroid values, both in Angstrom and pixel units, can therefore be accessed from the model as so:
```python
centroid = pars['center'].value

# Interpolating to arrive at the corresponding pixel value
centroid_pixval = np.interp(centroid, wchunk, skgauss_x)
```

## Amplitude Values
The normalized amplitude, out of a scale of 1.0, is found using the maximum, or peak, value of the Skewed Gaussian fit Model:
```python
# Normalized Amplitude
norm_amplitude = np.amax(out.best_fit, axis=0)
```

## Sigma
The sigma value is simply determined from the output of the normalized skewed model:
```python
mod2 = SkewedGaussianModel()
pars2 = mod2.guess(skgauss_y, x=skgauss_x)
out2 = mod2.fit(skgauss_y, pars2, x=skgauss_x)
sigma = pars2['sigma'].value
```

## Skew
The Skewed Gaussian model does not include skew as a default parameter. Thus, I converted the data into a pandas DataFrame and extrapolated the skew value from the it.

To use pandas, import the following library:
```python
import pandas as pd
```
```python
get_skew = pd.DataFrame(out.best_fit).skew()
skew = get_skew[0]
```

## License
[NEID Archive Data](https://neid.ipac.caltech.edu/search.php)

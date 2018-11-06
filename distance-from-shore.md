Source: http://oos.soest.hawaii.edu/thredds/ncss/dist2coast_1deg_ocean/dataset.html

## Install GDAL 2.1
Run `gdal-config --version` to check for the presence of GDAL >= 2.1
I needed to update.  I installed using these instructions

https://www.karambelkar.info/2016/10/gdal-2-on-mac-with-homebrew/

Also need to install the gdal python tools with 

```console
brew install gdal2-python
brew link --force gdal2-python
```

## Download the distance raster
- Download from http://oos.soest.hawaii.edu/thredds/ncss/dist2coast_1deg_ocean/dataset.html
- Select the variable `dist = distance to nearest coastline` and Submit
- Submit the request
- You should get a file called `dist2coast_1deg_ocean.nc`
This file is NetCDF format.  This is supported by gdal 2.1

## Verify the downloaded file
```console
Abelard:dist-raster paul$ gdal-config --version
2.2.2
Abelard:dist-raster paul$ gdalinfo dist2coast_1deg_ocean.nc 
Driver: netCDF/Network Common Data Format
Files: dist2coast_1deg_ocean.nc
Size is 36000, 18000
Coordinate System is `'
Origin = (-180.005000000076308,90.004999999940651)
Pixel Size = (0.010000000152592,-0.009999999881314)
...
Corner Coordinates:
Upper Left  (-180.0050000,  90.0050000) 
Lower Left  (-180.0050000, -89.9949979) 
Upper Right ( 179.9950055,  90.0050000) 
Lower Right ( 179.9950055, -89.9949979) 
Center      (  -0.0049973,   0.0050011) 
Band 1 Block=36000x1 Type=Int16, ColorInterp=Undefined
  NoData Value=0
  Unit Type: km
  Metadata:
    coordinates=lat lon 
    long_name=distance to nearest coastline
    NETCDF_VARNAME=dist
    units=km
    _FillValue=0
```
## Convert
- change file format to TIFF
- add wgs84 srs
- Remove nodata so the 0 values for onshore areas are exposed
- convert data type from Int16 to Int32

```console
gdal_translate -a_nodata none -a_srs wgs84 -ot Int32  \
 dist2coast_1deg_ocean.nc  dist2coast_1deg_ocean.tif
```

## Calc new values
- multiply every pixel by 1000 to convert from km to meters
```console
 gdal_calc.py -A dist2coast_1deg_ocean.tif \
  --outfile=dist2coast_1deg_ocean_meters.tif --calc="A*1000"
```
## Convert again
- match up to the way `gs://benthos-pipeline/data-dependencies/distance-from-shore.tif` is structured
- set nodata to -9999 (don't know why...)
- set datatype to Float32
- set Tiling to make 256x256 tiles internally.  NOTE:  important to do this AFTER gdal_calc.py because the calc is MUCH slower working on a tiled tiff

```console
gdal_translate -a_nodata -9999 -ot Float32 -co "TILED=YES" -co "COMPRESS=DEFLATE"  \
  dist2coast_1deg_ocean_meters.tif  dist2coast_1deg_ocean_meters_tiled.tif
```

## Verify 
- Make sure that the new pixel values are 1000x larger than the old ones

```console
$ gdallocationinfo -wgs84 -b 1 dist2coast_1deg_ocean.tif 0 0
Report:
  Location: (18000P,9000L)
  Band 1:
    Value: 572

$ gdallocationinfo -wgs84 -b 1 dist2coast_1deg_ocean_meters_tiled.tif 0 0

Report:
  Location: (18000P,9000L)
  Band 1:
    Value: 572000
```
- Compare to the old raster
```console
$ gdallocationinfo -wgs84 -b 1 distance-from-shore.tif 0 0
Report:
  Location: (25582P,12080L)
  Band 1:
    Value: 572909.1875
```

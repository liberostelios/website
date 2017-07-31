---
layout: post
title:  "Messing around with (City)GML on GDAL 2.2"
date:   2017-07-24 23:08:22 +0200
categories: citygml gdal
---
Recently, during a very interesting workshop at [FOSS4G Europe 2017](http://europe.foss4g.org), I came across the new [GMLAS](http://www.gdal.org/drv_gmlas.html) driver introduced in the latest GDAL 2.2.0 release. This driver promises some real support for parsing `.gml` files through OGR, contrary to the previous pretty basic [GML](http://www.gdal.org/drv_gml.html) driver that didn't really serve its purpose. This, allows us to play around with CityGML files both through GDAL's command-line tools, as well as through all supported languages of the library (for instance, you can use the Python wrapper to load some GML documents now).

The new driver has a really interesting approach: instead of trying to parse the whole document as one list of features, it analyzes the schema definitions (through `.xsd` files) and creates a relational representation of all features. Therefore, features are grouped as `layers` (you could think of them as `tables` in an RDBMS) and they are linked to each.

How does it work? Simply add the `GMLAS:` prefix before a filename, when using any GDAL tool or library, and the new driver will be put in charge. So, for instance, we can simply analyze the containing feature layers by running:

{% highlight bash %}
ogrinfo GMLAS:/path/to/file.gml
{% endhighlight %}

Workaround for GDAL 2.2 and CityGML
===================================

While the driver works wonderfully for INSPIRE datasets (which was its original purpose), there are still some small details related more complex implementations of GML documents. CityGML is one of those schemas that really stretches some of the GML mechanisms to their limits, so we can anticipate some datasets not being so trivial to be parsed by the driver. Indeed, when I first tried to load some city models I found it didn't load all feature classes.

Thankfully, [Even Rouaoult](https://github.com/rouault) was there to work on some of those issues and they have already been fixed and scheduled for the next release (so expect GDAL 2.3, maybe, to work with CityGML out-of-the-box).

Meanwhile, there is a small workaround to make CityGML schema work with GDAL 2.2:
- Run `ogrinfo` once against a CityGML file (if it fails, check the solution on the fixing missing schema locations)
- Go to your home folder (e.g., `/home/your_username/` on Linux) and find the `.gdal/gmlas_xsd_cache` folder. There must be several `.xsd` files in there, including all files describing the CityGML schema.
- Open all files of CityGML (they are of `schemas.opengis.net_citygml_V.0_MODULE.xsd` format) and inside the `xs:schema` tag duplicate the first `xmlns`. Then, add the prefix of this module to the second instance. So, for instance, if this is the base schema file (the core module), then the original tag should be changed from

    ~~~ xml
    <xs:schema xmlns="http://www.opengis.net/citygml/1.0" xmlns:xAL="urn:oasis:names:tc:ciq:xsdschema:xAL:2.0" xmlns:xs="http://www.w3.org/2001/XMLSchema" xmlns:gml="http://www.opengis.net/gml" targetNamespace="http://www.opengis.net/citygml/1.0" elementFormDefault="qualified" attributeFormDefault="unqualified">
    ~~~

    to

    ~~~ xml
    <xs:schema xmlns="http://www.opengis.net/citygml/1.0" xmlns:core="http://www.opengis.net/citygml/1.0" xmlns:xAL="urn:oasis:names:tc:ciq:xsdschema:xAL:2.0" xmlns:xs="http://www.w3.org/2001/XMLSchema" xmlns:gml="http://www.opengis.net/gml" targetNamespace="http://www.opengis.net/citygml/1.0" elementFormDefault="qualified" attributeFormDefault="unqualified">
    ~~~

    Notice, how the new `xmlns:core` argument is repeating the same schema. You should do this for all modules (or at least those that your dataset uses), for instance for the building module adding the `xmlns:bldg` property.

You have to do this only once and only if you are using GDAL 2.2.0 or 2.2.1. Of course, you have to repeat the same for every different CityGML version you parse in the future (again, if you are not in a newer than 2.2.1 version).

Fix missing schema locations
============================

In order for GMLAS to work, the `xsi:schemaLocation` property, which links to the `.xsd` files that describe the schema, must be set in the target file. I noticed that many datasets do not contain this, which does not allow the driver to analyze the schema. In this case, when calling GMLAS you will get the following error:

{% highlight bash %}
ERROR 1: No schema locations found when analyzing data file: XSD open option must be provided
{% endhighlight %}

There is an easy workaround for that, as you can provide the files manually through the `XSD=` option when opening the file. For instance, for a CityGML 1.0 file you may use this command:

{% highlight bash %}
ogrinfo GMLAS:input.gml -oo XSD=http://schemas.opengis.net/citygml/landuse/1.0/landUse.xsd,http://schemas.opengis.net/citygml/cityfurniture/1.0/cityFurniture.xsd,http://schemas.opengis.net/citygml/texturedsurface/1.0/texturedSurface.xsd,http://schemas.opengis.net/citygml/transportation/1.0/transportation.xsd,http://schemas.opengis.net/citygml/building/1.0/building.xsd,http://schemas.opengis.net/citygml/waterbody/1.0/waterBody.xsd,http://schemas.opengis.net/citygml/relief/1.0/relief.xsd,http://schemas.opengis.net/citygml/vegetation/1.0/vegetation.xsd,http://schemas.opengis.net/citygml/cityobjectgroup/1.0/cityObjectGroup.xsd,http://schemas.opengis.net/citygml/generics/1.0/generics.xsd
{% endhighlight %}

Cool stuff to do with GMLAS
===========================

Once we can load a file with GMLAS, we can do all sort of cool stuff with it. So, we can transform a CityGML file or one feature class to another format, which makes it possible to view the dataset (as 2D) in QGIS or other GIS clients. If you need to convert the whole schema in a relational database and you happen to have a PostGIS installation around, you can do something like this:

~~~ bash
ogr2ogr -f PostgreSQL PG:dbname=gmldb GMLAS:/path/to/file.gml -lco SCHEMA=citymodel -oo REMOVE_UNUSED_LAYERS=YES
~~~

This will read the whole city model and create a releational equivalent in the `citymodel` schema of the PostGIS database called `gmldb`. Notice the open option `REMOVE_UNUSED_LAYERS=YES`, which will only create tables for the features that are used otherwise the driver will create a few hundred empty tables which are described by the CityGML schema but probably are useless for this dataset.

Another cool thing is that you can even try to convert a specific "layer" (feature class) of the city model to a simple file format (e.g. GeoJSON, Shapefile). For instance, this is how you can get the footprints of buildings from a dataset that uses multi-surfaces to describe buildings:

~~~ bash
ogr2ogr -f GeoJSON /path/to/output.json gmlas:/path/to/input.gml -oo REMOVE_UNUSED_LAYERS=YES -oo REMOVE_UNUSED_FIELDS=YES -sql "SELECT * FROM groundsurface"
~~~

You can change the `sql` option to pick any feature class is useful for your application, or you can try to change the output driver to a shapefile, for instance. You may also use the `nlt` option to transform a geometry to another type. You can read more about `ogr2ogr` options in its [documentation page](http://www.gdal.org/ogr2ogr.html).
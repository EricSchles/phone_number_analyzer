#Idea for the project - phone number analyzer

Currently there is a lot of decoy ads being posted.  Buyers of sex click on these ads and we capture their phone numbers.  The problem is, what do we do with this data once it's been collected.  Sure we can identify the individuals and if they pop-up frequently enough we can say they are probably addicts and then get them help, or arrest them, depending.  But what else can be done?

The intention of the following tool is to address just this:

A GIS system that expects a CSV of phone numbers as input - takes the area code and first 3 digits of the number and then translate them to an approximate geological coordinate pair which is then visualized on a GIS map.  The goal would be to get an approximate location for all the phone numbers and then have frequency counts for each area of the map.  Another useful visual feature would be to see all the people assigned to each area.  The number of phone numbers should change over time with the following filters:

* total numbers per area over time - numbers never go down
* total numbers for the past week  - numbers can go up or down over time
* total numbers for the past day

This will allow the team to answer the following questions:

1) How has enforcement worked over time?
2) What areas are most affected now?
3) Which members of the community are most effective in reducing demand?


##Building/Using a GIS system, for free

There are a lot of solutions out there for python and GIS.  GeoDjango is the most packaged, with the least amount of extra installs, that I know of.  

First we'll install Geodjango, which luckily comes with Django1.8.  So I'll you'll need to do really is install django, instructions on how to do that can be found [here](https://docs.djangoproject.com/en/1.8/topics/install/), assuming you want the source or anything else.

But really all you need to do is: `sudo pip install django`

Before we can get started with building we'll need to install PostGIS, steps on how to do that can be found [here](http://www.saintsjd.com/2014/08/13/howto-install-postgis-on-ubuntu-trusty.html)

Next we'll need to install the postgres python module - pyscopg2: `sudo pip install pyscopg2`

Next all we need to do is set up GeoDjango to work with PostGIS, and we are off to the races.

Unfortunately the GeoDjango documentation for the first example is confusion and leaves out a few steps, so first we'll have to correct that:

Please note these directions are for Ubuntu 14.04

Installation:

Python:
`sudo pip install django #make sure you are installing django 1.8` 

`sudo pip install pyscopg2`

PostGres:

`sudo apt-get update`

`sudo apt-get install -y postgresql postgresql-contrib`

Testing postgres install:

`sudo -u postgres createuser -P USER_NAME_HERE`

`sudo -u postgres createdb -O USER_NAME_HERE DATABASE_NAME_HERE`

`psql -h localhost -U USER_NAME_HERE DATABASE_NAME_HERE`

Adding PostGIS support:

`sudo apt-get install -y postgis postgresql-9.3-postgis-2.1`

`sudo -u postgres psql -c "CREATE EXTENSION postgis; CREATE EXTENSION postgis_topology;" DATABASE_NAME_HERE`

changing everything to trusted, rather than requiring authentication - DO THIS FOR LOCAL DEVELOPMENT ONLY!!!

`sudo emacs /etc/postgresql/9.1/main/pg_hba.conf`

Change line:

`local   all             postgres                                peer`

To

`local   all             postgres                                trust`

Then restart postgres:

`sudo service postgresql restart`

Getting started with geodjango:

Now we are ready to get started:

`django-admin startproject geodjango`

`cd geodjango`

`python manage.py startapp world`

Now we'll go into the settings.py file:

`emacs geodjango/settings.py`

and edit the databases connection to look like this:

```
DATABASES = {
    'default': {
         'ENGINE': 'django.contrib.gis.db.backends.postgis',
         'NAME': 'geodjango',
         'USER': 'geo',
     }
} 
```

Notice that we haven't created the 'geodjango'-database so we'll do that now:

`sudo -u postgres createuser -P geo`

`sudo -u postgres createdb -O geo geodjango`

`sudo -u postgres psql -c "CREATE EXTENSION postgis; CREATE EXTENSION postgis_topology;" geodjango`

we'll also need to edit the installed aps, in the same file:

```
INSTALLED_APPS = (
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'django.contrib.gis',
    'world'
)
```

Great, now we can save and close that.

Next we'll need some data to visualize:

`mkdir world/data`

`cd world/data`

`wget http://thematicmapping.org/downloads/TM_WORLD_BORDERS-0.3.zip`

`unzip TM_WORLD_BORDERS-0.3.zip`

`cd ../..`

Now let's inspect our data so we now how our model should look - we should try to be consistent with how the data is annotated for portability and extensibility.  

For this we'll need gdal - `sudo apt-get install libgdal-dev python-gdal gdal-bin # the python library is unnecessary but nice to have :)`

Now we can inspect the annotation in the shapefile of our geospatial data:

`ogrinfo -so world/data/TM_WORLD_BORDERS-0.3.shp TM_WORLD_BORDERS-0.3`

We'll use this output to map to our models.py file:

`emacs world/models.py` and type:

```
from django.contrib.gis.db import models

class WorldBorder(models.Model):
    # Regular Django fields corresponding to the attributes in the
    # world borders shapefile.
    name = models.CharField(max_length=50)
    area = models.IntegerField()
    pop2005 = models.IntegerField('Population 2005')
    fips = models.CharField('FIPS Code', max_length=2)
    iso2 = models.CharField('2 Digit ISO', max_length=2)
    iso3 = models.CharField('3 Digit ISO', max_length=3)
    un = models.IntegerField('United Nations Code')
    region = models.IntegerField('Region Code')
    subregion = models.IntegerField('Sub-Region Code')
    lon = models.FloatField()
    lat = models.FloatField()

    # GeoDjango-specific: a geometry field (MultiPolygonField), and
    # overriding the default manager with a GeoManager instance.
    mpoly = models.MultiPolygonField()
    objects = models.GeoManager()

    # Returns the string representation of the model.
    def __str__(self):              # __unicode__ on Python 2
        return self.name
```

We are now ready to run our first migration :)

`python manage.py makemigrations`


`python manage.py sqlmigreate world 0001`


`python manage.py migrate`

Making our first map:

Now that we've done all the setup, we can leverage geodjango immmediately to create an interactive map!

We'll need to make a few small edits to some files, but essetentially, we are done:

open the admin.py file and type the following:

emacs world/admin.py:

```
from django.contrib.gis import admin
from models import WorldBorder

admin.site.register(WorldBorder, admin.GeoModelAdmin)
```

Finally we simply need create our admin credentials:

`python manage.py createsuperuser`
`python manage.py runserver`

Now head over to [http://127.0.0.1:8000/admin/](http://127.0.0.1:8000/admin/)

and enter your credentials.  Now you have a GIS system that you can play with, load data into, and get analysis out of!

I found the setup for GeoDjango to be slightly outdated and confusing and so I thought it was important to spend time on it instead of showing you examples of how to use GeoDjango, for examples of this I recommend:

GeoDjango Specific tutorials and examples:

Basic:

http://blog.mathieu-leplatre.info/geodjango-maps-with-leaflet.html

http://invisibleroads.com/tutorials/geodjango-googlemaps-build.html

http://davidwilson.me/2013/09/30/Colorado-Geology-GeoDjango-Tutorial/


Advanced:

http://blog.apps.chicagotribune.com/category/data-visualization/

Visualizing Crime data with GIS:

http://flowingdata.com/2009/06/23/20-visualizations-to-understand-crime/

Where to get your own data:

https://data.cityofchicago.org/

https://nycopendata.socrata.com/

How to do Geo-Python things with other tools:

https://2015.foss4g-na.org/sites/default/files/slides/Installation%20Guide_%20Spatial%20Data%20Analysis%20in%20Python.pdf

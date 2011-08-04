Django Jalali
=============
This module gives you a DateField same as Django's DateField but you can get and query data based on Jalali Date

Dependencies
------------
* [jdatetime](http://pypi.python.org/pypi/jdatetime/) -- `easy_install jdatetime`

Install
-------

    easy_install django_jalali

Usage
-----

### Direct Usage

1. Run : 


    $ django-admin.py startproject jalali_test


2. Start your app :

    $ python manage.py startapp foo

3. Edit settings.py and add django_jalali and your foo to your INSTALLED_APPS (also config DATABASES setting)

4. Edit foo/models.py 


    from django.db import models                                                                                                                          
    from django_jalali.db import models as jmodels

    class Bar(models.Model):
        objects = jmodels.jManager()
        name =  models.CharField(max_length=200)
        date =  jmodels.jDateField()
        def __str__(self):
            return "%s, %s"%(self.name, self.date)
    class BarTime(models.Model):
        objects = jmodels.jManager()
        name =  models.CharField(max_length=200)
        datetime = jmodels.jDateTimeField()
        def __str__(self):
            return "%s, %s" %(self.name, self.datetime)


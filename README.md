BLAH BLAH
=============
This module gives you a BLAH as BLAH's BLAH but you can get and query data based on BLAH BLAH

Dependencies
------------
* [BLAH](http://BLAH.BLAH.org/BLAH/BLAH/) -- `BLAH BLAH`

Install
-------

    BLAH BLAH

Usage
-----

### Direct Usage

 

```python
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
```

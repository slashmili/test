BLAH BLAH
=============
This module gives you a BLAH as BLAH's BLAH but you can get and query data based on BLAH BLAH


Links
------

[Local Link to rails.rdoc](/rails.rdoc) 
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

<div dir="rtl">

تست فارسی
=============
این یک تست برای نوشتن فارسی در زبان Markdown می‌باشد.

تست
-------
* [تست لینک](http://BLAH.BLAH.org/BLAH/BLAH/) --`یک تست می باشد`

نصب
-------

    برای نصب این امکان نیاز به تنظیمات خاصی ندارید

استفاده
-----

### استفاده مستقیم
<div dir="ltr">
```python
    from markdown import rtl                                                                                                                          
```
<div dir="rtl">
برای دیدن منبع برنامه بر روی سورس در بالای صفحه کلیک کنید

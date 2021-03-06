[![Build Status](https://travis-ci.org/trobz/openerp-booking-chart.png?branch=master)](https://travis-ci.org/trobz/openerp-booking-chart)


## Introduction

The Booking Chart View provide an overview in a calendar for any OpenERP resources, 
you can have different type of resource booking in your calendar and you can freely associate them to any resources listed in rows.

[Video Demo](http://trobz.github.io/openerp-booking-chart/ "OpenERP Booking Chart Presentation Page")

### Features

- Display any resources in a booking chart
- Easy scrolling navigation
- Resource lazy loading
- Zoom levels (from 1 week to 6 months)
- Auto-merge overlapping resources
- Native OpenERP search/group supported on listed resources
- Manual period selection
- Period freeze/unfreeze button
- Mixin to link any OpenERP model with resources booking


## Key concepts

The booking chart is decoupled from OpenERP models, by this way you are totally free to display any resources in your chart, 
the counterpart is that you have to code the link between OpenERP models and your booking chart.

We decide to go on this direction instead of having an OpenERP gantt-like configuration in `arch` tag because our module need 
to link booking chart to multiple models, with different logics, and it's hard to achieve this with just some configuration.


2 main models are used in the booking chart:

### booking_chart

```
- name
- resource_model        // ref to the model used to build the chart, used to list resources, 
						// search/group_by queries are applied on this model
- resource_domain       // force some additional domains, added when resource_model are retrieved
- supported_model_ids   // list of model supported by resources booking  
						// for origin and/or target (required for fields.reference selection)
```

A booking chart define from which model the chart will be created. 
Resources will be displayed as row in the chart and Resource Booking have to make a reference 
to specific object from this model to be correctly displayed.

### booking_resource

```
- name
- chart_id      // relation with a booking_chart object
- resource_ref  // reference to an object from the booking_chart.resource_model
- origin_ref    // reference to the object at the origin of the resource booking 
- target_ref    // reference to the object to open when the resource booking is 
                // clicked (not required)
- date_start
- date_end
- css_class
- message
- tags
```


A resource booking is the periodical element displayed in the chart, the object at the origin of 
the resource booking can be any OpenERP model (origin_ref).
If a target object is defined, it will be displayed in a form view at click on the resource booking. 

## Setup / Use

The module itself is installable from the OpenERP module interface.

However, the booking chart doesn't work out of the box, because of the flexibility to link any models, you will have to
add some code to create/update/delete resources booking.

A mixin helper is available but you can implement your own logic to keep your resources booking up to date with an other model.


### `booking.resource` and `booking.chart` views

Views are available to directly edit booking models, these views are only accessible 
to users with the "Technical Features" enabled.

Access to booking chart model views:           
`Settings > Technical > Booking Chart`  

## Configuration

The booking chart support some `arch` view configuration:

- tag `<items>`
  - attribute `title`: field from `booking_chart.resource_model` to display in the booking list

**example:**

```xml
<record id="bar_booking_view" model="ir.ui.view">
    ...
    <field name="arch" type="xml">
        <booking version="7.0">
            <items title="name" />
        </booking>
    </field>
</record>
```

### Model Mixin

To simplify this task, mixin model is available in `booking_chart.mixin`, 
this mixin is used by the `demo_task` module.


Basically, only a mapping between your model and the booking.resource has to be done, 
with different type of relation:

- simple mapping: value is just copied
- reference mapping: the value is defined according to a `osv.model` object
- custom mapping: define the field to get when an other field has been modified. useful to define mapping on function fields, see the example below.


The mixin is designed to automatically create, update and delete resources booking associated with the model.

**example from `demo_task` module**

```python
from openerp.osv import fields
from booking_chart.mixin import mixin
    
class task(mixin.resource):
    _inherit = "project.task"
    
    # ref to the booking_chart xml_id 
    #
    # you can change the way to get the booking_chart by 
    # overriding get_chart_id() method (more details below)
    #
    _booking_chart_ref = 'demo_task.users_booking_chart'
    
    _booking_resource_map = {
        # simple mapping, booking.resource field = task field 
        'name':        'name',
        'message':     'description',
        'date_start':  'date_start',
        'date_end':    'date_end',
        # reference mapping, booking.resource field = "task.field._name,task.field.id" 
        'resource_ref': 'user_id',
        'target_ref':   'project_id',
        # custom mapping, set booking.resource.css_class field when priority is 
        # updated with the value of task.booking_css_class
        'css_class':   'priority:booking_css_class'
    }
    
    
    def _get_booking_custom_fields(self, cr, uid, ids, field_names, arg, context=None):
        # resource booking color mapping with task.priority
        colors = {
            '0': 'red', '1': 'orange', '2': 'dark-blue',  '3': 'blue', '4': 'light-blue'
        }
        res = {}
        
        for task in self.browse(cr, uid, ids):
            res[task.id] = colors[task.priority] if task.priority in colors else ""
        
        return res
    
    # add a custom field to get the booking class css according to current status
    _columns = {
        'booking_css_class': fields.function(_get_booking_custom_fields, 
        									 method=True, 
        									 type='char', 
        									 string='Booking CSS Class', 
        									 readonly=True),
    }
       
task()
```

#### Customize the booking chart associated with the resource booking 

If you need a specific way to get the chart associated with the resource booking, you can override `get_chart_id()`.

This method has to return the booking.chart id, and in addition to `cr` and `uid`  parameters, the method get the current model.

** example **

```python
from booking_chart.mixin import mixin
    
class task(mixin.resource):
    _inherit = "project.task"
	
	...
	
	def get_chart_id(self, cr, uid, model):
		xml_id = 'users_booking_chart'
		
		if model.priority > 2:
			xml_id = 'priority_tasks_booking_chart'
		
		model_data = self.pool.get('ir.model.data')
		
		ref = model_data.get_object_reference(cr, uid, 'demo_task', xml_id)
        
        if len(ref) < 2:
            raise Exception('invalid xml_id: %s' % (xml_id))
        
        return ref[1] 
	
	...

task()
```

## Dependencies

- [Web Unleashed module](https://github.com/trobz/openerp-web-unleashed "OpenERP Web Unleashed")     
This module provide native support of Backbone and Marionette, simplifing dramatically the creation of rich views in OpenERP.  

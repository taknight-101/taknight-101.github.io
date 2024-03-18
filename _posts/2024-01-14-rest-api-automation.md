---
layout: post
title: "Servicenow REST-API based Automation"
permalink: "restapi-automation"
categories: [Platform Engineering, ServiceNow]
tags:  ["automation", "python", "hacks", "developer_productivity"]  # TAG names should always be lowercase
# image: /images/hacks/hack1/hack1.jpg


---



>  In this short blog, we explain how it's possible to automate typical servicenow workflows using the python [pysnow](https://pysnow.readthedocs.io/en/latest/index.html) module



![]({{ site.baseurl }}/images/hacks/hack1/h1.png)





* **Following is the python script used to implement the above workflow**
 
```python

import pysnow


# Create client object
c = pysnow.Client(instance='#pdi_name', user='admin', password='pdi_password')



def get_sys_id(table  = "sys_user", field_name = "user_name" , field_value = "admin"):        
    qb = (
        pysnow.QueryBuilder()
        .field(field_name).equals(field_value)

    )

    callers = c.resource(api_path=f'/table/{table}')

    response = callers.get(query=qb)
    return response.all()[0]['sys_id']



def create_incident(caller_id): 

    # Define a resource, here we'll use the incident table API
    incident = c.resource(api_path='/table/incident')

    # Set the payload
    new_record = {
        'caller_id' : caller_id , 
        'short_description': 'Fix Your Bugs',
        'description': 'Yes, you.'
    }

    # Create a new incident record
    result = incident.create(payload=new_record)
    return result["sys_id"]


def resolve_incident(incident_sys_id , resolution_code =  'Known error' , resolution_note = 'I fixed it :)' ):
  
    # Define a resource, here we'll use the incident table API
    incident = c.resource(api_path='/table/incident')

    update = {'close_code': resolution_code ,'close_notes':resolution_note  ,  'state': 6}

    # Update 'close_code', 'close_notes', and 'state' for input incident sys_id
    updated_record = incident.update(query={'sys_id': incident_sys_id }, payload=update)



resolve_incident(create_incident(get_sys_id()))

```

* **Screenshot of the created incident after resolution**




![]({{ site.baseurl }}/images/hacks/hack1/hack1.1.png)






-----------------
> Possible Next steps 


* Implement a data pipeline for more complex workflows using something like: [luigi](https://luigi.readthedocs.io/en/stable/index.html) 

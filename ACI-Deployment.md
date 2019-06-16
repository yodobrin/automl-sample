
## Using model & or Image

### Prerequisites & Initialize Workspace

* You need to use Python 3.6 - AzureML kernel or one that supports the same dependencies
* Each workspace is initilized with a configuration file named config.json, this file represent the details of the workspace and can be obtained from the workspace itself, it needs to be in the path to be picked up.
* This sample notebook does not specifiy exactly the name of the model or the image to be used - make sure to keep track on the one you use.
* Per experiment, a scoring file and conda env yaml file are avilable from the portal, note that during testing or validation of the exposed web-service you would need to pass the exact params in the exact order (if the auto generated file is used).

* The experiment starts with the portal

![image.png](https://user-images.githubusercontent.com/37622785/59563022-3c43ec80-903d-11e9-8877-a8e8257697b6.png)


```python
import azureml.core

print("SDK version:", azureml.core.VERSION)
```

### Assuming the config file exist, it would load it, note that login would be required


```python
from azureml.core import Workspace

ws = Workspace.from_config()
print(ws.name, ws.resource_group, ws.location, ws.subscription_id, sep = '\n')
```

### Load the model
list exsiting models

![load](https://user-images.githubusercontent.com/37622785/59563020-35b57500-903d-11e9-8e01-4fca79883999.png)


```python
from azureml.core.model import Model
models = Model.list(workspace=ws)
for m in models:
    print("Name:", m.name,"\tVersion:", m.version, "\tDescription:", m.description, m.tags)
```


```python
the_model = Model.list(workspace=ws, name='<model-name>')
```

### Creating an Image

![create](https://user-images.githubusercontent.com/37622785/59563019-2b937680-903d-11e9-9e11-1bc22f3f524a.png)


```python
from azureml.core.image import Image, ContainerImage
image_config = ContainerImage.image_configuration(runtime= "python",
                                 execution_script="scoring.py",
                                 conda_file="condaEnv.yml",
                                 tags = {'area': "<area>", 'type': "<classification>"},
                                 description = "<some descriptive info>")

image = Image.create(name = "<image-name>",
                     # this is the model object. note you can pass in 0-n models via this list-type parameter
                     # in case you need to reference multiple models, or none at all, in your scoring script.
                     models = the_model,
                     image_config = image_config, 
                     workspace = ws)
```


```python
image.wait_for_creation(show_output = True)
```


```python
from azureml.core.image import Image, ContainerImage
image  = Image(ws, name='<pre-created-image-name>')
```

### Configure WevService deployment


```python
from azureml.core.webservice import AciWebservice

aciconfig = AciWebservice.deploy_configuration(cpu_cores = 1, 
                                               memory_gb = 1, 
                                               tags = {'area': "<area-description>", 'type': "<classification>"}, 
                                               description = '<your description>')
```

### Deploy the webservice image to ACI

![deploy](https://user-images.githubusercontent.com/37622785/59563016-22a2a500-903d-11e9-81bb-6e381929ce24.png)


```python
from azureml.core.webservice import Webservice

aci_service_name = '<web-service-name>'
print(aci_service_name)
aci_service = Webservice.deploy_from_image(deployment_config = aciconfig,
                                           image = image,
                                           name = aci_service_name,
                                           workspace = ws)
aci_service.wait_for_deployment(True)
print(aci_service.state)
```


```python
print(aci_service.scoring_uri)
```

### Test Data
4 examples which were not part of the training

No 56,'Non-Travel',667,'Research & Development',1,4,'Life Sciences',1,2026,3,'Male',57,3,2,'Healthcare Representative',3,'Divorced',6306,26236,1,'Y','No',21,4,1,80,1,13,2,2,13,12,1,9

Yes 29,'Travel_Rarely',1092,'Research & Development',1,4,'Medical',1,2027,1,'Male',36,3,1,'Research Scientist',4,'Married',4787,26124,9,'Y','Yes',14,3,2,80,3,4,3,4,2,2,2,2

No 42,'Travel_Rarely',300,'Research & Development',2,3,'Life Sciences',1,2031,1,'Male',56,3,5,'Manager',3,'Married',18880,17312,5,'Y','No',11,3,1,80,0,24,2,2,22,6,4,14

Yes 56,'Travel_Rarely',310,'Research & Development',7,2,'Technical Degree',1,2032,4,'Male',72,3,1,'Laboratory Technician',3,'Married',2339,3666,8,'Y','No',11,3,4,80,1,14,4,1,10,9,9,8


### Test web service
Call the web service with some dummy input data to get a prediction.


```python
import json
#
# 
test_sample = json.dumps({'data': [
    [56,'Travel_Rarely',310,'Research & Development',7,2,'Technical Degree',1,2032,4,'Male',72,3,1,'Laboratory Technician',3,'Married',2339,3666,8,'Y','No',11,3,4,80,1,14,4,1,10,9,9,8]
]})
test_sample = bytes(test_sample,encoding = 'utf8')

prediction = aci_service.run(input_data=test_sample)
print(prediction)
```

### Delete ACI to clean up


```python
aci_service.delete()
```



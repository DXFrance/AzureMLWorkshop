# Retrain and republish an Azure ML Studio model from Python code


```python
# please update with your values

## default endpoint
myurl = 'https://europ##obfuscated##rue'
mykey = 'tT##obfuscated##5w=='

## prod endpoint
myurl_prod = 'https://##obfuscated##ails=true'
mykey_prod = 'N##obfuscated##='

## patch prod endpoint
myurl_patch_prod = 'https://eu##obfuscated##endpoints/prod'
mykey_patch_prod = mykey_prod

## retrain web service
myurl_retrain = 'https://##obfuscated##311c07/jobs'
mykey_retrain = 'o3##obfuscated##deA=='

## the 2 following values will be available at https://aka.ms/notes170301 during the workshop
mystoragename = 'st##obfuscated##we'
mystoragekey = 'j##obfuscated##Q=='

mystoragecontainer = 'retrain'
mystorageinputdatafile1 = 'immo.csv'
mystorageinputdatafile2 = 'immo-plus-10-percent.csv'
mystorageoutputlearnerfile = 'output2results.ilearner'
mystorageoutputdatafile = 'output1results.csv'

mytrainedmodelname = 'immo simple 170301 (starting experiment) [trained model]'
```


```python
# initializing other variables
LatestOutputName = ''
LatestBaseLocation = ''
LatestRelativeLocation = ''
LatestSasBlobToken = ''
```

## Test the Web Service (default)


```python
import urllib2
# If you are using Python 3+, import urllib instead of urllib2

import json 


data =  {

        "Inputs": {

                "input1":
                {
                    "ColumnNames": ["surface"],
                    "Values": [ [ "100" ], [ "150" ], ]
                },        },
            "GlobalParameters": {
}
    }

body = str.encode(json.dumps(data))

url = myurl
api_key = mykey
headers = {'Content-Type':'application/json', 'Authorization':('Bearer '+ api_key)}

req = urllib2.Request(url, body, headers) 

try:
    response = urllib2.urlopen(req)

    # If you are using Python 3+, replace urllib2 with urllib.request in the above code:
    # req = urllib.request.Request(url, body, headers) 
    # response = urllib.request.urlopen(req)

    result = response.read()
    print(result) 
except urllib2.HTTPError, error:
    print("The request failed with status code: " + str(error.code))

    # Print the headers - they include the requert ID and the timestamp, which are useful for debugging the failure
    print(error.info())

    print(json.loads(error.read()))
```

    {"Results":{"output1":{"type":"table","value":{
        "ColumnNames":["surface","Scored Labels"],
        "ColumnTypes":["Int32","Double"],
        "Values":[["100","100749.893789066"],["150","147449.944996009"]]}}}}


## Add an endpoint

Go to <https://services.azureml.net/classicWebservices/> and create a new endpoint called `prod`

Then, update the value of `myurl_prod` and `mykey_prod`

## Test the Web Service (prod)


```python
import urllib2
# If you are using Python 3+, import urllib instead of urllib2

import json 


data =  {

        "Inputs": {

                "input1":
                {
                    "ColumnNames": ["surface"],
                    "Values": [ [ "100" ], [ "150" ], ]
                },        },
            "GlobalParameters": {
}
    }

body = str.encode(json.dumps(data))

url = myurl_prod
api_key = mykey_prod
headers = {'Content-Type':'application/json', 'Authorization':('Bearer '+ api_key)}

req = urllib2.Request(url, body, headers) 

try:
    response = urllib2.urlopen(req)

    # If you are using Python 3+, replace urllib2 with urllib.request in the above code:
    # req = urllib.request.Request(url, body, headers) 
    # response = urllib.request.urlopen(req)

    result = response.read()
    print(result) 
except urllib2.HTTPError, error:
    print("The request failed with status code: " + str(error.code))

    # Print the headers - they include the requert ID and the timestamp, which are useful for debugging the failure
    print(error.info())

    print(json.loads(error.read()))
```

    {"Results":{"output1":{"type":"table","value":{
        "ColumnNames":["surface","Scored Labels"],
        "ColumnTypes":["Int32","Double"],
        "Values":[["100","100749.893789066"],["150","147449.944996009"]]}}}}


## Create a retrain web service

Add a web service input and 2 web service outputs (output1 for the evaluation of the model, output2 for the retrained model itself), the deploy the web service. This will provide you with the values of myurl_retrain and mykey_retrain.

## Retrain with new dataset

Upload the new dataset CSV file to the blob storage account. 
- The name must correspond to the `mystorageinputdatafile` variable.
- It must be in the container corresponding to `mycontainer` variable.
- It must be in the storage account corresponding to `mystorageaccount` variable.

To do this, the [Microsoft Azure Storage Explorer](http://storageexplorer.com) tool can be used from Windows, Mac or Linux.

A preconfigured storage account can also be provided to you.



```python
# How this works:
#
# 1. Assume the input is present in a local file (if the web service accepts input)
# 2. Upload the file to an Azure blob - you"d need an Azure storage account
# 3. Call BES to process the data in the blob. 
# 4. The results get written to another Azure blob.
#
# Note: You may need to download/install the Azure SDK for Python.
# See: http://azure.microsoft.com/en-us/documentation/articles/python-how-to-install/

import urllib2
# If you are using Python 3+, import urllib instead of urllib2

import json
import time
from azure.storage.blob import *

def printHttpError(httpError):
    print("The request failed with status code: " + str(httpError.code))

    # Print the headers - they include the requert ID and the timestamp, which are useful for debugging the failure
    print(httpError.info())

    print(json.loads(httpError.read()))
    return


def processResults(result):

    global LatestBaseLocation
    global LatestRelativeLocation
    global LatestSasBlobToken
    results = result["Results"]
    for outputName in results:
        result_blob_location = results[outputName]
        sas_token = result_blob_location["SasBlobToken"]
        base_url = result_blob_location["BaseLocation"]
        relative_url = result_blob_location["RelativeLocation"]

        print("The results for " + outputName + " are available at the following Azure Storage location:")
        print("BaseLocation: " + base_url)
        print("RelativeLocation: " + relative_url)
        print("SasBlobToken: " + sas_token)

        if outputName == 'output2':
            LatestBaseLocation = base_url
            LatestRelativeLocation = relative_url
            LatestSasBlobToken = sas_token
    return



def uploadFileToBlob(input_file, input_blob_name, storage_container_name, storage_account_name, storage_account_key):
    blob_service = BlobService(account_name=storage_account_name, account_key=storage_account_key)

    print("Uploading the input to blob storage...")
    data_to_upload = open(input_file, "r").read()
    blob_service.put_blob(storage_container_name, input_blob_name, data_to_upload, x_ms_blob_type="BlockBlob")

def invokeBatchExecutionService():

    storage_account_name = mystoragename
    storage_account_key = mystoragekey
    storage_container_name = mystoragecontainer
    connection_string = "DefaultEndpointsProtocol=https;AccountName=" + storage_account_name + ";AccountKey=" + storage_account_key
    api_key = mykey_retrain
    url = myurl_retrain


    # Skipping this step as we run from a notebook, not locally
    #uploadFileToBlob("input1data.csv", # Replace this with the location of your input file
    #                 "input1datablob.csv", # Replace this with the name you would like to use for your Azure blob; this needs to have the same extension as the input file 
    #                 storage_container_name, storage_account_name, storage_account_key)

    payload =  {

        "Inputs": {

            "input1": { "ConnectionString": connection_string, "RelativeLocation": "/" + 
                       storage_container_name + "/" + mystorageinputdatafile2 },
        },     

        "Outputs": {

            "output2": { "ConnectionString": connection_string, "RelativeLocation": "/" + 
                        storage_container_name + "/" + mystorageoutputlearnerfile },

            "output1": { "ConnectionString": connection_string, "RelativeLocation": "/" + 
                        storage_container_name + "/" + mystorageoutputdatafile },
        },
        "GlobalParameters": {
}
    }

    body = str.encode(json.dumps(payload))
    headers = { "Content-Type":"application/json", "Authorization":("Bearer " + api_key)}
    print("Submitting the job...")

    # If you are using Python 3+, replace urllib2 with urllib.request in the following code

    # submit the job
    req = urllib2.Request(url + "?api-version=2.0", body, headers)
    try:
        response = urllib2.urlopen(req)
    except urllib2.HTTPError, error:
        printHttpError(error)
        return

    result = response.read()
    job_id = result[1:-1] # remove the enclosing double-quotes
    print("Job ID: " + job_id)


    # If you are using Python 3+, replace urllib2 with urllib.request in the following code
    # start the job
    print("Starting the job...")
    req = urllib2.Request(url + "/" + job_id + "/start?api-version=2.0", "", headers)
    try:
        response = urllib2.urlopen(req)
    except urllib2.HTTPError, error:
        printHttpError(error)
        return

    url2 = url + "/" + job_id + "?api-version=2.0"

    while True:
        print("Checking the job status...")
        # If you are using Python 3+, replace urllib2 with urllib.request in the follwing code
        req = urllib2.Request(url2, headers = { "Authorization":("Bearer " + api_key) })

        try:
            response = urllib2.urlopen(req)
        except urllib2.HTTPError, error:
            printHttpError(error)
            return    

        result = json.loads(response.read())
        status = result["StatusCode"]
        if (status == 0 or status == "NotStarted"):
            print("Job " + job_id + " not yet started...")
        elif (status == 1 or status == "Running"):
            print("Job " + job_id + " running...")
        elif (status == 2 or status == "Failed"):
            print("Job " + job_id + " failed!")
            print("Error details: " + result["Details"])
            break
        elif (status == 3 or status == "Cancelled"):
            print("Job " + job_id + " cancelled!")
            break
        elif (status == 4 or status == "Finished"):
            print("Job " + job_id + " finished!")
        
            processResults(result)
            break
        time.sleep(5) # wait a few seconds
    return

invokeBatchExecutionService()
```

    Submitting the job...
    Job ID: 86da71e06f3945a9b770d83181f2ed68
    Starting the job...
    Checking the job status...
    Job 86da71e06f3945a9b770d83181f2ed68 not yet started...
    Checking the job status...
    Job 86da71e06f3945a9b770d83181f2ed68 running...
    Checking the job status...
    Job 86da71e06f3945a9b770d83181f2ed68 running...
    Checking the job status...
    Job 86da71e06f3945a9b770d83181f2ed68 running...
    Checking the job status...
    Job 86da71e06f3945a9b770d83181f2ed68 running...
    Checking the job status...
    Job 86da71e06f3945a9b770d83181f2ed68 running...
    Checking the job status...
    Job 86da71e06f3945a9b770d83181f2ed68 running...
    Checking the job status...
    Job 86da71e06f3945a9b770d83181f2ed68 running...
    Checking the job status...
    Job 86da71e06f3945a9b770d83181f2ed68 running...
    Checking the job status...
    Job 86da71e06f3945a9b770d83181f2ed68 running...
    Checking the job status...
    Job 86da71e06f3945a9b770d83181f2ed68 finished!
    The results for output2 are available at the following Azure Storage location:
    BaseLocation: https://storage170301we.blob.core.windows.net/
    RelativeLocation: retrain/output2results.ilearner
    SasBlobToken: ?sv=2015-02-21&sr=b&sig=qsq##obfuscated##st=2017-03-01T10%3A01%3A12Z&se=2017-03-02T10%3A06%3A12Z&sp=r
    The results for output1 are available at the following Azure Storage location:
    BaseLocation: https://storage170301we.blob.core.windows.net/
    RelativeLocation: retrain/output1results.csv
    SasBlobToken: ?sv=2015-02-21&sr=b&sig=cw##obfuscated##&st=2017-03-01T10%3A01%3A12Z&se=2017-03-02T10%3A06%3A12Z&sp=r


## Check the result

check the blob corresponding to `mystorageaccountname`/`blob.core.windows.net`/`mystoragecontainer`/`mystorageoutputdatafile`

here is a sample output: 

```
Negative Log Likelihood,Mean Absolute Error,Root Mean Squared Error,Relative Absolute Error,Relative Squared Error,Coefficient of Determination
Infinity,0.0912634519229332,0.107777663489987,1.23406681461624E-06,1.66834112858664E-12,0.999999999998332
761.748085547315,2269.32795754446,2379.65491644217,0.0306859127600318,0.000813308466497854,0.999186691533502
```

## Patch the prod endpoint


```python
print(LatestBaseLocation)
print(LatestRelativeLocation)
print(LatestSasBlobToken)
```

    https://storage170301we.blob.core.windows.net/
    retrain/output2results.ilearner
    ?sv=2015-02-21&sr=b&sig=qs##obfuscated##3D&st=2017-03-01T10%3A01%3A12Z&se=2017-03-02T10%3A06%3A12Z&sp=r



```python
import urllib2
# If you are using Python 3+, import urllib instead of urllib2

import json 

data =  {
            "Resources": [
                {
                    "Name": mytrainedmodelname,
                    "Location": 
                    {
                            # Replace these values with the ones that specify the location of the new value for this resource. For instance,
                            # if this resource is a trained model, you must first execute the training web service, using the Batch Execution API,
                            # to generate the new trained model. The location of the new trained model would be returned as the "Result" object
                            # in the response. 
                            "BaseLocation": LatestBaseLocation,
                            "RelativeLocation": LatestRelativeLocation,
                            "SasBlobToken": LatestSasBlobToken
                    }
                }
            ]
        }

body = str.encode(json.dumps(data))

url = myurl_patch_prod
api_key = mykey_patch_prod
headers = {'Content-Type':'application/json', 'Authorization':('Bearer '+ api_key)}

req = urllib2.Request(url, body, headers)
req.get_method = lambda: 'PATCH'
response = urllib2.urlopen(req)

# If you are using Python 3+, replace urllib2 with urllib.request in the above code:
# req = urllib.request.Request(url, body, headers) 
# response = urllib.request.urlopen(req)

result = response.read()
print(result) 
```

    


## Test the web service after its update (prod)


```python
import urllib2
# If you are using Python 3+, import urllib instead of urllib2

import json 


data =  {

        "Inputs": {

                "input1":
                {
                    "ColumnNames": ["surface"],
                    "Values": [ [ "100" ], [ "150" ], ]
                },        },
            "GlobalParameters": {
}
    }

body = str.encode(json.dumps(data))

url = myurl_prod
api_key = mykey_prod
headers = {'Content-Type':'application/json', 'Authorization':('Bearer '+ api_key)}

req = urllib2.Request(url, body, headers) 

try:
    response = urllib2.urlopen(req)

    # If you are using Python 3+, replace urllib2 with urllib.request in the above code:
    # req = urllib.request.Request(url, body, headers) 
    # response = urllib.request.urlopen(req)

    result = response.read()
    print(result) 
except urllib2.HTTPError, error:
    print("The request failed with status code: " + str(error.code))

    # Print the headers - they include the requert ID and the timestamp, which are useful for debugging the failure
    print(error.info())

    print(json.loads(error.read()))
```

    {"Results":{"output1":{"type":"table","value":{
        "ColumnNames":["surface","Scored Labels"],
        "ColumnTypes":["Int32","Double"],
        "Values":[["100","110824.883167972"],["150","162194.93949561"]]}}}}



```python

```

### Getting Started with DICOMAnon Scripting
To interact with the API you must have an API node running.
From the API node, get the IP address, port, and API key. The API key is a secret key that is used to authenticate.
Create a Console application and reference the NuGet package `DICOMAnonAPI`.

#### The Client
The base object to work with is a client object. 
```cs
var ipAddress = "192.168.1.1";
var port = 13999;
var apiKey = "kdcdknentpspienfn23dfdns";
var client = new DAClient(ipAddress, port, apiKey);
```

#### Interacting with the Worklist
Before you start working with the worklist, you might want to clear it to prevent
any old data from being operated on.

##### Clearing the Worklist
```cs
var successfulClear = client.ClearWorklist();
```

##### Adding a new Job to the Worklist
The basic object is a WorklistItem. A WorklistItem contains the following properties:
- `JobId` - The unique identifier for the job
- `PatientId` - The patient ID
- `MapToId` (optional) - The new ID to map to (the new patient ID)
- `MapToName` (optional) - The new name to map to (the new patient name)`

A WorklistItem contains a list of WorklistInstances. A WorklistInstance represents a single instance of a DICOM file,
series or study. To construct a WorklistInstance, you need to provide the following properties:
```cs
var wl = new WorklistItem();
wl.PatientId = "12345";
//Add a new instance to the worklist
wl.Instances.Add(new WorklistInstance()
                    {
          //A single DICOM instance (plan, dose, structure set, etc.)
                        SOPInstanceUID = "1.2.1.15589.....",
                        SeriesInstanceUID = "1.2.1.121354.....""
                    });
//Add a new series to the worklist
wl.Instances.Add(new WorklistInstance()
					{
			//A DICOM series (useful for images with multiple instances)
						SeriesInstanceUID = "1.2.1.121354.....""
					});
```
Then add the worklist item to the worklist
```cs
//DICOMAnon will return a unique job ID for the worklist item
var jobId = _client.AddToWorklist(wlItem);
```
##### Modifying the Settings for the Worklist Jobs
You can modify the settings for the worklist jobs. The API is interacting with the DICOMAnon user interface,
so modifying the settings will affect the behavior other users potentially. To keep the settings, store the 
original settings and restore them after the job is done.
```cs
var settings = client.GetSettings();
//Reset anonymization settings
settings.AnonymizationSettings = new AnonymizationSettings();
//Disable the modifications
settings.AnonymizationSettings.IsModificationEnabled = false;
//Clear any post processors that could have been added
settings.PostProcessors.Clear();
//Add a post processor to inject custom Json object
settings.AddOrUpdatePostProcessor(new DAAPI.Models.PostProcessors.InjectCustomElement()
{
    IsEnabled = true,
    Name = "Inject JSON Object",
    Settings = new DAAPI.Models.PostProcessors.CustomElementSettings()
    {
        GroupId = 27,
        ElementId = 27,
        VR = DAAPI.Models.PostProcessors.VROption.UT,
        Data = JsonConvert.SerializeObject(new
        {
            //Any object that can be serialized to JSON
        })
    }
});
//Add a post processor to send to another DICOM node
settings.AddOrUpdatePostProcessor(new DAAPI.Models.PostProcessors.SendToDICOMProcess()
{
    IsEnabled = true,
    Name = "Back to VMS",
    Settings = new DAAPI.Models.PostProcessors.DICOMDestinationSettings()
    {
        AeTitle = "VMS_AE",
        Port = 51402,
        IpAddress = "192.168.5.5",
        SendingAETitle = "DICOMAnon",
    }
});
client.UpdateSettings(settings);
```
##### Starting Operations on the Worklist
```cs
var startedRun = client.Run();
```
##### Getting the Status of Jobs
You can get the status of jobs by calling the `GetStatus` method. The method returns a `WorklistItemProgress` object.
```cs
var status = client.GetItemStatus(job);
//Job progress 0-100
var progress = status.Progress;
//Job status
var wlStatus = status.Status;
if(wlStatus != WorklistStatus.COMPLETE)
{
    //Wait - check again later
}
```

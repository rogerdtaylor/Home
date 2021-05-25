## P2 - PRB0044179 - Support UI Function to Import Master Profiles behaves oddly and takes a long time to complete.

Whate needs to be done
1) need to update

<br>
<br>
## `ORIGINAL TEXT`

See https://microlise.service-now.com/problem.do?sys\_id=2cf5d6401bcc7010693eb165464bcbd4&sysparm\_record\_target=problem&sysparm\_record\_row=2&sysparm\_record\_rows=2&sysparm\_record\_list=u\_development\_domain%3D54922250db466f80456972fabf961951%5Eu\_release\_planned\_for%3D21.6%5Eu\_type%3Dci%5Eassigned\_to%3D38541ed6db6d7b0008e1fc75ae961977%5EORDERBYnumber%5EORDERBYsys\_id

The slow parts of the SupportUI which take longer than 5 minutes, and cause the F5 to retry the requests are:

```
AmberSupportUI/ImportData							
"Import" then "Update Database"	
ImportDataController.ImportAll() Created new thread and moved processing into that thread. All other methods already used singalR to send updates back to the UI.  ?? will this be enough ?? 

"Synchronise Master Profiles" then "Check master profiles"
SynchroniseMasterProfilesController.CheckMasterProfiles()  This is the major blocker, this calls [usp_GetUnsynchronisedProfiles] which calls vw_VehicleProfile_UnsynchronisedDefaultProfiles and its in the view where the blocking occurs.
```

? do we move this method in to a new thread and allow it to run. If this then returns back to a progress page it will have a status of zero for X minutes and then go straight to 100% when it completes... .is this correct??

```
AmberSupportUI/ImportModels							ImportModelsController.cs
"Apply changes to the database"					ApplyChanges(string SignalRConnectionId)
"Check for incorrectly classified vehicles"		CheckVehicles(string SignalRConnectionId)
```

?? just move to a new thread ??
?? could almost see this being done by the AGS as a background import with a completion email or something ??

If the F5 notices that a request takes more then 5 minutes before it's returned the result, then it automatically resubmits the request. Instead of helping this actually means that after 5 minutes we have 2 copies of the job running, then after 10 minutes, we have 4 copies of the job running, ...

The only way to solve this is to ensure that we don't have any requests that take longer than 5 minutes.
This can be done by changing the initial request to just be a Start... request, and ensure that the waiting for completion is done separately.

From https://www.jerriepelser.com/blog/communicate-status-background-job-signalr/

MVC method to start the background job, but returns quickly:
[HttpPost]
public IActionResult StartProgress()
{
string jobId = Guid.NewGuid().ToString("N");
\_queue.QueueAsyncTask(() => PerformBackgroundJob(jobId));

```
    return RedirectToAction("Progress", new {jobId});
}
```

Private async method that executes the job slowly in the background:
private async Task PerformBackgroundJob(string jobId)
{
while(doing work)
{
// Keep reporting progress
await \_hubContext.Clients.Group(jobId).SendAsync("progress", progress);
}
}

Add to the Hub:
public async Task AssociateJob(string jobId)
{
await Groups.AddToGroupAsync(Context.ConnectionId, jobId);
}
and call from the Progress page in the client.

Change client side code to handle the job being completed, when informed via the callback,
not via the original request completing.

e.g.
Change SynchroniseMasterProfilesController.CheckMasterProfiles() to StartCheckMasterProfiles()
Put the call to:
\_vehicleProfileBroker.GetUnsynchronisedProfiles();
in a background CheckMasterProfilesAsync() job method.
Then you have 2 options:
1\. Change CheckMasterProfiles\.cshtml to not use the model to get the results\,
but to be passed them through the SignalR callback.
2\. Change StartCheckMasterProfiles\(\) to return CheckMasterProfilesProgress\.cshtml page\,
change CheckMasterProfilesAsync() to store the results in the session,
CheckMasterProfilesProgress page should show progress and waits for completion before redirecting to
a GetCheckedMasterProfiles action which retrieves the results from the stored session,
before using the existing CheckMasterProfiles.cshtml to render them.
As the results model is a list of 4 properties, I think I would try and get it to return the model in the SignalR callback,
and change the page to be populated from the Javascript model.
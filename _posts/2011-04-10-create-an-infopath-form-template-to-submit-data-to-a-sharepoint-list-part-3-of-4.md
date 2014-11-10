---
layout: post
image:
  feature: abstract-11.jpg
  featuredImage: /images/InfoPath-SharePoint-List/step6image5.png
comments: true
share: true
date: 2011-04-10
modified: 2014-11-09
title: 'Create an InfoPath form template to submit data to a SharePoint list. (Part 3 of 4)'
description: How can I submit data to a SharePoint list from an InfoPath form?
longDescription: This walkthrough take us through the steps in creating an InfoPath form to submit data to a SharePoint list.
categories: SharePoint
tags: [SharePoint 2010, InfoPath, Walkthrough]
---

The steps on creating the data connections and wiring up of these data connections with the field can be found on my previous blog :

> [Create an InfoPath form template to submit data to a SharePoint list.(Part 2 of 4)]({{site.url}}/sharepoint/create-an-infopath-form-template-to-submit-data-to-a-sharepoint-list-part-2-of-4/)

# Step 6 – The Coding / Submitting the Form


First step is to choose the language you want to write the code. I prefer C#.

Click on the Language button on the Developer Tab in the InfoPath ribbon, and select the code language and the project location.

[![Step6Image1]({{ site.url }}/images/InfoPath-SharePoint-List/step6image1.png)]({{ site.url }}/images/InfoPath-SharePoint-List/step6image1.png)

Click on the _Submit Options_ in the Data Tab.

[![Step6Image2]({{ site.url }}/images/InfoPath-SharePoint-List/step6image2.png)]({{ site.url }}/images/InfoPath-SharePoint-List/step6image2.png)

Select _Allow users to submit this form_ checkbox and select _Perform custom action using Code_. Click on the Edit Code button.

[![Step6Image3]({{ site.url }}/images/InfoPath-SharePoint-List/step6image3.png)]({{ site.url }}/images/InfoPath-SharePoint-List/step6image3.png)

<div class="note tip">
  <h5>Tip</h5>
  <p>
    If you get an error mentioning VSTA is not installed (Find the correct error code here), no need to worry. You have not installed VSTA while installing Office 2010. <br/>
    To get it installed, Go to Control Panel, Uninstall a program. Select Office 2010 from the list, right click and Select Change. <br/>
    In the Microsoft InfoPath section, select <i>Visual Studio Tools for Application</i>, select Run From my Computer, and click Continue.
  </p>
  <p><a href="{{ site.url }}/images/InfoPath-SharePoint-List/step6image4.png"><img src="{{ site.url }}/images/InfoPath-SharePoint-List/step6image4.png" alt="Step6Image4" /></a></p>
</div>

Before starting, lets add the following references.
{% highlight C# %}
System.Core
Microsoft.mshtml
Microsoft.SharePoint
Microsoft.SharePoint.Client
Microsoft.SharePoint.Client.Runtime              
{% endhighlight %}

And add the following import/using statements
{% highlight C# %}
using mshtml;
using Microsoft.SharePoint;
using Microsoft.SharePoint.Client;
using System.Collections.Generic;           
{% endhighlight %}

Let us now build a utility function to get the field values from the InfoPath Form.
{% highlight C# %}
/// <summary>
/// Gets field values
/// </summary> 

/// <param name="xpath"></param>
/// <returns></returns>
private string GetValue(string xpath)
{
    // return the value of the specified node
    XPathNavigator myNav = this.MainDataSource.CreateNavigator().SelectSingleNode(xpath, NamespaceManager);
    return (myNav != null) ? myNav.Value : string.Empty;
}
{% endhighlight %}

Now let us build a function to submit data to the SharePoint list.

This can be done using the `GetValue()` function we just wrote. So code for retrieving data from Title field will be
{% highlight C# %}
string title = GetValue("/my:myFields/my:Title");
{% endhighlight %}

The XPath value (/my:myFields/my:Title) can be retrieved by right clicking the required field from the Fields window on the right side of the InfoPath window and clicking ‘_Copy XPath_’ (Last but one option)

[![Step6Image5]({{ site.url }}/images/InfoPath-SharePoint-List/step6image5.png)]({{ site.url }}/images/InfoPath-SharePoint-List/step6image5.png)

Repeating the steps for all the fields, we will get a code as follows.
{% highlight C# %}
/// <summary>
/// Submits Content to a List
/// </summary>
private void SubmitToList(string listName)
{
    string title = GetValue("/my:myFields/my:Title");
    string description = GetValue("/my:myFields/my:Description");
    string businessNeed = GetValue("/my:myFields/my:BusinessNeed");
    string businessUnit = GetValue("/my:myFields/my:BusinessUnit");
    string projectType = GetValue("/my:myFields/my:ProjectType");
    string requestedBy = GetValue("/my:myFields/my:RequestedBy/pc:Person/pc:AccountId");
}
{% endhighlight %}

We now have the values for the drop down lists. Now we need to get the corresponding data field.
{% highlight C# %}
XPathNavigator businessUnitNav = this.DataSources["BusinessUnits"].CreateNavigator();
if (!string.IsNullOrEmpty(businessUnit)) 
    businessUnit = businessUnitNav
        .SelectSingleNode("/BusinessUnits/BusinessUnit[@Value='"
        + businessUnit 
        + "']", 
        NamespaceManager).GetAttribute("Data", "");
{% endhighlight %}

Repeating the steps for all the dropdown lists, we will get the following code.
{% highlight C# %}
XPathNavigator businessUnitNav = this.DataSources["BusinessUnits"].CreateNavigator();
XPathNavigator projectTypeNav = this.DataSources["ProjectType"].CreateNavigator(); 

if (!string.IsNullOrEmpty(businessUnit)) 
    businessUnit = businessUnitNav
        .SelectSingleNode("/BusinessUnits/BusinessUnit[@Value='" 
            + businessUnit + "']", 
        NamespaceManager).GetAttribute("Data", "");
        
if (!string.IsNullOrEmpty(projectType)) 
    projectType = projectTypeNav
        .SelectSingleNode("/ProjectTypes/ProjectType[@Value='" 
            + projectType + "']", 
        NamespaceManager).GetAttribute("Data", "");
{% endhighlight %}

Now we need to add the code to add everything into the SharePoint list. Finally The function will look like this.
{% highlight C# %}
/// <summary>
/// Submits Content to a List
/// </summary>
private void SubmitToList(string listName)
{
	string title = GetValue("/my:myFields/my:Title");
	string description = GetValue("/my:myFields/my:Description");
	string businessNeed = GetValue("/my:myFields/my:BusinessNeed");
	string businessUnit = GetValue("/my:myFields/my:BusinessUnit");
	string projectType = GetValue("/my:myFields/my:ProjectType");
	string requestedBy = GetValue("/my:myFields/my:RequestedBy/pc:Person/pc:AccountId"); 

	XPathNavigator businessUnitNav = this.DataSources["BusinessUnits"].CreateNavigator();
	XPathNavigator projectTypeNav = this.DataSources["ProjectType"].CreateNavigator();
	if (!string.IsNullOrEmpty(businessUnit))
		businessUnit = businessUnitNav
			.SelectSingleNode("/BusinessUnits/BusinessUnit[@Value='"
				+ businessUnit + "']", 
			NamespaceManager).GetAttribute("Data", "");
	if (!string.IsNullOrEmpty(projectType))
		projectType = projectTypeNav
			.SelectSingleNode("/ProjectTypes/ProjectType[@Value='" 
				+ projectType + "']", 
			NamespaceManager).GetAttribute("Data", "");

	//Adding the values to SharePoint List
	//This is the SharePoint list data connection we made in InfoPath form.
	SharePointListRWQueryConnection conn = (SharePointListRWQueryConnection) this.DataConnections["Business Requests"]; 
	Uri siteUri = conn.SiteUrl;

	SPSecurity.RunWithElevatedPrivileges(delegate()
	{
		using (SPSite site = new SPSite(siteUri.AbsoluteUri))
		{
			if (site != null)
			using (SPWeb web = site.OpenWeb(siteUri.AbsolutePath))
			{
				ClientContext clientContext = new ClientContext(siteUri.AbsoluteUri);
				Web clientweb = clientContext.Web;

				web.AllowUnsafeUpdates = true;

				SPList task = web.Lists[listName];
				SPListItem item = task.AddItem();
				item["Title"] = title;
				item["Description"] = description;
				item["Business Need"] = businessNeed;
				item["Business Unit"] = businessUnit;
				item["Project Type"] = projectType;

				SPFieldUserValueCollection fieldValues = new SPFieldUserValueCollection();
				if (!string.IsNullOrEmpty(requestedBy))
				{
					Microsoft.SharePoint.Client.User user = clientweb.EnsureUser(requestedBy);
					clientContext.Load(user);
					clientContext.ExecuteQuery();

					SPFieldUserValue requestedByValue = new SPFieldUserValue(web, user.Id, user.LoginName);
					fieldValues.Add(requestedByValue);
					item["Requested By"] = fieldValues;
				}
				else
				{
					//If the requested by column is blank, add the user who raised the request as Requested by
					Microsoft.SharePoint.Client.User user = clientweb.EnsureUser(this.User.LoginName);
					clientContext.Load(user);
					clientContext.ExecuteQuery();
					SPFieldUserValue requestedByValue = new SPFieldUserValue(web, user.Id, user.LoginName);
					fieldValues.Add(requestedByValue);
					item["Requested By"] = fieldValues;
				}

				item.Update();

				web.AllowUnsafeUpdates = false;
			}
		}
	});
}
{% endhighlight %}

Add the following lines of code to FormEvents_Submit() function.
{% highlight C# %}
Public void FormEvents_Submit(object sender, SubmitEventArgs e)
{
    // If the submit operation is successful, set
    // e.CancelableArgs.Cancel = false;
    // Write your code here.
    try
    {
        e.CancelableArgs.Cancel = false;
        SubmitToList("Business Requests");
    }
    catch (Exception ex)
    {
        e.CancelableArgs.Cancel = true;
        e.CancelableArgs.Message = ex.Message;
        e.CancelableArgs.MessageDetails = ex.ToString();
    }
}
{% endhighlight %}
Now let us go back to the form and wire up the submit button to submit form. Right click the Submit button, and select properties. On the following dialog box, Select Submit from the action and click apply.

[![Step6Image6]({{ site.url }}/images/InfoPath-SharePoint-List/step6image6.png)]({{ site.url }}/images/InfoPath-SharePoint-List/step6image6.png)

Before I forget, we need to change the security level of the InfoPath form. Click File from the ribbon and click Form Options button.

On the Form Options dialog box, click Security and Trust. Unclick Automatically determine security level and select Full Trust.

Click OK.

[![Step6Image7]({{ site.url }}/images/InfoPath-SharePoint-List/step6image7.png)]({{ site.url }}/images/InfoPath-SharePoint-List/step6image7.png)

Your InfoPath is ready to be used now. Press F5 on the Visual Studio and you should see the InfoPath working.

On Pressing F5, InfoPath will be launched and we might get a dialog box as below. Click Yes.

[![Step6Image8]({{ site.url }}/images/InfoPath-SharePoint-List/step6image8.png)]({{ site.url }}/images/InfoPath-SharePoint-List/step6image8.png)

This is my InfoPath form.

[![Step6Image9]({{ site.url }}/images/InfoPath-SharePoint-List/step6image9.png)]({{ site.url }}/images/InfoPath-SharePoint-List/step6image9.png)

The Steps 4 and 5 can be found in my previous blog – [<< Create an InfoPath form template to submit data to a SharePoint list.(Part 2 of 4)]({{site.url}}/sharepoint/create-an-infopath-form-template-to-submit-data-to-a-sharepoint-list-part-2-of-4/)

The next steps can be found in my next blog -[ Create an InfoPath form template to submit data to a SharePoint list.(Part 4 of 4) >>]({{site.url}}/sharepoint/create-an-infopath-form-template-to-submit-data-to-a-sharepoint-list-part-4-of-4/)

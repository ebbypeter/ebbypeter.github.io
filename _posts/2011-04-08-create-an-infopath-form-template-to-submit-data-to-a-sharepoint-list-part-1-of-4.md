---
layout: post
image:
  feature: abstract-11.jpg
comments: true
share: true
date: 2011-04-08
modified: 2014-11-09
title: 'Create an InfoPath form template to submit data to a SharePoint list. (Part 1 of 4)'
description: How can I submit data to a SharePoint list from an InfoPath form?
longDescription: This walkthrough take us through the steps in creating an InfoPath form to submit data to a SharePoint list.
categories: SharePoint
tags: [SharePoint 2010, InfoPath, Walkthrough]

---

The goal of this walkthrough is to create an InfoPath form template to submit data to a SharePoint list. Let us see how to make this in 8 Steps…

Assumptions made for this walkthrough are:  

  1. SharePoint 2010 environment is available.    
  2. Office 2010 x64 bit version is installed.  
  

# Step 1 – Creating the list


Create a new SharePoint list based on Tasks list and name the list **Business Requests**.

Add the following fields to the task list.  

  1. Business Need - Multiple lines of text / RichTextBox  
  2. Business Unit - Choice  
  3. Project Type - Choice  
  4. Requested By - Person or Group

Let the contents of the two choice fields be:  

| Business Unit               | Project Type   |
|:----------------------------|:---------------|
| Customer Services           | Infrastructure |
| Finance                     | IT             |
| Human Resources             | New Projects   |
| Information Technology      | Advertising    |
| Infrastructure Development  | Others         |
| Legal                       |                |
| Strategy and Planning       |                |
|----
{: rules="groups"}

The Business Requests list should look like this now.

[![Step1Image1]({{ site.url }}/images/InfoPath-SharePoint-List/step1image12.png)]({{ site.url }}/images/InfoPath-SharePoint-List/step1image12.png)


# Step 2 – Creating the XML files


We would need to make the XML files of the choice boxes so that the choices can be displayed inside the InfoPath form.

Let us make 2 XML files now, for Business Unit and Project Type.

The XMLs will look as follows: (The zip file contains the XML files)

### **BusinessUnits.Xml**
{% highlight xml %}
<?xml version=”1.0″ encoding=”utf-8″ ?>
<BusinessUnits>
 <BusinessUnit Value=""   Data="Select…"/>
 <BusinessUnit Value="1"  Data="Customer Services"/>
 <BusinessUnit Value="2"  Data="Finance"/>
 <BusinessUnit Value="3"  Data="Human Resources"/>
 <BusinessUnit Value="4"  Data="Information Technology"/>
 <BusinessUnit Value="5"  Data="Infrastructure Development"/>
 <BusinessUnit Value="6"  Data="Legal"/>
 <BusinessUnit Value="7"  Data="Strategy and Planning"/>
</BusinessUnits>
{% endhighlight %}
### **ProjectTypes.Xml**
{% highlight xml %}
<?xml version=”1.0″ encoding=”utf-8″ ?>
<ProjectTypes>
 <ProjectType Value=””  Data=”Select…”/>
 <ProjectType Value=”1” Data="Infrastructure"/>
 <ProjectType Value="2" Data=”IT”/>
 <ProjectType Value=”3” Data="New Projects"/>
 <ProjectType Value=”3” Data=”Advertising”/>
 <ProjectType Value=”3” Data=”Other”/>
</ProjectTypes>
{% endhighlight %}

# Step 3 – Creating InfoPath Forms
Launch Microsoft InfoPath Designer 2010, and select Blank Form from the available form templates.

[![Step3Image1]({{ site.url }}/images/InfoPath-SharePoint-List/step3image12.png)]({{ site.url }}/images/InfoPath-SharePoint-List/step3image12.png)

Select the required _Page Layout Template_, and theme from the _Page Design_ tab of the ribbon. You can also add table design from the _Insert_ tab. After this step, we are ready to add in the required fields into the InfoPath form.

I found it always easy to fill in all the information/text that is needed on the form, before we add any textboxes, dropdown list, buttons etc…

My InfoPath form looks like this now:

[![Step3Image2]({{ site.url }}/images/InfoPath-SharePoint-List/step3image22.png)]({{ site.url }}/images/InfoPath-SharePoint-List/step3image22.png)

Now let us add the textboxes, drop downs, buttons and person picker. Remember we have not wired it up to any data source yet. You can get all the required controls from the _Home Tab_, _Controls_ section in the ribbon.

I am adding the following controls for the fields  

  * Title – TextBox  
  * Description – RichTextBox  
  * Business Need – RichTextBox  
  * Business Unit – DropDownList  
  * Project Type – DropDownList  
  * Requested By – Person/Group Picker


I have also named the controls and made all the controls except Person/Group Picker cannot be blank. This can be done by right clicking the control and select properties.

[![Step3Image3]({{ site.url }}/images/InfoPath-SharePoint-List/step3image32.png)]({{ site.url }}/images/InfoPath-SharePoint-List/step3image32.png)

Add a button in the form and name it Submit. Save the form. Now my form looks like below. The InfoPath form is saved in Zip file.

[![Step3Image4]({{ site.url }}/images/InfoPath-SharePoint-List/step3image42.png)]({{ site.url }}/images/InfoPath-SharePoint-List/step3image42.png)

<div class="note download">
  <h5>Download / Links</h5>
  <p>The Business Request Form solution can be downloaded from here : 
    <a href="http://tq5bzg.bay.livefilestore.com/y1pdr8TsYDSSt2Y7b2V8PLlyPcCb1Y4bNLPDaDvw7QsOUiZGtBH09BAXfhC3lf1x4sB0TFjqlMqBFrIEtd_4cgsINleuNEX5Bzw/Business%20Request%20Form.zip?download&psid=1">Business Request Form Solution</a>
  </p>
</div>



The next steps can be found in my next blog - [Create an InfoPath form template to submit data to a SharePoint list.(Part 2 of 4) >>]({{site.url}}/sharepoint/create-an-infopath-form-template-to-submit-data-to-a-sharepoint-list-part-2-of-4/)

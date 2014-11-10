---
layout: post
image:
  feature: abstract-11.jpg
  featuredImage: /images/InfoPath-SharePoint-List/step6image5.png
comments: true
share: true
date: 2011-04-09
modified: 2014-11-09
title: 'Create an InfoPath form template to submit data to a SharePoint list. (Part 2 of 4)'
description: How can I submit data to a SharePoint list from an InfoPath form?
longDescription: This walkthrough take us through the steps in creating an InfoPath form to submit data to a SharePoint list.
categories: SharePoint
tags: [SharePoint 2010, InfoPath, Walkthrough]
---

The steps on designing the InfoPath form, the specification of the lists and the XML files are in the part 1 of the walkthrough which can be accessed from here :

>[Create an InfoPath form template to submit data to a SharePoint list.(Part 1 of 4)]({{site.url}}/sharepoint/create-an-infopath-form-template-to-submit-data-to-a-sharepoint-list-part-1-of-4/)

# Step 4 – Data Connections

Now let us start creating the data connections for the choice boxes and the form.

Click on the _Data Connections_ button on the _Data_ Tab in the InfoPath ribbon

[![Step4Image1]({{ site.url }}/images/InfoPath-SharePoint-List/step4image1.png)]({{ site.url }}/images/InfoPath-SharePoint-List/step4image1.png)

Click on Add Button on the _Data Connections_ dialog box, and click on _Create a new connection to _and _Receive Data_.

[![Step4Image2]({{ site.url }}/images/InfoPath-SharePoint-List/step4image2.png)]({{ site.url }}/images/InfoPath-SharePoint-List/step4image2.png)

Click Next and as source select XML document.

[![Step4Image3]({{ site.url }}/images/InfoPath-SharePoint-List/step4image3.png)]({{ site.url }}/images/InfoPath-SharePoint-List/step4image3.png)

Click Next, and on the next page click Resource Files. Click Add and include all the XML files one by one.

[![Step4Image4]({{ site.url }}/images/InfoPath-SharePoint-List/step4image4.png)]({{ site.url }}/images/InfoPath-SharePoint-List/step4image4.png)

Click BusinessUnit.xml and click OK. On the Data Connections Wizard, click Next and Finish.

[![Step4Image5]({{ site.url }}/images/InfoPath-SharePoint-List/step4image5.png)]({{ site.url }}/images/InfoPath-SharePoint-List/step4image5.png)

BusinessUnits data connection is now added. Repeat the steps for the other two XML files. This time you can just click on the Resource Files and select the file. We have already added the XML files in the resource files in the previous step.

[![Step4Image6]({{ site.url }}/images/InfoPath-SharePoint-List/step4image6.png)]({{ site.url }}/images/InfoPath-SharePoint-List/step4image6.png)

Let us now add the last data connection. This connection is to connect the InfoPath form to the target SharePoint list, which in our case is the Business Requests list.

Start by Clicking on the _Data Connections_ button in the _Data Tab_ in the Ribbon.

Click Add and Select _Create a new connection_ and _Receive data._ Click Next.

For the source of your data, Select _SharePoint Library or List_.

[![Step4Image7]({{ site.url }}/images/InfoPath-SharePoint-List/step4image7.png)]({{ site.url }}/images/InfoPath-SharePoint-List/step4image7.png)

Click Next and enter your SharePoint site where you have made the list.

[![Step4Image8]({{ site.url }}/images/InfoPath-SharePoint-List/step4image8.png)]({{ site.url }}/images/InfoPath-SharePoint-List/step4image8.png)

Click Next, and select our list (Business Request) from the lists and libraries.

[![Step4Image9]({{ site.url }}/images/InfoPath-SharePoint-List/step4image9.png)]({{ site.url }}/images/InfoPath-SharePoint-List/step4image9.png)

Click Next and select the required fields that you need to access from the form. I have selected all the fields anyway.

[![Step4Image10]({{ site.url }}/images/InfoPath-SharePoint-List/step4image10.png)]({{ site.url }}/images/InfoPath-SharePoint-List/step4image10.png)

Click Next, Next and Finish.


# Step 5 – Wiring up

Let us now connect the drop down boxes with the data connections we just created.

In the form, right click on the BusinessUnit drop down box and select ‘Drop-Down List Box Properties…’.

[![Step5Image1]({{ site.url }}/images/InfoPath-SharePoint-List/step5image1.png)]({{ site.url }}/images/InfoPath-SharePoint-List/step5image1.png)

On the List box choices, Select _Get choices from an external data source_ and Select _BusinessUnits_ from the Data Source.

Click on the Icon on the Entries textbox and select BusinessUnit as shown in the image below.

[![Step5Image2]({{ site.url }}/images/InfoPath-SharePoint-List/step5image2.png)]({{ site.url }}/images/InfoPath-SharePoint-List/step5image2.png)

Similarly click on the icon next to Value Text box and Display name text box and select Value and Data respectively.

[![Step5Image3]({{ site.url }}/images/InfoPath-SharePoint-List/step5image3.png)]({{ site.url }}/images/InfoPath-SharePoint-List/step5image3.png)

Click Ok and repeat the steps for Project Type Drop-Down box. My InfoPath form is available in the zip file.

<div class="note tip">
  <h5>Tip</h5>
  <p>You can always test the form by clicking on the preview button on the Home Tab.</p>
</div>

<div class="note download">
  <h5>Download / Links</h5>
  <p>The Business Request Form solution can be downloaded from here : 
    <a href="http://tq5bzg.bay.livefilestore.com/y1pdr8TsYDSSt2Y7b2V8PLlyPcCb1Y4bNLPDaDvw7QsOUiZGtBH09BAXfhC3lf1x4sB0TFjqlMqBFrIEtd_4cgsINleuNEX5Bzw/Business%20Request%20Form.zip?download&psid=1">Business Request Form Solution</a>
  </p>
</div>

The Steps 1, 2 and 3 can be found in my previous blog – [<< Create an InfoPath form template to submit data to a SharePoint list.(Part 1 of 4)]({{site.url}}/sharepoint/create-an-infopath-form-template-to-submit-data-to-a-sharepoint-list-part-1-of-4/)

The next steps can be found in my next blog - [Create an InfoPath form template to submit data to a SharePoint list.(Part 3 of 4) >>]({{site.url}}/sharepoint/create-an-infopath-form-template-to-submit-data-to-a-sharepoint-list-part-3-of-4/)

---
layout: post
image:
  feature: abstract-11.jpg
  featuredImage: /images/InfoPath-SharePoint-List/step6image5.png
comments: true
share: true
date: 2011-04-10
modified: 2014-11-09
title: 'Create an InfoPath form template to submit data to a SharePoint list. (Part 4 of 4)'
description: How can I submit data to a SharePoint list from an InfoPath form?
longDescription: This walkthrough take us through the steps in creating an InfoPath form to submit data to a SharePoint list.
categories: SharePoint
tags: [SharePoint 2010, InfoPath, Walkthrough]
---

Please find my previous blog to know more about submitting the InfoPath details to SharePoint

> [Create an InfoPath form template to submit data to a SharePoint list.(Part 3 of 4)]({{site.url}}/sharepoint/create-an-infopath-form-template-to-submit-data-to-a-sharepoint-list-part-3-of-4/)

# Step 7 – Publishing InfoPath form template into SharePoint

To publish this InfoPath form, click on the File Tab in the InfoPath and click Publish Button and click SharePoint Server.

On the Publishing Wizard, Enter the SharePoint site

[![Step7Image1]({{ site.url }}/images/InfoPath-SharePoint-List/step7image1.png)]({{ site.url }}/images/InfoPath-SharePoint-List/step7image1.png)

Click Next and select _Administrator –approved form template (advanced)_.

[![Step7Image2]({{ site.url }}/images/InfoPath-SharePoint-List/step7image2.png)]({{ site.url }}/images/InfoPath-SharePoint-List/step7image2.png)

Click Next and enter a location and file name for the InfoPath form.

[![Step7Image3]({{ site.url }}/images/InfoPath-SharePoint-List/step7image3.png)]({{ site.url }}/images/InfoPath-SharePoint-List/step7image3.png)

Click Next and continue with the wizard.

Now we have an XSN file that needs to be published into SharePoint.

To do this, start SharePoint Central Admin in Administrative mode.

[![Step7Image4]({{ site.url }}/images/InfoPath-SharePoint-List/step7image4.png)]({{ site.url }}/images/InfoPath-SharePoint-List/step7image4.png)

Click on General Application Settings and then under InfoPath form services, click upload form template.

[![Step7Image5]({{ site.url }}/images/InfoPath-SharePoint-List/step7image5.png)]({{ site.url }}/images/InfoPath-SharePoint-List/step7image5.png)

Select the published xsn file in the next window and click upload, and next page click OK.

[![Step7Image6]({{ site.url }}/images/InfoPath-SharePoint-List/step7image6.png)]({{ site.url }}/images/InfoPath-SharePoint-List/step7image6.png)

Now we are on the Manage Form Templates page. Click the arrow on the right side of our form and click Activate to a Site Collection.

[![Step7Image7]({{ site.url }}/images/InfoPath-SharePoint-List/step7image7.png)]({{ site.url }}/images/InfoPath-SharePoint-List/step7image7.png)

Select the required site collection and click OK.

The InfoPath form is now published on to the specified site collection.


# Step 8 – Final Steps


Go to the site collection on which we have activated the InfoPath form

Click All Site Contents and select Form Templates.

You can see our InfoPath form inside this library. Click on the InfoPath form and select Edit in Browser.

Save the URL, and share it as a link on SharePoint Site or with users. Clicking on the link will open the InfoPath form and on submitting, the data will be stored in the Business Requests list.

Please put a comment in case of any concerns, questions or problems.

The Steps 4 and 5 can be found in my previous blog – [<< Create an InfoPath form template to submit data to a SharePoint list.(Part 3 of 4)]({{site.url}}/sharepoint/create-an-infopath-form-template-to-submit-data-to-a-sharepoint-list-part-3-of-4/)

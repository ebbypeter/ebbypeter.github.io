---
layout: post
image:
  feature: abstract-11.jpg
  featuredImage: /images/SharePoint-2007-List-Template/image.png
comments: true
share: true
date: 2011-08-25
modified: 2014-11-09
title: 'Convert a SharePoint 2007 list template STP to SharePoint 2010'
description: How can I convert a SharePoint 2007 list template STP to SharePoint 2010?
longDescription: How to convert a SharePoint 2007 list template STP to work in SharePoint 2010.
categories: SharePoint
tags: [SharePoint 2010, InfoPath, Walkthrough]
---

Last day, when I was trying to use a MOSS 2007 list template STP in SharePoint 2010, I was met with an error.

[![image]({{ site.url }}/images/SharePoint-2007-List-Template/image.png)]({{ site.url }}/images/SharePoint-2007-List-Template/image.png)

Microsoft SharePoint Foundation version 3 templates are not supported in this version of the product.  

Correlation ID: `{9809bf7a-457b-4c67-bca4-489b5957b8ea}`  

I tried opening an STP of SharePoint 2007 and SharePoint 2010 list template, and compared. The only major difference I found was the version number inside the file’s XML data. Updating the version number in the XML to one that corresponds to SharePoint 2010, made the 2007 list template work in 2010.  

So if you need to migrate a list template STP from SharePoint 2007 to SharePoint 2010, you could try the following steps:  

> **Step 1 :** Rename the original STP to CAB  
> 
> **Step 2 :** Extract the CAB File (I used WinRar) contents to a folder (let’s call it SourceFolder)  
> 
> **Step 3 :** Edit the manifest.xml file, search for the ProductVersion element. This should have a value 3 if it is a SharePoint 2007 list template.  
> 
> **Step 4 :** Change the ProductVersion value to 4  
> 
> **Step 5 :** Repackage the manifest.xml into a .CAB. I have used the MakeCab.exe which is available in Windows.  
> 
> Syntax to package is  
> 
> Makecab <SourceFolder\manifest.xml> <SourceFolder\templateName.cab>  
> 
> **Step 6:** Change the extension from .CAB to .STP and upload it

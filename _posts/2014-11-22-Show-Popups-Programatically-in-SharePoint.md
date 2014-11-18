---
layout: post
image:
  feature: abstract-11.jpg
  featuredImage: /images/SharePoint-Popups/AllPopups.jpg
comments: true
share: true
title: Show Dialog box / Popups programmatically in SharePoint 2013
description: How can I display a dialog box / popup in SharePoint 2013 programatically?
longDescription: Popup / Dialog box is a very important component in SharePoint UI. It can be used to display messages, get information, or open external websites. <br/></br/>In this post we will see how to create a Popup box or Dialog box in SharePoint programmatically.
categories: SharePoint
tags: [SharePoint, SharePoint 2013, C#, JavaScript]
---

##What are we trying to achieve
Programmatically show Dialog box / Popups in SharePoint 2013 with custom code / C#.  
In this post, we will be building two types of dialog boxes.  
In the first type, clicking on the "OK" button refreshes the page. This is useful in situations where we are displaying a process complete message or in similar situations.  
In the second type, clicking on the "OK" button redirects you to a different page. This can be used in situations where a confirmation needs to be taken etc..  

The Result is something like what is shown below.  
[![Results]({{ site.url }}/images/SharePoint-Popups/Popup.JPG)]({{ site.url }}/images/SharePoint-Popups/Popup.JPG)  

<div class="note tip">
  <h5>Tip</h5>
  <p>If you are familiar with how Model popup &amp; dialog box works in SharePoint, then you can skip to the implementation part.</p>
</div>

##How it is done in SharePoint Product?##
Before we continue, let’s revisit how the OOB dialog boxes are displayed.  
When a pop up box needs to be displayed, an application page is loaded using an OOB Model Dialog JavaScript.  

For example, consider clicking the `New Document` button in a Document Library; using Fiddler, we can see that the application page called is  
_http://howcanidoit.fourthcoffee.dev_**/_layouts/15/Upload.aspx?List={4B9CFE85-543E-4B64-9D12-CF7ADA634B16}&RootFolder=&IsDlg=1**  

The upload page is as follows  
[![Upload Page]({{ site.url }}/images/SharePoint-Popups/Upload_Normal.JPG)]({{ site.url }}/images/SharePoint-Popups/Upload_Normal.JPG)  

And opening the url gives us this page  
[![Upload Page Link]({{ site.url }}/images/SharePoint-Popups/Upload_AppPage_IsDlg.JPG)]({{ site.url }}/images/SharePoint-Popups/Upload_AppPage_IsDlg.JPG)  

Similarly Check-In calls the page  
_http://howcanidoit.fourthcoffee.dev_**/_layouts/15/checkin.aspx?List={4B9CFE85-543E-4B64-9D12-CF7ADA634B16}&FileName=%2FDocuments%2FRandom%20Document%2Edocx&IsDlg=1**  

Well… You get the idea…  

The url contains some parameters relevant to the page we are visiting and an **`IsDlg=1`** property.  

The JavaScript APIs to display the dialog box are present in **`SP.UI.Dialog.Js`** file and we can use `SP.UI.ModalDialog.showModalDialog(options)` to display the dialog box.

<div class="note info">
  <h5>Note</h5>
  <p>The corresponding images for Check-in is available in the following links <br/>
    <a href="{{ site.url }}/images/SharePoint-Popups/Checkin_Normal.JPG" alt="Alternate">CheckIn Page</a><br/>
    <a href="{{ site.url }}/images/SharePoint-Popups/Checkin_AppPage_IsDlg.JPG">CheckIn Page Url</a><br/>
    <a href="{{ site.url }}/images/SharePoint-Popups/Checkin_AppPage.JPG">CheckIn Page url without IsDlg</a><br/>
  </p>
</div>

##The IsDlg Property##
You would have noticed that opening the url gave a basic page and does not look anything like an application page. This is achieved through the IsDlg=1 parameter passed in the query.  
The `IsDlg=1` parameter in the query indicate to SharePoint that the application page is intended to be displayed inside a dialog box. This causes SharePoint to display only the content of the page and hide the SharePoint chrome (Header / footer etc).  

So opening the Upload Url without the `IsDlg=1` parameter gives the following page  
[![Upload Page without IsDlg]({{ site.url }}/images/SharePoint-Popups/Upload_AppPage.JPG)]({{ site.url }}/images/SharePoint-Popups/Upload_AppPage.JPG)  
That looks like a normal application page.  
The Url used was
_http://howcanidoit.fourthcoffee.dev_**/_layouts/15/Upload.aspx?List={4B9CFE85-543E-4B64-9D12-CF7ADA634B16}&RootFolder=**  

##Implementation##
That’s enough of theory… Let us implement it…  

There are four parts to the solution.  

1.	**Application Page**	- Displays the message that we need to show.
2.	**Wrapper Class**	- Optionally required to create the correct application page url (with the parameters) and JavaScript function call.
3.	**JavaScript functions**	- Make sure that the SP.UI.Dialog.Js is loaded and calls the required APIs.
4.	**Constants Class**	- Optional. Holds all the constants value.

###Application Page###

**Aspx Page**
{% highlight HTML linenos %}
<asp:Content ID="PageHead" ContentPlaceHolderID="PlaceHolderAdditionalPageHead" runat="server">
    <Sharepoint:ScriptLink runat="server" Name="SP.js" Localizable="false"  ID="s1" LoadAfterUI="true"/>
    <Sharepoint:ScriptLink runat="server" Name="SP.Runtime.js" Localizable="false"  ID="s2" LoadAfterUI="true"/>
</asp:Content>

<asp:Content ID="PageTitle" ContentPlaceHolderID="PlaceHolderPageTitle" runat="server">
    <%= MessageTitleQueryValue %>
</asp:Content>

<asp:Content ID="PageTitleInTitleArea" ContentPlaceHolderID="PlaceHolderPageTitleInTitleArea" runat="server" >
    <%= MessageTitleQueryValue %>
</asp:Content>

<asp:Content ID="Main" ContentPlaceHolderID="PlaceHolderMain" runat="server">
    <div style="min-height: 150px; padding-bottom: 10px;">
        <span>
            <img style="float: left; width: 50px;" src="<%= MessageImageValue %>" alt="<%= MessageImageText %>"/>
        </span>
        <p style="margin-left: 60px; min-height: 100px;">
            <%= MessageContentValue %>
        </p>
    </div>
    <div style="text-align: right;">
        <% if (MessageButtonOptions.Equals("Ok"))
           { %>
            <asp:Button runat="server" ID="OkButton" Text="OK" OnClick="OkButton_OnClick"/>      
        <% }
           else if (MessageButtonOptions.Equals("OkCancel"))
           { %>
            <asp:Button runat="server" ID="OkButton1" Text="OK" OnClick="OkButton_OnClick"/> &nbsp;&nbsp;
            <asp:Button runat="server" ID="CancelButton" Text="Cancel" OnClick="CancelButton_OnClick"/>      
        <% } %>
        
    </div>
</asp:Content>
{% endhighlight %}

**Aspx/C# Code Behind**
{% highlight C# linenos %}
using System;
using System.Linq;
using Microsoft.SharePoint.WebControls;
using SharePointPopups.Common;

namespace SharePointPopups.Layouts.SharePointPopups
{
    /// <summary>
    /// Show Message Application Page
    /// </summary>
    public partial class ShowMessage : LayoutsPageBase
    {
        //Constants      
        public string MessageButtonOptions = string.Empty;
        public string MessageContentValue = string.Empty;
        public string MessageImageText = string.Empty;
        public string MessageImageValue = string.Empty;
        public string MessageTitleQueryValue = string.Empty;

        protected void Page_Load(object sender, EventArgs e)
        {
            if (Request.QueryString.AllKeys.Contains(Constants.MessageContentQueryName))
            {
                MessageContentValue = Request.QueryString[Constants.MessageContentQueryName];
            }

            if (Request.QueryString.AllKeys.Contains(Constants.MessageTitleQueryName))
            {
                MessageTitleQueryValue = Request.QueryString[Constants.MessageTitleQueryName];
            }

            if (Request.QueryString.AllKeys.Contains(Constants.MessageTypeQueryName))
            {
                MessageImageText = Request.QueryString[Constants.MessageTypeQueryName];
                switch (Request.QueryString[Constants.MessageTypeQueryName].ToUpperInvariant())
                {
                    case "ERROR":
                        MessageImageValue = Constants.ErrorImage;
                        break;
                    case "WARNING":
                        MessageImageValue = Constants.WarnImage;
                        break;
                    case "INFORMATION":
                        MessageImageValue = Constants.InfoImage;
                        break;
                }
            }

            if (Request.QueryString.AllKeys.Contains(Constants.MessageOptionsQueryName))
            {
                MessageButtonOptions = Request.QueryString[Constants.MessageOptionsQueryName];
            }
        }

        protected void OkButton_OnClick(object sender, EventArgs e)
        {
            Page.ClientScript.RegisterStartupScript(GetType(), "PopupScript",
                "SP.UI.ModalDialog.commonModalDialogClose(SP.UI.DialogResult.OK, 'OK');", true);
        }

        protected void CancelButton_OnClick(object sender, EventArgs e)
        {
            Page.ClientScript.RegisterStartupScript(GetType(), "PopupScript",
                "SP.UI.ModalDialog.commonModalDialogClose(SP.UI.DialogResult.cancel, 'Cancel');", true);
        }
    }
}
{% endhighlight %}

####Wrapper Class####  
{% highlight C# linenos %}
namespace SharePointPopups.Common
{
    public static class PopupOperations
    {
        public static string ShowPopup(Constants.PopupType messageType, string title, string message,
            Constants.PopupButtons buttons, int width, int height, bool refreshPage)
        {
            const string pageUrlFormat = "{0}?Type={1}&Title={2}&Message={3}&Options={4}&isDlg=1";
            const string dialogStringFormat = "openInDialog({0}, {1}, false, true, {2},'{3}');";

            string pageUrl = string.Format(pageUrlFormat, Constants.ShowMessagePage, messageType, title, message,
                buttons);
            string dialogString = string.Format(dialogStringFormat, width, height, refreshPage.ToString().ToLower(),
                pageUrl);

            return dialogString;
        }

        public static string ShowPopupAndRedirect(Constants.PopupType messageType, string title, string message,
            Constants.PopupButtons buttons, int width, int height, string redirectUrl)
        {
            const string pageUrlFormat = "{0}?Type={1}&Title={2}&Message={3}&Options={4}&isDlg=1";
            const string dialogStringFormat = "openInDialogAndRedirect({0}, {1}, false, true, '{2}','{3}');";

            string pageUrl = string.Format(pageUrlFormat, Constants.ShowMessagePage, messageType, title, message,
                buttons);
            string dialogString = string.Format(dialogStringFormat, width, height, pageUrl, redirectUrl);

            return dialogString;
        }
    }
}
{% endhighlight %}  

####JavaScript Functions####  
{% highlight JavaScript linenos %}
function openInDialog(dlgWidth, dlgHeight, dlgAllowMaximize, dlgShowClose, needCallbackFunction, pageUrl ) {
    var options = {
        url: pageUrl,
        width: dlgWidth,
        height: dlgHeight,
        allowMaximize: dlgAllowMaximize,
        showClose: dlgShowClose
    };

    if (needCallbackFunction) {
        options.dialogReturnValueCallback = Function.createDelegate(null, CloseDialogCallback);
    }
    SP.SOD.execute('sp.ui.dialog.js', 'SP.UI.ModalDialog.showModalDialog', options);
}


var redirectUrlValue = "";

function openInDialogAndRedirect(dlgWidth, dlgHeight, dlgAllowMaximize, dlgShowClose, pageUrl, redirectUrl ) {
    var options = {
        url: pageUrl,
        width: dlgWidth,
        height: dlgHeight,
        allowMaximize: dlgAllowMaximize,
        showClose: dlgShowClose
    };
    redirectUrlValue = redirectUrl;
    options.dialogReturnValueCallback = Function.createDelegate(null, CloseDialogAndRedirectCallback);

    SP.SOD.execute('sp.ui.dialog.js', 'SP.UI.ModalDialog.showModalDialog', options);
}

function CloseDialogCallback(dialogResult, returnValue) {
    if (dialogResult == SP.UI.DialogResult.OK) { // refresh parent page
        SP.SOD.execute('sp.ui.dialog.js', 'SP.UI.ModalDialog.RefreshPage', SP.UI.DialogResult.OK);

    }
    // if user click on Close or Cancel
    else if (dialogResult == SP.UI.DialogResult.cancel) { // Do Nothing or add any logic you want 
    } else { //alert("else " + dialogResult);
    }
}

function CloseDialogAndRedirectCallback(dialogResult, returnValue) {
    if (dialogResult == SP.UI.DialogResult.OK) { // refresh parent page
        SP.SOD.execute('sp.ui.dialog.js', 'SP.UI.ModalDialog.RefreshPage', SP.UI.DialogResult.OK);
        window.location.href = redirectUrlValue;
    }
    // if user click on Close or Cancel
    else if (dialogResult == SP.UI.DialogResult.cancel) { // Do Nothing or add any logic you want 
    } else { //alert("else " + dialogResult);
    }
}
{% endhighlight %}  

####Constants Class####
{% highlight C# linenos %}

namespace SharePointPopups.Common
{
    public static class Constants
    {
        public enum PopupButtons
        {
            Ok,
            OkCancel
        }

        public enum PopupType
        {
            Information,
            Warning,
            Error
        }

        public const string ErrorImage = "/Style%20Library/Custom%20Images/Error.png";
        public const string WarnImage = "/Style%20Library/Custom%20Images/Warn.png";
        public const string InfoImage = "/Style%20Library/Custom%20Images/Info.png";

        public const string MessageTitleQueryName = "Title";
        public const string MessageContentQueryName = "Message";
        public const string MessageTypeQueryName = "Type";
        public const string MessageOptionsQueryName = "Options";

        public const string ShowMessagePage = "/_layouts/15/SharePointPopups/ShowMessage.aspx";
    }
}
{% endhighlight %}

##Usage##

This is how we call the Dialog box from C#.
What we are actually doing here is calling the JavaScript Function using `Page.ClientScript.RegisterStartupScript`.  
`PopupOperations.ShowPopup` &amp; `PopupOperations.ShowPopupAndRedirect` creates the required function call with the appropriate urls.  
The last parameter in the `PopupOperations.ShowPopup` defines if the page needs to be refreshed.  
Similarly the last parameter in the `PopupOperations.ShowPopupAndRedirect` defines the page to which the user needs to be redirected on pressing "OK"

###Show Popup###
{% highlight C# linenos %}
Page.ClientScript.RegisterStartupScript(typeof (Page), "Popout Script",
                PopupOperations.ShowPopup(Constants.PopupType.Information,
                    "Information Message",
                    "This is a popup with OK Button. Clicking OK Will refresh the page.",
                    Constants.PopupButtons.Ok,
                    500, 210, true)
                , true);
{% endhighlight %}

###Show Popup &amp; Redirect###
{% highlight C# linenos %}
Page.ClientScript.RegisterStartupScript(typeof (Page), "Popout Script",
                PopupOperations.ShowPopupAndRedirect(Constants.PopupType.Error,
                    "Error Message",
                    "This is a popup with OK Button. Clicking OK will redirect to the settings page.",
                    Constants.PopupButtons.Ok,
                    500, 210,
                    "/_layouts/15/settings.aspx")
                , true);
{% endhighlight %}

<div class="note download">
  <h5>Downloads</h5>
  <p>The complete source code that demonstrates the concept and all the code that is described in this post is available at GitHub in the following address <br/>
    <a href="https://github.com/ebbypeter/SharePointPopups.git" alt="Sample Source Code">SharePoint Popups</a><br/>
  </p>
</div>

##Solution Description##
The solution structure and the webpart is shown below.  
Once the solution is deployed, the webpart can be found in the WebParts Gallery under the `HowCanIDoIt` category.

[![Solution Structure]({{ site.url }}/images/SharePoint-Popups/SolutionStructure.JPG)]({{ site.url }}/images/SharePoint-Popups/SolutionStructure.JPG)  
{: .pull-right}

[![Solution Structure]({{ site.url }}/images/SharePoint-Popups/Webpart.JPG)]({{ site.url }}/images/SharePoint-Popups/Webpart.JPG)  

<div class="note download">
  <h5>Downloads</h5>
  <p>The complete source code that was used in this post can be found at <br/>
    <a href="https://github.com/ebbypeter/SharePointPopups.git" alt="Sample Source Code">SharePoint Popups</a><br/>
  </p>
</div>

---
Liked this post or have feedbacks / corrections for me? Let me know in the comments below.
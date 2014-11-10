---
layout: post
image:
  feature: abstract-11.jpg
  featuredImage: /images/minimize-an-application/image.png
comments: true
share: true
modified: 2014-11-09
title: Minimize an application to System Tray in C#
description: How can I minimize an application to System Tray in C#?
longDescription: In this post we will see how we can minimize an application to System Tray in C#.
categories: C#
tags: [C#]
---

To minimize an application form to System Tray, we can use the **NotifyIcon** control in Visual Studio.

[![NotifyIcon]({{ site.url }}/images/minimize-an-application/thumb/image.png)]({{ site.url }}/images/minimize-an-application/image.png)
{: .pull-right}

**NotifyIcon** is in the **System.Windows.Forms** namespace.
Drag and drop a **NotifyIcon** control to your form.

To minimize the application to the system tray we need to handle the appropriate events. There are multiple events that can be used to achieve the output... The most appropriate will be the `Form Resize` event or the `Form SizeChanged` event.

<br/><br/><br/>
I am using `Form Resize` event in the following snippet.

{% highlight C# linenos %}
private void Form1_Resize(object sender, EventArgs e)               
{                
    if (FormWindowState.Minimized == this.WindowState)                
    {                
        notifyIcon1.BalloonTipTitle = "Minimize Sucessful";                
        notifyIcon1.BalloonTipText = "Sucessfully minimized app to the System Tray";                
        notifyIcon1.Visible = true; //Show the notifyIcon in the system tray                
        notifyIcon1.ShowBalloonTip(500); // Show the notification balloon                
        this.Hide(); //hide the form                
    }                
    else if (FormWindowState.Normal == this.WindowState)                
    {                
        notifyIcon1.Visible = false; //Hide the tray icon when the form is displayed                
    }                
}
{% endhighlight %}
      
<div class="note tip">
  <h5>Tip</h5>
  <p>If you are not interested in the balloon tip, we can omit the BalloonTip and ShowBalloon lines. </p>
</div>


So now we managed to minimize the application to the System Tray. To revive it back we need to write a couple more lines. <br/>
This time the event to be handled is `NotifyIcon MouseDoubleClick` or `NotifyIcon MouseClick`.

{% highlight C# linenos %}
private void notifyIcon1_MouseDoubleClick(object sender, MouseEventArgs e)               
{                
    this.Show();                
    notifyIcon1.Visible = false;                
}
{% endhighlight %}

---
layout: post
image:
  feature: abstract-11.jpg
  featuredImage: /images/minimize-a-wpf-application/image1.png
comments: true
share: true
modified: 2014-11-09
title: Minimize a WPF application to System Tray in C#
description: How can I minimize a WPF application to System Tray in C#?
longDescription: In this post we will see how we can minimize a WPF application to System Tray using C#.
categories: WPF
tags: [C#, WPF]
---

There is no direct way to minimize a WPF application to system tray. This does not means that we cannot do that at all. 

[![image]({{ site.url }}/images/minimize-a-wpf-application/thumb/image1.png)]({{ site.url }}/images/minimize-a-wpf-application/image1.png)
{: .pull-right}

<br/>
To minimize WPF app to System Tray/add the Tray notification, we just need to add the "`System.Windows.Forms`" and "`System.Drawing`" assembly references to your WPF project. Once we add the references, we can create new instance of the “NotifyIcon” class and use it in our WPF application.  

<br/><br/>  
After we add the references to the WPF application, we have to create a global variable for **NotifyIcon**.

{% highlight C# %}
private System.Windows.Forms.NotifyIcon MyNotifyIcon;
{% endhighlight %}


Now we have to initiate the NotifyIcon. This can be done in the Window’s constructor. We also have to declare an Event to retrieve back the application from SystemTray.

{% highlight C# linenos %}
public MainWindow()
{
    InitializeComponent();
    MyNotifyIcon = new System.Windows.Forms.NotifyIcon();
    MyNotifyIcon.Icon = new System.Drawing.Icon(@"C:\Archive\Icon-Archive.ico");
    MyNotifyIcon.MouseDoubleClick += 
        new System.Windows.Forms.MouseEventHandler(MyNotifyIcon_MouseDoubleClick);
}
{% endhighlight %}

    
To minimize the application to the system tray we need to handle the appropriate events. There are multiple events that can be used to achieve the output… The most appropriate will be the `Form SizeChanged` event.

To revive the app back we need to write a couple more lines. This time the event to be handled is `NotifyIcon MouseDoubleClick` or `NotifyIcon MouseClick`.

{% highlight C# linenos %}
void MyNotifyIcon_MouseDoubleClick(object sender, System.Windows.Forms.MouseEventArgs e)
{
    this.WindowState = WindowState.Normal;
}
    
private void Window_StateChanged(object sender, EventArgs e)
{
    if (this.WindowState == WindowState.Minimized)
    {
        this.ShowInTaskbar = false;
        MyNotifyIcon.BalloonTipTitle = "Minimize Sucessful";
        MyNotifyIcon.BalloonTipText = "Minimized the app";
        MyNotifyIcon.ShowBalloonTip(400);
        MyNotifyIcon.Visible = true;
    }
    else if (this.WindowState == WindowState.Normal)
    {
        MyNotifyIcon.Visible = false;
        this.ShowInTaskbar = true;
    }
}
{% endhighlight %}

<div class="note tip">
  <h5>Tip</h5>
  <p>If you are not interested in the balloon tip, we can omit the BalloonTip and ShowBalloon lines. </p>
</div>
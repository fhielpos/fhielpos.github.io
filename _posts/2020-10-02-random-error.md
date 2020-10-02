---
layout: post
title: "Random error"
---
Have you ever had an ultra-random error?

Once I was going back home from the office when I received an alert that all the services were down. My first reaction was to check if the site still had internet (was everything self-hosted with only one provider at that time), which was the case, so I tried to SSH into a VM with no luck, I connected a TeamViewer to the server and everything seemed ok. No choice, I had to come back to the office.

Basic troubleshooting later:
-	Ping to VMs ok
-	Ping to internet ok
-	Ping from VMs to server not ok
-	Ping from VMs to internet not ok

All the VMs failing at the exact same time, with no apparent reason. Huh.

I rebooted one VM to see if the error persisted. It did. Okay, reboot the whole VM service. Now everything works again. Weird.

I’m still in the office. Let’s make it brake again so I don’t have to deal with this again. My first reaction was to unplug the network cable from the server and see if that recreates the problem. It did, indeed. Again, restart the whole service and try restarting the router now, just to make sure if it’s only when there’s no link or no internet. It broke again.

Definitely this VM solution was not the best. But what a weird problem it was.

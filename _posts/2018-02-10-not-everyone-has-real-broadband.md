---
layout: post
title: "Not everyone has real broadband"
date: "2018-02-10 14:00:00 -0200"
excerpt: A plea for developers and designers to remember those of us with cringeworthy internet speeds.
---

Brazil, like the US, is a country of continental proportions and diversity. And much like in the US, internet access is not that great: slow speeds, high prices, and very few alternatives. That is, assuming you have an alternative to begin with.

I first got broadband access at home in 2001. With a 256 Kbps DSL connection, I was happy that I could stay online all day long and download things 5 times faster than over the previous dial-up connection. A few years later, the downstream speed had doubled to a whopping 500 Kbps. By 2009, it was bumped to 2 Mbps, still ADSL. Progress seemed inevitable, but then it stopped there.

In 2014, I moved to another house one mile away, and there, not even DSL was available: I had to use a ~8 Mbps 3G connection with a 20 GB monthly data cap. DSL access became available one year later, at the mind-blowing speed of 4 Mbps.

But this article is not yet another rant about poor internet access. Instead, it is a plea for developers and designers to adapt a few of their practices so as to improve the user experience of those unlucky among us: the ones stuck with internet speeds that make most people cringe when you tell them.

During all these years behind shitty internet links, I noticed a few things that seem innocent at first, but are really annoying when you are behind a narrow internet pipe:

**Hiding download/upload rates**

This wasn't so much of a problem some years ago, but it seems that as time goes by and everyone gets very fast internet access, showing download/upload information in the UI has become a thing of the past. "Please wait" is not really useful when you have slow internet (you're pretty much waiting most of the time anyway).

Knowing the download rate can help determine when you need to yell at another household member to stop watching cute dog videos. Download size and time-left estimations are also really useful so you can decide whether to go for a cup of coffee, for a walk, or even go get some sleep :-)

Windows Update and some other Microsoft tools exhibit this problem to an annoying degree, but I'm seeing the disapperance of download/upload information in other software and OS as well. One partial workaround is to open a system monitoring tool to check the network speed, but that not very informative nor convenient.

I understand there's a trend towards simplification of user interfaces, and grandma doesn't really care that Windows Update is downloading 450 MB at 2 MB/s. But there are other ways this could be shown, perhaps as tooltips, or expandable interface elements. It wouldn't hurt anyone, but it'd certainly make some people happier :-)

**Constant app updates**

Some developers **really** like updating their apps. While it's nice to have up-to-date software, with all the bug fixes and improvements that come with it, it can also become an annoyance for those with slow internet links. You end with dozens of 30 MB-plus app updates every week, and while those are downloading, internet speeds are noticeably affected for everyone else.

I don't mind having frequent updates, but do we really need two updates for Google Sheets every week?

A possible solution would be having a distinction for optional and recommended updates, so that auto-updating policies could be applied at the user's discretion.

**Slow websites**

This one should go without saying, you know the drill: avoid large images and resources, don’t use 12 different ad networks in one page, don’t embed autoplay videos and follow well-known optimization guidelines. Some browsers will even detect common problems and suggest optimizations. Your mobile users will also thank you.

**Treating wifi connections as free-for-all**

When I had that data capped 3G connection, I connected an iPad to the network, and it decided it could upload and sync backups, photos and everything else. In that first month, I ended up reaching the data cap in a few days just because of that iPad. It was an oversight of my part, but I didn't think my mother could be such an avid content producer :-)

The solution was to disable all sorts of cloud sync, but there could be a better way. I think Windows has this feature, and everyone else should copy it: just let us set some wifi connections as metered, and then establish OS and application download/upload behavior around that.

**200 MB-plus drivers**

This one is specific to Windows environments: drivers are getting bigger, and sometimes unnecessarily so, it seems.

nVidia issues frequent updates for their GPU drivers, and every one of those requires a 400 MB download. That's 15 minutes of downtime for someone with a 4 Mbps link. It'd be nice if those updates were diff/partial downloads, or segmented for GPU families.

Razer requires a 200 MB-plus download and a cloud service registration just for turning off the LED in my mouse. And Intel's drivers are not that compact either.

**Update tools that require user intervention**

Again, this one is about Windows, and I think it annoys everyone: you tell Windows to install updates, it fails. You have to click again, it downloads and asks you to reboot. You reboot, and when you check, it seems Windows forgot to download or install some updates. Now imagine how awful that's when it takes 15 minutes between one error and another.

**Automatic downloads**

Automatically downloading content and applications without the user's permission should not happen, period. It's an intrusive practice on its own. But it's even more of a problem when that user has limited internet access. Windows (again...) has the terrible habit of downloading and updating apps that I never asked to be installed in the first place.

One time I noticed download speeds were worse than usual. I opened the Task Manager and saw that my network interface was in fact downloading at full speed (4 Mbps...). After digging a bit I found the Windows Store downloading apps I didn't want. This same type of behavior also happens from time to time on Android, so it's not exclusively a Windows thing.

> Always ask yourself: what if I had only 10% of the bandwidth I have now?

The experience of a sizeable portion of your users will be improved if you develop your application, website or service with these things in mind. When designing a user interface, or adding a feature that seems automagic, or when asking your users to update your application, always ask yourself: what if I had only 10% of the bandwidth I have now? I guarantee many people will be positively affected by the design choices you make after answering that question.

For those who may be wondering: despite the problems I mentioned above, the online experience in 2018 is ok at 4 Mbps. You get to watch Netflix (1 stream at 720p is fine, 2 simultaneous streams causes noticeable quality loss). You can download and play games online if you are willing to wait a bit, and two or three people can browse the web comfortably at the same time. Sometimes you have to yell at your relatives because you need to load something fast. And if you don’t use your LAN that much, you can get away with pretty much any wifi router: I swear it won’t be slower than your WAN connection, not even behind lead walls :-)

And as for why I don’t simply move to another place with faster internet: 1) I have, this is an epilogue to my experience and 2) it’s not like I lived that far from civilization, it was less than 2 miles from downtown. I would be willing to pay a premium for internet access, but in the current competition model, operators simply don’t care about customers in less dense areas. This is in Brazil, but I’m sure lots of Americans can relate to my story.

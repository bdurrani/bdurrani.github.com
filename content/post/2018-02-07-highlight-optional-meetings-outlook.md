---
title: "Highlight optional meetings in outlook"
date: 2018-02-07
tags: ["outlook"]
---

I find that Outlook calendar (I currently use the 2016 version) doesn't highlight 
optional meeting invites very well. You have to read the fine print of the invite
or dig around in the Scheduling assistant in the Calendar view.

One trick I found on [stack overflow](https://superuser.com/questions/1032432/how-do-i-know-in-an-accepted-a-outlook-meeting-request-that-i-am-required-or-opt)
lets you change the color of any meetings on the calendar 
where you were an optional invite.

- Right click on the calendar
- Select **View Settings**
- Select **Conditional formatting**
- Add a new rule, give it a name and a colour of choice
- Select **Condition** and then **Advanced**
- Click **Field**, select **All appointment fields** and then **Optional attendees**
- Set the condition to **includes** and the value to your name.
 
Optional appointments should then show up in a different colour. 

I made mine red. I don't attend optional meetings :)
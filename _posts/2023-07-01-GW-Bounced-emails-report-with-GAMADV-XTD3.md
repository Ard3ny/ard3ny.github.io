---
title: GW Bounced emails report with GAMADV-XTD3
date: 2023-07-01 11:00:00 +0100
categories: [Sysadmin]
tags: [sysadmin, gamadv-xtd3, google-workspace, bash]
math: false
mermaid: false
---

> TLDR: I've created a bash script which, will send you daily/weekly/whatever email reports about all bounced emails in your Google Workspace using GAMADV-XTD3.
{: .prompt-info }

## How does it work?
If you are not familiar with GAMADV-XTD3, you can check my [previous post](https://thetechcorner.sk/posts/Upgrade-GAMADV-XTD3-from-GAM/).

Basically after installing and authorizing GAMADV-XTD3 in your GW, you can access some of the Google Workspace Admin features using [Google API](https://developers.google.com/workspace)

One of that features is tracking emails with specific attributes such as: 
* FROM whom was the email sent
* TO whom was the email sent
* WHEN 
* ...

### When does the email bounce?
A bounced email can be either a hard bounce or a soft bounce. A hard bounce means that the email address is permanently unavailable and should not receive electronic mail. A soft bounce is temporary and may be caused by server outages or a full inbox.

When the email bounces in the Gmail you automatically receive a notification email back about the email being bounced. This mail is always sent from the same address: mailer-daemon@googlemail.com

We are going to use this feature and automate it with the script.


## How to?
Create and copy the following script to the location of your preference

```bash
vim /root/scripts/bounced_mails_report.sh
```

```bash
#!/bin/bash

#For the email filter
#-------------------------------------------------------------------------------------
today=$(date -d "today" +"%d/%m/%Y")
yesterday=$(date -d "1 days ago" +"%d/%m/%Y")
weekago=$(date -d "7 days ago" +"%d/%m/%Y")
mail_filter_from="mailer-daemon@googlemail.com"

#For email report message
#-------------------------------------------------------------------------------------
sender="UserReportFrom@domain.test"
recipient="UserReportTo@domain.test"                                                                                                                                                           
cc="CcReportTo@domain.test"
subject="Bounced emails from $yesterday"
message="CSV report in the attachment"
attachment="/tmp/output.csv"


/root/bin/gamadv-xtd3/gam all users print messages query "from:$mail_filter_from after:$weekago before:$today" headers User,threadId,id,Date,Subject,From,To,Delivered-To,Content-Type,Message-ID,List-ID > $attachment

/root/bin/gamadv-xtd3/gam user "$sender" sendemail recipient "$recipient" cc "$cc" subject "$subject" message "$message" attach $attachment
```
###  Change all of the values according to your needs

### Add the script execute privileges 
```bash
chmod +x /root/scripts/bounced_mails_report.sh
```

> Run the script manually and check if everything is working as intended.
{: .prompt-info }

## Add the script to the crontab
Open crontab from the terminal

```bash
crontab -e
```

Change the frequency according to your needs with the [crontab guru](https://crontab.guru/)

```bash
0 9 * * * /root/scripts/bounced_mails_report.sh >/dev/null 2>&1 
```
> This is going to run every day at 9am and discard all of the output so crontab wont send any emails.
{: .prompt-info }


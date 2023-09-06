---
title: GW Manage signatures off all users in a workspace with GAMADV-XTD3
date: 2023-09-05 12:00:00 +0100
categories: [Sysadmin]
tags: [sysadmin, gamadv-xtd3, google-workspace]
math: false
mermaid: false
---
## Who is this for?
For GW admins, who would like to automatically manage all of your stuff signatures in Google Workspace, without paying for some obscure 3rd party tool.

## How is this done?
With open-source tool GAMADV-XTD3.

## If you don't know what GAM and GAMADV-XTD3
Check out my [previous post](https://blog.thetechcorner.sk/posts/Upgrade-GAMADV-XTD3-from-GAM/) where I explained everything about GAMADV-XTD3 

## Disclaimer
> This was already done by some gentleman whos name I wasnt able to find. The tutorial was just a little old, confusing and few things needed to be changed. 
[Original post](https://docs.google.com/document/d/1hPOsuh5_45ysdDkFVxjITpDomY7GOWUSpeg3AuUOxqo/view)
{: .prompt-info }


# How to
## First we need to create a custom signature template
You can either create your own or download it. There are plenty of sites just for this.

I'll be using mine
```bash
vim signature_template
```
```bash
<div dir="ltr">
  <div style="margin: 8px 0px 0px; padding: 0px; color: rgb(34, 34, 34); font-family: &quot;Google Sans&quot;, Roboto, RobotoDraft, Helvetica, Arial, sans-serif;">
    <div>
      <div dir="ltr">
        <font color="#888888">
          <div dir="ltr">
            <div dir="ltr">
              <div style="margin: 8px 0px 0px; padding: 0px; color: rgb(34, 34, 34); font-family: &quot;Google Sans&quot;, Roboto, RobotoDraft, Helvetica, Arial, sans-serif;">
                <div>
                  <div dir="ltr">
                    <font color="#888888">
                      <div dir="ltr">
                        <div dir="ltr">
                          <table cellspacing="0" cellpadding="0" border="0" style="background: none; border: 0px; margin: 0px; padding: 0px;">
                            <tbody>
                              <tr>
                                <td style="padding: 0px 0px 0px 12px;">
                                  <table cellspacing="0" cellpadding="0" border="0" style="background: none; border: 0px; margin: 0px; padding: 0px;">
                                    <tbody>
                                      <tr>
                                        <td colspan="2" style="padding-bottom: 5px; color: rgb(149, 183, 4); font-size: 18px;">{first} {last}</td>
                                      </tr>
                                      <tr>
                                        <td colspan="2" style="color: rgb(0, 0, 0); font-size: 14px;"><i>{RT}{title}{/RT}</i></td>
                                      </tr>
                                      <tr>
                                        <td colspan="2" style="color: rgb(0, 0, 0); font-size: 14px;"><strong>bigCompany s.r.o.</strong></td>
                                      </tr>
                                      <tr>
                                        <td width="20" valign="top" style="vertical-align: top; width: 20px; color: rgb(149, 183, 4); font-size: 14px;">p:</td>
                                        <td valign="top" style="vertical-align: top; color: rgb(0, 0, 0); font-size: 14px;">{RT}{phone}{/RT}</td>
                                      </tr>
                                      <tr>
                                        <td width="20" valign="top" style="vertical-align: top; width: 20px; color: rgb(149, 183, 4); font-size: 14px;">a:</td>
                                        <td valign="top" style="vertical-align: top; color: rgb(0, 0, 0); font-size: 14px;">Big Street 69, 333 03 NewYork</td>
                                      </tr>
                                      <tr>
                                        <td width="20" valign="top" style="vertical-align: top; width: 20px; color: rgb(149, 183, 4); font-size: 14px;">w:</td>
                                        <td valign="top" style="vertical-align: top; font-size: 14px;">
                                          https://www.example.test/
```
You've maybe noticed multiple variables/tags in the template. 
{first}, {last}, {last}, {title}, {phone}

These are going to be dynamically replaced by the value of your User. So if you are using your own signature template, you need to add these variables.

Adding {RT}{/RT} around the tag makes the variable skipable if user/users don't have it, therefore not leaving an empty ugly space.

### More/less values
You can add some other variables which will be pulled from the user. To list all possible values
```bash
gam user <User Email Address> print allfields
```

## Print list of users and their signatures in specified OU
```bash
gam ou <OU_name> print signature > signatures.csv
```
Don't forget to change "<OU_name>" (without <> symbols)

## Edit the file so only people without signature stays and pull necessary variable values from GW
```bash
gam csvkmd users signatures.csv keyfield User matchfield isPrimary True skipfield signature '.+' print fields name,organizations,ou,phones > Signatures_to_be_updated.csv
```
This is going to take the list of all users and their signatures, leaving only the ones without the signature variable filled out and print them in the wanted format with all the necessary variable values (name,organizations,ou,phones)

If you want to update the signatures of all the people (not only the ones without signatures)
```bash
gam csvkmd users signatures.csv keyfield User matchfield isPrimary True print fields name,organizations,ou,phones > Signatures_to_be_updated.csv
```

## Update signatures of all the people without any signature
```bash
gam csv Signatures_to_be_updated.csv matchfield orgUnitPath "^/<OU_name>.*" gam user ~primaryEmail signature file signature_template html replace first ~name.givenName replace last ~name.familyName replace title ~organizations.0.title replace phone ~phones.0.value
```
Don't forget to change "<OU_name>" (without <> symbols)
If you have added more user variables, add those as well.


# Run the script in the crontab
## Create script
```bash
vim /root/update_signatures.sh
```

```bash
#!/bin/bash
/root/bin/gamadv-xtd3/gam ou <OU_name> print signature > signatures.csv

/root/bin/gamadv-xtd3/gam csvkmd users signatures.csv keyfield User matchfield isPrimary True skipfield signature '.+' print fields name,organizations,ou,phones > Signatures_to_be_updated.csv
#/root/bin/gamadv-xtd3/gam csvkmd users signatures.csv keyfield User matchfield isPrimary True print fields name,organizations,ou,phones > Signatures_to_be_updated.csv

/root/bin/gamadv-xtd3/gam csv Signatures_to_be_updated.csv matchfield orgUnitPath "^/<OU_name>.*" gam user ~primaryEmail signature file signature_template html replace first ~name.givenName replace last ~name.familyName replace title ~organizations.0.title replace phone ~phones.0.value
```
```bash
chmod +x /root/update_signatures.sh
```

## Create a cronjob
Run daily or change value according to your needs

```bash
0 8 * * * /root/update_signatures.sh 2>&1 >> /tmp/signatures_cron.log
```


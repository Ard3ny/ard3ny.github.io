---
title: GW Set signatures for all users with GamADV-XTD3
date: 2023-07-15 11:00:00 +0100
categories: [Sysadmin]
tags: [sysadmin, gamadv-xtd3, google-workspace, bash]
math: false
mermaid: false
---

> TLDR: 
{: .prompt-info }


## Print signatures off all users from OU
```bash
gam ou test_ou print signature > signatures.csv

```


## Leave only people without the signature
Create and copy the following script to the location of your preference

```bash
gam csvkmd users signatures.csv keyfield User matchfield isPrimary True skipfield signature '.+' print fields name,organizations,ou,phones > UsersWithoutSignaturesOU.csv
```



## If you want all people (even the ones with the signature)
```bash
dokncit
```


## Create signature template
You have multiple options here
- You can either use one of the already downloaded signatures from your Users
- Use one of the online signature creator tools
- Ask AI to create you some
- Or if you are good with html, just create it yourself 
- Just use mine


```
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
                                        <td colspan="2" style="color: rgb(0, 0, 0); font-size: 14px;"><i>{title}</i></td>
                                      </tr>
                                      <tr>
                                        <td colspan="2" style="color: rgb(0, 0, 0); font-size: 14px;"><strong>thiss s.r.o.</strong></td>
                                      </tr>
                                      <tr>
                                        <td width="20" valign="top" style="vertical-align: top; width: 20px; color: rgb(149, 183, 4); font-size: 14px;">p:</td>
                                        <td valign="top" style="vertical-align: top; color: rgb(0, 0, 0); font-size: 14px;">{RT}{phone}{/RT}</td>
                                      </tr>
                                      <tr>
                                        <td width="20" valign="top" style="vertical-align: top; width: 20px; color: rgb(149, 183, 4); font-size: 14px;">a:</td>
                                        <td valign="top" style="vertical-align: top; color: rgb(0, 0, 0); font-size: 14px;">Shire, 222 66 Middle Earth</td>
                                      </tr>
                                      <tr>
                                        <td width="20" valign="top" style="vertical-align: top; width: 20px; color: rgb(149, 183, 4); font-size: 14px;">w:</td>
                                        <td valign="top" style="vertical-align: top; font-size: 14px;">
                                          https://www.thetechcorner.sk/
```


## Update signature of specified Users 

```bash
gam csv UsersWithoutSignaturesOU.csv matchfield orgUnitPath "^/test_ou.*" gam user ~primaryEmail signature file signature_template html replace first ~name.givenName replace last ~name.familyName replace title ~organizations.0.title replace phone ~phones.0.value
```
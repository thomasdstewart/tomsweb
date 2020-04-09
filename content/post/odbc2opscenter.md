---
title: "ODBC 2 OpsCenter"
summary: ""
authors: []
tags: ["blog", "linux", "backup", "netbackup"]
categories: []
date: 2010-09-16 21:50:00
---
OpsCenter is a Symantec product that makes reports for various backup products including Symantec NetBackup. I recently decided that I wanted to improve the reporting of our backups so looked into paying with OpsCenter. It turns out that it's not actually that configurable which is a shame as it's mostly quite good.

It does however use a standard RDMS that appears to be an OEMed version of Sybase's SQL Anywhere to store the data. I thought, great now I'll be able use my own tools to query and report on the back. I want to make pretty pie charts, report on total data transfered and as well as actual important thinks like jobs that have failed.

I did a bit of light goggling without much success, however I was convinced that I could configure an ODBC driver on windows to talk to the database so that Crystal Reports could create said pir charts!

I then remembered, we pay quite a substantial amount of money to Symantec for support. I thought, amassing I'll just ask them, they can tell me how to configure the ODBC driver and save me a bunch of time figuring it out! Roll forward one months of emails (they don't seem to like the phone), and I got this conclusion

```
On 2010-09-13 06:21, enterprise_support@symantec.com wrote:
> Further to our discussion regarding this case, it has been confirmed
> by backline that we can only provide you the ODBC drivers for the
> OpsCenter(As ****** has already sent to you), however unfortunately we
> cannot support or provide the technical know how regarding integrating
> OpsCenter database with any third party frond end reporting software
> like Crystal Reports.
```

So much for Support! The age old saying, if you want something done, do it your self, thus I have documented ODBC2OpsCenter for any one else out there that needs it! I really wish I could use something more free to do all this stuff.

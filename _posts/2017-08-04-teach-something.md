---
layout: post
title:  "Teach Something"
date:   2017-08-07 12:00:00 +0000
categories: powershell
---
So, I saw a curious sight today at work. I saw a colleague with a PowerShell window open, and an Excel spreadsheet - with the contents of said spreadsheet being something like this.

| Account | First Name | Last Name | Command |
| ---- | ---- | ---- | ---- |
| example1@contoso.com | Example | User | Get-ADUser example1@contoso.com \| Set-ADUser -DisplayName "Example User" |
| example2@contoso.com | Example2 | User2 | Get-ADUser example2@contoso.com \| Set-ADUser -DisplayName "Example2 User2" |
....

And so on, for a few hundred rows - with the contents of that last column of course all being generated with the CONCATENATE function. And first name and last name being derived manually.

In this, I recognized something I'd done in earlier life - Excel is not the thing you're meant to do this kind of thing in, but it was what I was taught in school, and it was "easy". (I am not sure what modern kids are being taught in school, but I'm pretty sure it leans more towards the "developer" side than the "ops" side). Excel is a tool, just like PowerShell is - and sometimes, it's about the tool you know.

Now, the code they had written in the command column wouldn't work (they had `-LDAPFilter` on both the Get- and Set- side of the equation for starters), so I took five minutes and wrote up a simple script. This isn't necessarily the shortest way of writing this code (or the best performing, or the one I'd necessarily use - I'd use something shorter if I was writing it at the console, or something which tests against the fail state below if it was to be part of a longer script), but being simple to understand was more important in this case.

``` powershell
Import-Module ActiveDirectory #Even though this isn't necessary on newer versions of PowerShell, it is best practice.

$users = Get-Content users.txt

foreach ($user in $users) {
    $ADObject = Get-ADUser -Filter {UserPrincipalName -eq $user}
    $First = $ADObject.GivenName
    $Last = $ADObject.Surname
    $ADObject | Set-ADUser -DisplayName "$first $last"
}

```

(That first `$ADObject` line is the third revision; the first one tried `Get-ADUser $user`, the second was the same as the third but with `"$user"` instead which apparently PowerShell v4 doesn't like).

Now, of course, I didn't include the -PassThru parameter, so no output. So we checked a few manually, and they seemed fine, but my colleague pointed out the potential fail state of a user not having the GivenName or Surname filled in and so getting a display name of " ". So I wrote a quick one liner so we could see everything:

```powershell
$users | foreach {Get-ADUser -filter {UserPrincipalName -eq $_} -Properties DisplayName | select DisplayName,GivenName,Surname,UserPrincipalName}
```

And sure enough, there were about 5 or 6 that hadn't had the name filled in, and they did that manually. Hours of busywork for the apprentice, reduced to half an hour or less - and crucialy, it got them interested in learning more about PowerShell.

The moral of this story is a pretty simple one - you don't need to be the absolute best at a topic to go ahead and teach someone something - especially if it's going to save them a whole lot of time and effort in the long run.
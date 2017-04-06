---
layout: post
title:  "Putting the Powers in PowerShell"
date:   2016-12-07 12:00:00 +0000
categories: powershell
---

So, up until this week, one of my defencencies in PowerShell was using *powers of numbers*. Finding that the `^` operator did not work, when I needed to square numbers, I would do things like the below:

    $variable * $variable

For anything more complex, it'd inevitably be back to my old friend Excel.

----

To give the answer away early, the answer is in PowerShell's `[math]` type. We can get a list of methods available in this type by using `[math] | Get-Member -Static`

```PowerShell
PS C:\> [math] | Get-Member -Static


TypeName: System.Math

Name            MemberType Definition
----            ---------- ----------
Abs             Method     static sbyte Abs(sbyte value), static int16 Abs(int16 value), static int Abs(int value), ...
Acos            Method     static double Acos(double d)
Asin            Method     static double Asin(double d)
Atan            Method     static double Atan(double d)
...
```

Specifically, the answer lies in the `[math]::Pow(x,y)` method - translating to the standard x<sup>y</sup> notation.

```PowerShell
PS C:\> [math]::pow(5,3)
125
```

Simple. That's all you need to do. If you want to see what got me to learn this, read on.

----

Like a lot of PowerShell's more under-the-hood features, I finally found the answer for this when trying to do something silly with a video game.

## The Scenario

The latest Pokémon games - *Pokémon Sun & Moon* - contain a feature known as "Poké Pelago", which allows you to train your Pokémon when not playing the game - at a rate of approximately 300 experience points per 30 minutes (which can be sped up to 15 minutes with in-game items). 

I have a Pokémon in the "Slow" [experience group] []. They are currently at level 40 (`$CurrentLevel`). At level 80 (`$TargetLevel`), they learn a useful move. There are ways to make them learn this move earlier, but how long would it take to use Poké Pelago to go from `$CurrentLevel` to `$TargetLevel`?

## Tasks

1. Find the amount of experience points that correspond with `$CurrentLevel` (`$CurrentExp`).
2. Find the amount of experience points that correspond with `$TargetLevel` (`$TargetExp`).
3. Subtract `$CurrentExp` from $TargetExp, divide the result by 300 to get the number of Poké Pelago sessions. (`$Sessions`)
4. Use `New-TimeSpan` to translate `$Sessions` into the amount of time it will take.

1 and 2 on this list are repeated work, and so should be made into a function. 

The formula for calculating "Slow" experience points is given in the Bulbapedia article above - being 5*n*<sup>3</sup>, where *n* is the Pokémon's level.

## The Script

```PowerShell
function Get-PokémonSlowExperiencePoints {
    param(
        [int]$Level
    )
    5 * [math]::pow($level,3)
}
```

> (side note: I'm cutting some corners here, but if you're writing functions for a (primarily) English language audience, you should always include an alias if you want to use a character like `é` in your function name. Otherwise, someone is going to type `Get-Poke` and wonder why tab completion doesn't work).

With the function above defined, the rest of the script becomes very easy to write:

``` PowerShell
$CurrentLevel = 40
$TargetLevel = 80

$CurrentExp = Get-PokémonSlowExperiencePoints -Level $CurrentLevel
$TargetExp = Get-PokémonSlowExperiencePoints -Level $TargetLevel
$Sessions = ($TargetExp - $CurrentExp) / 300

Write-Warning "Without Poké Beans (1 session = 30 minutes)"
New-TimeSpan -Minutes (30 * $Sessions)

Write-Warning "With Poké Beans (1 session = 15 minutes)"
New-TimeSpan -Minutes (15 * $Sessions)
```

Which gives the output below:

```PowerShell
WARNING: Without Poké Beans (1 session = 30 minutes)


Days              : 155
Hours             : 13
Minutes           : 20
Seconds           : 0
Milliseconds      : 0
Ticks             : 134400000000000
TotalDays         : 155.555555555556
TotalHours        : 3733.33333333333
TotalMinutes      : 224000
TotalSeconds      : 13440000
TotalMilliseconds : 13440000000

WARNING: With Poké Beans (1 session = 15 minutes)
Days              : 77
Hours             : 18
Minutes           : 40
Seconds           : 0
Milliseconds      : 0
Ticks             : 67200000000000
TotalDays         : 77.7777777777778
TotalHours        : 1866.66666666667
TotalMinutes      : 112000
TotalSeconds      : 6720000
TotalMilliseconds : 6720000000
```

So yes, it is not worth training my Pokémon in this way - particularly as the game only allows you to queue up 99 Poké Pelago sessions at once, and it would take 7466.66667.

If I wanted to, I could turn this script into a function, and/or write a more advanced helper function that accounts for the other kind of experience groups and perhaps takes pipeline input. Doing these is left as an exercise to the reader - this was just a quick script to demonstrate to myself that it would not be worth trying to use Poké Pelago in this way.

*Since initially writing this article, an exploit has been discovered in the game that causes Poké Pelago sessions to complete instantly, thus making the venture more worthwhile.*

[experience group]: http://bulbapedia.bulbagarden.net/wiki/Experience
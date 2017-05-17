---
layout: post
title: Date arithmetic is scary 
date: 2017-05-15 19:29 +0200
---
I don't need to tell you that. If you ever watched [Computerphile](https://www.youtube.com/channel/UC9-y-6csu5WGm29I7JiwpnA) explain the [horror that lurks behind dealing with timezones](https://www.youtube.com/watch?v=-5wpm-gesOY), chances are you got the point.

So, hopefully, you use a library to take care of your Date/Time needs. If you are a Java programmer, this will most likely be JodaTime or the Java 8 DateTime API. Even so, you still have quite enough rope in hand to hang yourself. Consider the following example (which uses the ``java.time`` library, but applies to JodaTime in the same way):

```java
package de.elang.datetime;

import org.junit.jupiter.api.Test;

import java.time.Period;
import java.time.ZoneId;
import java.time.ZonedDateTime;

import static java.time.ZoneOffset.UTC;
import static org.junit.jupiter.api.Assertions.assertEquals;

/**
 * Created by eike on 14.05.17.
 */
class DateTimeBehaviourFuckery {

    @Test
    void addingTheSamePeriodToTwoRepresentationsOfTheSameTimeWillResultInIdenticalInstants() {
        final ZonedDateTime centralEuropeanTimeBeforeDST = ZonedDateTime.of(2017, 3, 20, 11, 27, 33, 0, ZoneId.of("CET"));
        ZonedDateTime centralEuropeanTimeInDST = centralEuropeanTimeBeforeDST.plus(Period.ofMonths(3));

        final ZonedDateTime greenwichTime = centralEuropeanTimeBeforeDST.withZoneSameInstant(UTC);
        final ZonedDateTime greenwichLater = greenwichTime.plus(Period.ofMonths(3));

        assertEquals(centralEuropeanTimeInDST.toInstant().toEpochMilli(), greenwichLater.toInstant().toEpochMilli());
    }
}
```

If, like me, you naïvely assumed this test to succeed, do read on. Otherwise, feel free to gloat or just consider yourself lucky you dodged that bullet.

So, unexepectedly, this test **fails** with a time difference of 3600000 milliseconds (i.e. one hour) between the two dates. What happened?

In adding three months to the date of March 20, 2017 we inadvertedly crossed the point in time at which daylight savings time comes into effect, i.e. on Sunday, March 26, 2017 the clock went from 02:00 in the morning to 03:00 in the morning. Adding three months to our initial date in CET brings us from ``2017-03-20T11:27:33+01:00[CET]`` to ``2017-06-20T11:27:33+02:00[CET]``, which is of course ``2017-06-20T09:27:33Z``.

UTC does not have the notion of daylight savings time, so we go straight from ``2017-03-20T10:27:33Z`` to ``2017-06-20T10:27:33Z`` without saving that extra hour, so our three months are actually an hour longer in this case.

# What to do?
There is no absolute answer to that question, you will have to decide on a solution for yourself. 

I am assuming that:

* your are storing your dates as "milliseconds since the epoch" in your database
* you don't want to duplicate any of the efforts that have gone into the Java8 DateTime library or JodaTime and therefore want to deserialize your DB entries into proper objects in order to do date calculations.
* you don't have the luxury of just doing everything in UTC and presenting it as such
* the result of the calculation is something you want to store (e.g. a due date, reminder, etc.)

Based on those assumptions you can:
* Convert all dates to UTC as soon as possible and convert them into a local timezone at the last moment, in which case any period spanning a DST gap could be an hour too long (northern hemisphere) or too short (southern hemisphere) compared to a calculation in local time.
* Do all calculations in local time, which makes them valid only in the context of their timezone. 
* Work around the issue with custom code that addresses your specific need.

# Conclusion
Date calcuations can be nasty even when using mature date/time libraries. Be mindful of these pitfalls and address them in a way that best fits your needs.

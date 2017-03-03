# Cronos

## Build status

**Windows** | **Linux / OS X** 
--- | ---
[![Build status](https://ci.appveyor.com/api/projects/status/v6cq97gdg87j7utl/branch/master?svg=true)](https://ci.appveyor.com/project/odinserj/cronos/branch/master) | [![Build Status](https://travis-ci.org/HangfireIO/Cronos.svg?branch=master)](https://travis-ci.org/HangfireIO/Cronos)

## Features

* Parse cron expressions comprising five or six fields. See [Cron format](#cron-format).
* Calculate next execution in the **local time** within given time zone.
* Support extended format with non-standard characters: `?`, `L`, `W`, `#`.
* Handle the transition from standard time to **daylight saving time** and vice versa.

## Installation

```
PM> Install-Package Cronos
```

## Usage

### Next execution based on 6 fields expression

```csharp
var expression = CronExpression.Parse("0 30 * * * *");
var easternTimeZone = TimeZoneInfo.FindSystemTimeZoneById("Eastern Standard Time");

var nextTime = expression.Next(DateTimeOffset.Now, DateTimeOffset.MaxValue, easternTimeZone));
```

### Next execution based on 5 fields expression

```csharp
var expression = CronExpression.Parse("30 * * * *");
var easternTimeZone = TimeZoneInfo.FindSystemTimeZoneById("Eastern Standard Time");

var nextTime = expression.Next(DateTimeOffset.Now, DateTimeOffset.MaxValue, easternTimeZone));
```

### Next execution in UTC time

```csharp
var expression = CronExpression.Parse("* * * * *");

var nextTime = expression.Next(DateTimeOffset.Now, DateTimeOffset.MaxValue, TimeZoneInfo.Utc);
```

### Next Friday the thirteenth

```csharp
var expression = CronExpression.Parse("0 0 13 * FRI");

var nextTime = expression.Next(DateTimeOffset.Now, DateTimeOffset.MaxValue, TimeZoneInfo.Utc);
```

### Daylight Saving Time

**Cronos** handles the transition from standard time (ST) to Daylight saving time (DST). 

**Setting the clocks forward**

If next execution falls on invalid time when the clocks jump forward then next execution will shift to next valid time. See example:

```csharp
var expression = CronExpression.Parse("0 30 2 * * *");
var easternTimeZone = TimeZoneInfo.FindSystemTimeZoneById("Eastern Standard Time");

// 2016-03-13 - the day when DST starts in Eastern time zone. The clocks jump from 1:59 am ST to 3:00 am DST. 
// So duration from 2:00 am to 2:59 am is invalid.

var startTime = new DateTimeOffset(2016, 03, 13, 01, 50, 00, easternTimeZone.BaseUtcOffset);

// Should be scheduled to 2:30 am ST but that time is invalid. Next valid time is 3:00 am DST.
var nextTime = expression.Next(startTime, DateTimeOffset.MaxValue, easternTimeZone);

Console.WriteLine("Next execution at " + nextTime);

// Next execution at 2016-03-13 03:00:00 AM -04:00
```

**Setting the clocks backward**

When DST ends you set the clocks backward so you have duration which repeats twice. If you are in USA the duration was e.g. 2016/11/06 from 1:00 am to 1:59 am. If next execution falls on this duration behavior depends on cron expression:

* Cron expression describes certain time of a day, e.g. `"0 30 1 * * ?"` - 1:30 am every day, or `"0 0,45 1,2 * * ?"` - 1:00 am, 1:45 am, 2:00 am, 2:45 am every day. In this case each cron job will be scheduled only before clock shifts. Reason is when you describe certain time of day you mean that it should be scheduled once a day regardless whether there is clock shifts in that day.

    ```csharp
var expression = CronExpression.Parse("0 30 1 * * ?");
var easternTimeZone = TimeZoneInfo.FindSystemTimeZoneById("Eastern Standard Time");

var startTime = new DateTime(2016, 11, 06, 00, 59, 00);
var startDateTimeOffset = new DateTimeOffset(startTime, easternTimeZone.GetUtcOffset(startTime));

var nextTime = expression.Next(startDateTimeOffset, DateTimeOffset.MaxValue, easternTimeZone);
Console.WriteLine("Next execution at " + nextTime);

nextTime = expression.Next(nextTime?.AddSeconds(1));
Console.WriteLine("Next execution at " + nextTime);

// Next execution at 2016-03-13 01:30:00 AM -04:00
// Next execution at 2016-03-13 02:30:00 AM -05:00
    ```

* Cron expression describes secondly, minutely or hourly job, e.g. `"0 30 * * * ?"`, `"0 * 1 * * ?"`, `"0,5 */10 * * * ?"`. In this case each cron job will be scheduled before and after clock shifts.

    ```csharp
var expression = CronExpression.Parse("0 30 * * * ?");
var easternTimeZone = TimeZoneInfo.FindSystemTimeZoneById("Eastern Standard Time");

var startTime = new DateTime(2016, 11, 06, 00, 59, 00);
var startDateTimeOffset = new DateTimeOffset(startTime, easternTimeZone.GetUtcOffset(startTime));

var nextTime = expression.Next(startDateTimeOffset, DateTimeOffset.MaxValue, easternTimeZone);
Console.WriteLine("Next execution at " + nextTime);

nextTime = expression.Next(nextTime?.AddSeconds(1));
Console.WriteLine("Next execution at " + nextTime);

nextTime = expression.Next(nextTime?.AddSeconds(1));
Console.WriteLine("Next execution at " + nextTime);

// Next execution at 2016-11-06 01:30:00 AM -04:00
// Next execution at 2016-11-06 01:30:00 AM -05:00
// Next execution at 2016-11-06 02:30:00 AM -05:00
    ```

## Cron format

**Cronos** uses a cron expression comprising five or six fields separated by white space that represents a set of times.

                                             Allowed values     Allowed special characters     Comment
												                                           
    ┌───────────── second (optional)         0-59               * , - /                    
    │ ┌───────────── minute                  0-59               * , - /                    
    │ │ ┌───────────── hour                  0-23               * , - /                    
    │ │ │ ┌───────────── day of month        1-31               * , - / L W ?              
    │ │ │ │ ┌───────────── month             1-12 or JAN-DEC    * , - /                    
    │ │ │ │ │ ┌───────────── day of week     0-7  or MON-SUN    * , - / # L ?                  0 and 7 means SUN
    │ │ │ │ │ │
    │ │ │ │ │ │
    │ │ │ │ │ │
    * * * * * *

**Star `*`**

`*` means any value. Used to select all values within a field. For example, `*` in the hour field means "every hour":

| Expression    | Description                          |
|---------------|--------------------------------------|
| `* * * * * *` | Every second                         |
| `* * * * *`   | Every minute                         |
| `30 3 * * *`  | At 3:30 AM every day                 |
| `0  0 1 * *`  | At midnight, on day 1 of every month |

**Comma `,`**

Commas are used to separate items of a list.

| Expression        | Description                           |
|-------------------|---------------------------------------|
| `15,45 * * * * *` | Every minute at 15 and 45 seconds     |
| `* * * * SAT,SUN` | Every minute on saturdays and sundays |
| `* * * * 6,7`     | Every minute on saturdays and sundays |
| `* * * * 0,6`     | Every minute on saturdays and sundays |

**Hyphens `-`**

Hyphens define ranges. 

| Expression        | Description                                                       |
|-------------------|-------------------------------------------------------------------|
| `0-30 1 * * *`    | Every minute between 01:00 AM and 01:30 AM                        |
| `45-15 1 * * *`   | Every minute from 1:00 AM to 01:15 AM and from 1:45 AM to 1:59 AM |
| `0 0 * * MON-FRI` | At 00:00, Monday through Friday                                   |

**L character**

`L` stands for "last". When used in the day-of-week field, it allows you to specify constructs such as "the last Friday" (`5L`) of a given month. In the day-of-month field, it specifies the last day of the month.

| Expression    | Description                                          |
|---------------|------------------------------------------------------|
| `0 0 L * *`   | At 00:00 AM on the last day of the month             |
| `0 0 L-1 * *` | At 00:00 AM the day before the last day of the month |
| `0 0 * * 1L`  | At 00:00 AM on the last monday of the month          |

**W character**

`W` character is allowed for the day-of-month field. This character is used to specify the weekday (Monday-Friday) nearest the given day. As an example, if you were to specify `15W` as the value for the day-of-month field, the meaning is: "the nearest weekday to the 15th of the month." So, if the 15th is a Saturday, `Next` returns Friday the 14th. If the 15th is a Sunday, `Next` returns Monday the 16th. If the 15th is a Tuesday, then `Next` returns Tuesday the 15th. However, if you specify "1W" as the value for day-of-month, and the 1st is a Saturday, `Next` returns the 3rd, as it does not 'jump' over the boundary of a month's days. The 'W' character can be specified only when the day-of-month is a single day, not a range or list of days.

| Expression        | Description                                              |
|-------------------|----------------------------------------------------------|
| `0 0 1W * *`      | At 00:00 AM, on the first weekday of every month         |
| `0 0 10W * *`     | At 00:00 AM on the weekday nearest day 10 of every month |
| `0 0 LW * *`      | At 00:00, on the last weekday of the month               |

**Hash `#`**

`#` is allowed for the day-of-week field, and must be followed by a number between one and five. It allows you to specify constructs such as "the second Friday" of a given month. 

| Expression        | Description                                              |
|-------------------|----------------------------------------------------------|
| `0 0 * * 6#3`     | At 00:00 AM on the third Friday of the month             |
| `0 0 * * 1#1`     | At 00:00 AM on the first Monday of the month             |
| `0 0 * 1 1#1`     | At 00:00 AM on the first Monday of the January           |

**Question mark `?`**

`?` is "no specific value" and a synonym of `*`. It's supported but **non-obligatory**. `0 0 5 * *` is the same as `0 0 5 * ?`. You can specify `?` only in one field. For example, `* * ? * ?` is wrong expression.

| Expression    | Description                          |
|---------------|--------------------------------------|
| `* * * * * ?` | Every second                         |
| `* * * ? * *` | Every second                         |
| `* * * * ?`   | Every minute                         |
| `* * ? * *`   | Every minute                         |
| `0  0 1 * ?`  | At midnight, on day 1 of every month |
| `0  0 ? * 1`  | At midnight every Monday             |

**Slash `/`**

Slashes can be combined with ranges to specify step values. 

| Expression        | Description                                                                                |
|-------------------|--------------------------------------------------------------------------------------------|
| `*/5 * * * * *`   | Every 5 seconds                                                                            |
| `0 1/5 * * *`     | Every 5 hours, starting at 01:00                                                           |
| `*/30 */6 * * *`  | Every 30 minutes, every 6 hours: at 00:00, 00:30, 06:00, 06:30, 12:00, 12:30, 18:00, 18:30 |
| `0 0  15/2 * *`   | At 00:00, every 2 days, starting on day 15 of the month                                    |
| `0 0 * 2/3 *`     | At 00:00, every 3 months, February through December                                        |
| `0 0 * * 1/2`     | At 00:00, every 2 days of the week, starting on Monday                                     |

**Specify Day of month and Day of week**

You can specify both Day of month and Day of week, it allows you to specify constructs such as "Friday the thirteenth". 

| Expression        | Description                                                                                |
|-------------------|--------------------------------------------------------------------------------------------|
| `0 0 13 * 5`      | At 00:00, Friday the thirteenth                                                            |
| `0 0 13 2 5`      | At 00:00, Friday the thirteenth, only in February                                          |


## License

Cronos is under the [Apache License 2.0][Apache-2.0].

[Apache-2.0]:LICENSE

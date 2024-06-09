+++
title = '# 1. Exploring Go Generics | Building a scheduler service in Go'
date = 2024-04-02T11:42:30+05:30
+++

**_How confident are you with Go Generics?_** - This question took me by surprise, prompting me to acknowledge that I haven't fully tapped into Go's features like generics in my daily tasks. That's why I'm diving into Go Generics now, starting with a hands-on project: building a Scheduler service capable of handling scheduled tasks like making HTTP calls etc (Lets keep is simple as of now ). This project is also a chance for me to try out new packages and follow test driven development.

## But why a scheduler ? theres lot of options already.

I have been using services like [Pipedream](pipedream.com) to create journeys/automations which helps me to schedule HTTP calls, and it works absolutely fine. What I miss in it is the ability to schedule a skip for any automation, for example I wish that my scheduled curl call could be skipped every alternate saturday, or maybe skip the automation on a particular date. Last but not the least, free trial is going to expire soon, and I would prefer to pay $XX for my own ec2 which I could use for many other things insted of just one service.

Below are the features that I wanted my scheduler to have:

- Configure HTTP calls to be scheduled to run every n `seconds/minutes/hours/days/weeks/months/year`.
- Configure using `crontab`.
- Configure `skipping` of schedule on particular conditions.
- Configure same http call to multiple endpoints.
- Configure retries on particular status code responses.
- Configure alerts on success/failure etc.

## Mapping Out the Project: Libraries and Approaches

Now that I have decided what to build, lets think about how. To keep things simple I am going to use [DuckDB](https://duckdb.org/docs/) ,I had my eye on it for a while but didnt get a chance to use it. Ideally I would use [Viper](https://github.com/spf13/viper) for config management but since this is about exploration, I have decided to give a shot to [Koanf](https://github.com/knadh/koanf) which is much more light weight then Viper.

Predefined cron expressions
(Copied from https://en.wikipedia.org/wiki/Cron#Predefined_scheduling_definitions, with text modified according to this implementation)

```
Entry Description Equivalent to
@annually Run once a year at midnight in the morning of January 1 0 0 0 1 1 \* _
@yearly Run once a year at midnight in the morning of January 1 0 0 0 1 1 _ _
@monthly Run once a month at midnight in the morning of the first of the month 0 0 0 1 _ \* _
@weekly Run once a week at midnight in the morning of Sunday 0 0 0 _ _ 0 _
@daily Run once a day at midnight 0 0 0 \* \* \* _
@hourly Run once an hour at the beginning of the hour 0 0 _ \* \* \* \*
@reboot Not supported
- Show next cron job and time
```

Breakdown of schedule string. (ISO 8601 Notation)
Example schedule string:

R2/2017-06-04T19:25:16.828696-07:00/PT10S
This string can be split into three parts:

Number of times to repeat/Start Datetime/Interval Between Runs
Number of times to repeat
This is designated with a number, prefixed with an R. Leave out the number if it should repeat forever.

Examples:

R - Will repeat forever
R1 - Will repeat once
R231 - Will repeat 231 times.
Start Datetime
This is the datetime for the first time the job should run.

Kala will return an error if the start datetime has already passed.

Examples:

2017-06-04T19:25:16
2017-06-04T19:25:16.828696
2017-06-04T19:25:16.828696-07:00
2017-06-04T19:25:16-07:00
To Note: It is recommended to include a timezone within your schedule parameter.

Interval Between Runs
This is defined by the ISO8601 Interval Notation.

It starts with a P, then you can specify years, months, or days, then a T, preceded by hours, minutes, and seconds.

Lets break down a long interval: P1Y2M10DT2H30M15S

P - Starts the notation
1Y - One year
2M - Two months
10D - Ten days
T - Starts the time second
2H - Two hours
30M - Thirty minutes
15S - Fifteen seconds
Now, there is one alternative. You can optionally use just weeks. When you use the week operator, you only get that. An example of using the week operator for an interval of every two weeks is P2W.

Examples:

P1DT1M - Interval of one day and one minute
P1W - Interval of one week
PT1H - Interval of one hour.
More Information on ISO8601
Wikipedia's Article

# partialscrub
Partial scrubbing for btrfs filesystems

Scrubbing big btrfs filesystems can take a long time.
Also, in case of restart, scrubbing will not continue.

Here is an attempt to create a **daily**/hourly executed script that will start/resume btrfs scrubbing based on specified criteria.

## Criteria that control the partial scrub:

* `(-p number)`: specify the percentage of data that you want to scrub on every execution. Default 100% (scrub everything)

* In case we want to ensure that scrub will continue even after restart, we have to execute the script hourly/daily.
In that case, maybe we want to scrub once a month/week but not every day.
`(-f frequency)` solves that problem by specifying after how many seconds the program will start over a finished scrub.
Default value (monthly): 2629744 (average tropical month in seconds)

* `(-d)`: scrub devices of filesystem individually. Useful in raid56 where scrubbing must happen be executed per device, see below.

## General Notes:

* Ideal scenario is to run daily, even when you want to commplete scrub within one go. Daily execution will makse sure that scrub will finish regardless of restarts/shutdown/downtime.

* **When the program is called, it will always resume scrubbing, until it's `-p` criteria is fulfilled.**
* When the program stops, scrubbing will be aborted (if running) and status of scrub will be printed.
* The scrub will run with `-c 3` in order to minimize impact of scrub
* Script will run only one instance at a time iin order to minimize impact of scrub. The instances that were started later, will wait until the first instance is finished. `flock` is used to ensure that.
* GNU/date utility is required, busybox `date` is not enough.

## Common use cases:

Scrub my filesystem monthly and every time try to complete the scrub:

Put this in a daily cron/systemd and when the scrub is finished, scrub will not continue, unless month has roughly* passed:
`partialscrub.sh /filesystem`

Scrub weekly and every time try to complete the scrub:

Put this in a daily cron/systemd and when the scrub is finished, scrub will not continue, unless a week has roughly* passed:
`partialscrub.sh -f 604800 /filesystem`

Scrub monthly, but try to minimize the duration of the scrub each day. (-p 100/30)
`partialscrub.sh -p 4 /filesystem`

Scrub weekly, but try to minimize the duration of the scrub each day. (-p 100/7)

`partialscrub.sh -p 15 -f 604800 /filesystem`

*The program will calculate when the last scrub event happened and react based on that.
If the program finishes on 25 of a month and the frequency -f is monthly, then scrub will start over on 1st of the next month.
If the program finishes on Sunday and the frequency -f is weekly, then scrub will start over on Monday.

The calculation is this: if (now div frequency) - ((last action) div frequency) > 0 then start over scrub.

**Do not use very low `-p`, in order to make sure that the scrub can finish within `-f` period!**

## raid56:

According to: https://lore.kernel.org/linux-btrfs/20200627032414.GX10769@hungrycats.org/#r
raid56 filesystems must be scrubbed individually. Every time that a device scrubbing is finished, the next execution will examine if all devices have been scrubbed.
If this is not the case the next device will be scrubbed.


```
Scrub btrfs data periodically, each time maximum p percentage is scrubbed. When scrub finishes it will not restart, unless f seconds have passed
Usage: /root/bin/partialscrub.sh [-d|--(no-)multiple_devices] [-p|--percentage <arg>] [-f|--frequency <arg>] [-h|--help] [<device>]
        -d, --multiple_devices, --no-multiple_devices: specify to scrub one device at a time, useful for raid5/6 (off by default)
        -p, --percentage: how much data to scrub, default is to scan all (default: '100')
        -f, --frequency: how frequently scrub should be run (default: '2629744') which is the average tropical month in seconds
        -h, --help: Prints help

```

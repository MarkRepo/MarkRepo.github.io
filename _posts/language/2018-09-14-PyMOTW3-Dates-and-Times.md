---
title: PyMOTW-3 --- Dates and Times
categories: language
tags: PyMOTW-3
---

Python不包括日期和时间的原生类型，但有三个模块用于在多个表示中操作日期和时间值。  
time模块从底层C库公开与时间相关的函数。它包括用于检索时钟时间和处理器运行时间的函数，以及基本的解析和字符串格式化工具。  
datetime模块为日期、时间和其组合值提供更高级别的接口。datetime中的类支持算术，比较和时区配置。  
calendar模块创建周、月和年的格式化表示。它还可用于计算重复事件、给定日期是星期几以及其他基于日历的值。

## time — Clock Time

目的：提供用于操作时钟时间的功能。  
time模块提供对几种不同类型时钟的访问，每种时钟都可用于不同目的。标准系统调用如`time（）`报告系统“墙上”时间。`monotonic（）`时钟可用于测量长时间运行进程中经过的时间，因为即使系统时间发生变化，也可保证不会向后移动。对于性能测试，`perf_counter（）`提供对最高可用分辨率的时钟的访问，以使短时间测量更准确。CPU时间可通过`clock（）`获得，`process_time（）`返回处理器时间和系统时间的组合。

>注意  
这些实现公开了C库函数来操作日期和时间。由于它们与底层C实现相关联，因此某些细节（例如，纪元的开始和支持的最大日期值）是特定于平台的。 有关完整的详细信息，请参阅库文档。

### Comparing Clocks

时钟的实现细节因平台而异。使用`get_clock_info（）`访问有关当前实现的基本信息，包括时钟的分辨率。

```python
#time_get_clock_info.py
import textwrap
import time

available_clocks = [
    ('clock', time.clock),
    ('monotonic', time.monotonic),
    ('perf_counter', time.perf_counter),
    ('process_time', time.process_time),
    ('time', time.time),
]

for clock_name, func in available_clocks:
    print(textwrap.dedent('''\
    {name}:
        adjustable    : {info.adjustable}
        implementation: {info.implementation}
        monotonic     : {info.monotonic}
        resolution    : {info.resolution}
        current       : {current}
    ''').format(
        name=clock_name,
        info=time.get_clock_info(clock_name),
        current=func())
    )
```
Mac OS X的此输出显示monotonic和perf_counter时钟使用相同的底层系统调用实现。

```
$ python3 time_get_clock_info.py

clock:
    adjustable    : False
    implementation: clock()
    monotonic     : True
    resolution    : 1e-06
    current       : 0.046796

monotonic:
    adjustable    : False
    implementation: mach_absolute_time()
    monotonic     : True
    resolution    : 1e-09
    current       : 716028.526210432

perf_counter:
    adjustable    : False
    implementation: mach_absolute_time()
    monotonic     : True
    resolution    : 1e-09
    current       : 716028.526241605

process_time:
    adjustable    : False
    implementation: getrusage(RUSAGE_SELF)
    monotonic     : True
    resolution    : 1e-06
    current       : 0.046946999999999996

time:
    adjustable    : True
    implementation: gettimeofday()
    monotonic     : False
    resolution    : 1e-06
    current       : 1521404584.966362
```

### Wall Clock Time

时间模块的核心函数之一是time（），它将自“epoch”开始以来的秒数作为浮点值返回。

```python
#time_time.py
import time

print('The time is:', time.time())
```
epoch（纪元）是时间测量的开始，对于Unix系统来说，它是1970年1月1日的0:00。虽然值总是浮点数，但实际精度与平台有关。

```
$ python3 time_time.py

The time is: 1521404585.0243158
```
浮点表示在存储或比较日期时很有用，但对生成人类可读表示没有用。对于记录或打印时间，ctime（）可能更有用。

```python
#time_ctime.py
import time

print('The time is      :', time.ctime())
later = time.time() + 15
print('15 secs from now :', time.ctime(later))
```
此示例中的第二个print（）调用显示如何使用ctime（）格式化除当前时间以外的时间值。

```
$ python3 time_ctime.py

The time is      : Sun Mar 18 16:23:05 2018
15 secs from now : Sun Mar 18 16:23:20 2018
```

### Monotonic Clocks

因为time（）查看系统时钟，并且用户或系统服务可以更改系统时钟以在多台计算机之间同步时钟，所以重复调用time（）可能会产生前进和后退的值。当尝试测量持续时间或以其他方式使用这些时间进行计算时，这可能导致意外行为。通过使用monotonic（）来避免这些情况，monotonic（）始终返回前进的值。

```python
#time_monotonic.py
import time

start = time.monotonic()
time.sleep(0.1)
end = time.monotonic()
print('start : {:>9.2f}'.format(start))
print('end   : {:>9.2f}'.format(end))
print('span  : {:>9.2f}'.format(end - start))
```
未定义单调时钟的起始点，因此返回值仅对使用其他时钟值进行计算有用。在该示例中，使用monotonic（）来测量睡眠的持续时间。

```
$ python3 time_monotonic.py

start : 716028.72
end   : 716028.82
span  :      0.10
```

### Processor Clock Time

当time（）返回墙上时间时，clock（）返回处理器时钟时间。clock（）返回的值反映了程序运行时使用的实际时间。

```python
#time_clock.py
import hashlib
import time

# Data to use to calculate md5 checksums
data = open(__file__, 'rb').read()

for i in range(5):
    h = hashlib.sha1()
    print(time.ctime(), ': {:0.3f} {:0.3f}'.format(time.time(), time.clock()))
    for i in range(300000):
        h.update(data)
    cksum = h.digest()
```
在此示例中，格式化的ctime（）与time（）的浮点值一起打印，并且每次通过循环迭代打印clock（）。

>注意 
如果要在系统上运行该示例，则可能必须向内部循环添加更多循环，或者使用更大量的数据来实际查看时间差异。

```
$ python3 time_clock.py

Sun Mar 18 16:23:05 2018 : 1521404585.328 0.051
Sun Mar 18 16:23:05 2018 : 1521404585.632 0.349
Sun Mar 18 16:23:05 2018 : 1521404585.935 0.648
Sun Mar 18 16:23:06 2018 : 1521404586.234 0.943
Sun Mar 18 16:23:06 2018 : 1521404586.539 1.244
```
通常，如果程序没有执行任何操作，则处理器时钟不会滴答（tick）。

```python
#time_clock_sleep.py
import time

template = '{} - {:0.2f} - {:0.2f}'

print(template.format(time.ctime(), time.time(), time.clock()) )

for i in range(3, 0, -1):
    print('Sleeping', i)
    time.sleep(i)
    print(template.format(time.ctime(), time.time(), time.clock()) )
```
在这个例子中，循环通过在每次迭代后进入休眠状态来完成很少的工作。即使应用程序处于睡眠状态，time（）值也会增加，但clock（）值不会。

```
$ python3 -u time_clock_sleep.py

Sun Mar 18 16:23:06 2018 - 1521404586.89 - 0.04
Sleeping 3
Sun Mar 18 16:23:09 2018 - 1521404589.90 - 0.04
Sleeping 2
Sun Mar 18 16:23:11 2018 - 1521404591.90 - 0.04
Sleeping 1
Sun Mar 18 16:23:12 2018 - 1521404592.90 - 0.04
```
调用sleep（）会从当前线程让出控制权并要求它等待系统将其唤醒。如果一个程序只有一个线程，这有效地阻塞了应用程序，使它不工作。

### Performance Counter

重要的是具有用于测量性能的高分辨率单调时钟。确定最佳时钟数据源需要特定于平台的知识，Python在perf_counter（）中提供了这些知识。

```python
#time_perf_counter.py
import hashlib
import time

# Data to use to calculate md5 checksums
data = open(__file__, 'rb').read()

loop_start = time.perf_counter()

for i in range(5):
    iter_start = time.perf_counter()
    h = hashlib.sha1()
    for i in range(300000):
        h.update(data)
    cksum = h.digest()
    now = time.perf_counter()
    loop_elapsed = now - loop_start
    iter_elapsed = now - iter_start
    print(time.ctime(), ': {:0.3f} {:0.3f}'.format(iter_elapsed, loop_elapsed))
```
与monotonic（）一样，perf_counter（）的纪元是未定义的，并且这些值用于比较和计算值，而不是绝对时间。

```
$ python3 time_perf_counter.py

Sun Mar 18 16:23:13 2018 : 0.378 0.378
Sun Mar 18 16:23:13 2018 : 0.376 0.754
Sun Mar 18 16:23:14 2018 : 0.372 1.126
Sun Mar 18 16:23:14 2018 : 0.376 1.502
Sun Mar 18 16:23:14 2018 : 0.376 1.879
```

### Time Components

将时间存储为经过的秒数在某些情况下很有用，但有时程序需要访问日期（年，月等）的各个字段。time模块定义`struct_time`用于保存日期和时间值，其中组件已分解，因此易于访问。有几个函数返回`struct_time`值而不是浮点数。

```python
#time_struct.py
import time

def show_struct(s):
    print('  tm_year :', s.tm_year)
    print('  tm_mon  :', s.tm_mon)
    print('  tm_mday :', s.tm_mday)
    print('  tm_hour :', s.tm_hour)
    print('  tm_min  :', s.tm_min)
    print('  tm_sec  :', s.tm_sec)
    print('  tm_wday :', s.tm_wday)
    print('  tm_yday :', s.tm_yday)
    print('  tm_isdst:', s.tm_isdst)

print('gmtime:')
show_struct(time.gmtime())
print('\nlocaltime:')
show_struct(time.localtime())
print('\nmktime:', time.mktime(time.localtime()))
```
gmtime（）函数以UTC格式返回当前时间。localtime（）返回应用的当前时区的当前时间。mktime（）接受struct_time并将其转换为浮点表示。

```
$ python3 time_struct.py

gmtime:
  tm_year : 2018
  tm_mon  : 3
  tm_mday : 18
  tm_hour : 20
  tm_min  : 23
  tm_sec  : 14
  tm_wday : 6
  tm_yday : 77
  tm_isdst: 0

localtime:
  tm_year : 2018
  tm_mon  : 3
  tm_mday : 18
  tm_hour : 16
  tm_min  : 23
  tm_sec  : 14
  tm_wday : 6
  tm_yday : 77
  tm_isdst: 1

mktime: 1521404594.0
```

### Working with Time Zones

确定当前时间的功能取决于是“通过程序还是使用为系统设置的默认时区”设置时区。更改时区不会改变实际时间，只会改变它的表示方式。  
要更改时区，请设置环境变量TZ，然后调用tzset（）。时区可以指定很多细节，直到夏令时的开始和停止时间。但是，通常使用时区名称更容易，并让底层库导出其他信息。  
此示例程序将时区更改为几个不同的值，并显示更改如何影响时间模块中的其他设置。

```python
#time_timezone.py
import time
import os

def show_zone_info():
    print('  TZ    :', os.environ.get('TZ', '(not set)'))
    print('  tzname:', time.tzname)
    print('  Zone  : {} ({})'.format(time.timezone, (time.timezone / 3600)))
    print('  DST   :', time.daylight)
    print('  Time  :', time.ctime())
    print()

print('Default :')
show_zone_info()

ZONES = [
    'GMT',
    'Europe/Amsterdam',
]

for zone in ZONES:
    os.environ['TZ'] = zone
    time.tzset()
    print(zone, ':')
    show_zone_info()
```
用于准备示例的系统的默认时区是US / Eastern。 示例中的其他区域更改tzname，日光标志和时区偏移值。

```
$ python3 time_timezone.py

Default :
  TZ    : (not set)
  tzname: ('EST', 'EDT')
  Zone  : 18000 (5.0)
  DST   : 1
  Time  : Sun Mar 18 16:23:14 2018

GMT :
  TZ    : GMT
  tzname: ('GMT', 'GMT')
  Zone  : 0 (0.0)
  DST   : 0
  Time  : Sun Mar 18 20:23:14 2018

Europe/Amsterdam :
  TZ    : Europe/Amsterdam
  tzname: ('CET', 'CEST')
  Zone  : -3600 (-1.0)
  DST   : 1
  Time  : Sun Mar 18 21:23:14 2018
```

### Parsing and Formatting Times

strptime（）和strftime（）这两个函数在struct_time和时间值的字符串表示之间进行转换。有一长串格式化指令可用于支持不同样式的输入和输出。完整列表记录在时间模块的库文档中。  
此示例将当前时间从字符串转换为struct_time实例并返回到字符串。

```python
#time_strptime.py
import time

def show_struct(s):
    print('  tm_year :', s.tm_year)
    print('  tm_mon  :', s.tm_mon)
    print('  tm_mday :', s.tm_mday)
    print('  tm_hour :', s.tm_hour)
    print('  tm_min  :', s.tm_min)
    print('  tm_sec  :', s.tm_sec)
    print('  tm_wday :', s.tm_wday)
    print('  tm_yday :', s.tm_yday)
    print('  tm_isdst:', s.tm_isdst)

now = time.ctime(1483391847.433716)
print('Now:', now)

parsed = time.strptime(now)
print('\nParsed:')
show_struct(parsed)

print('\nFormatted:',
      time.strftime("%a %b %d %H:%M:%S %Y", parsed))
```
输出字符串与输入不完全相同，因为月份的日期前缀为零。

```
$ python3 time_strptime.py

Now: Mon Jan  2 16:17:27 2017

Parsed:
  tm_year : 2017
  tm_mon  : 1
  tm_mday : 2
  tm_hour : 16
  tm_min  : 17
  tm_sec  : 27
  tm_wday : 0
  tm_yday : 2
  tm_isdst: -1

Formatted: Mon Jan 02 16:17:27 2017
```

## datetime — Date and Time Value Manipulation

目的：datetime模块包括用于进行日期和时间的解析、格式化和算术运算的函数和类。  
datetime包含用于处理日期和时间的函数和类，分开或一起。

### Times

时间值用time类表示。time实例具有hour，minute，second和microsecond属性，还可以包括时区信息。

```python
#datetime_time.py
import datetime

t = datetime.time(1, 2, 3)
print(t)
print('hour       :', t.hour)
print('minute     :', t.minute)
print('second     :', t.second)
print('microsecond:', t.microsecond)
print('tzinfo     :', t.tzinfo)
```
初始化time实例的参数是可选的，但默认值0不太可能是正确的。

```
$ python3 datetime_time.py

01:02:03
hour       : 1
minute     : 2
second     : 3
microsecond: 0
tzinfo     : None
```
time实例仅保存时间值，而不包含与时间关联的日期。

```python
#datetime_time_minmax.py
import datetime

print('Earliest  :', datetime.time.min)
print('Latest    :', datetime.time.max)
print('Resolution:', datetime.time.resolution)
```
min和max类属性反映了一天中的有效时间范围。

```
$ python3 datetime_time_minmax.py

Earliest  : 00:00:00
Latest    : 23:59:59.999999
Resolution: 0:00:00.000001
```
time分辨率限制为整秒。

```python
#datetime_time_resolution.py
import datetime

for m in [1, 0, 0.1, 0.6]:
    try:
        print('{:02.1f} :'.format(m), datetime.time(0, 0, 0, microsecond=m))
    except TypeError as err:
        print('ERROR:', err)
```
微秒的浮点值会导致TypeError。

```
$ python3 datetime_time_resolution.py

1.0 : 00:00:00.000001
0.0 : 00:00:00
ERROR: integer argument expected, got float
ERROR: integer argument expected, got float
```

### Dates

日历日期值用date类表示。实例具有year，month和日day属性。使用today()类方法很容易创建表示当前日期的date。

```python
#datetime_date.py
import datetime

today = datetime.date.today()
print(today)
print('ctime  :', today.ctime())
tt = today.timetuple()
print('tuple  : tm_year  =', tt.tm_year)
print('         tm_mon   =', tt.tm_mon)
print('         tm_mday  =', tt.tm_mday)
print('         tm_hour  =', tt.tm_hour)
print('         tm_min   =', tt.tm_min)
print('         tm_sec   =', tt.tm_sec)
print('         tm_wday  =', tt.tm_wday)
print('         tm_yday  =', tt.tm_yday)
print('         tm_isdst =', tt.tm_isdst)
print('ordinal:', today.toordinal())
print('Year   :', today.year)
print('Mon    :', today.month)
print('Day    :', today.day)
```
此示例以多种格式打印当前日期：

```
$ python3 datetime_date.py

2018-03-18
ctime  : Sun Mar 18 00:00:00 2018
tuple  : tm_year  = 2018
         tm_mon   = 3
         tm_mday  = 18
         tm_hour  = 0
         tm_min   = 0
         tm_sec   = 0
         tm_wday  = 6
         tm_yday  = 77
         tm_isdst = -1
ordinal: 736771
Year   : 2018
Mon    : 3
Day    : 18
```
还有用于从POSIX时间戳或表示公历中日期值的整数（其中1年1月1日为1，后续每天将值递增1）创建实例的类方法。

```python
#datetime_date_fromordinal.py
import datetime
import time

o = 733114
print('o               :', o)
print('fromordinal(o)  :', datetime.date.fromordinal(o))

t = time.time()
print('t               :', t)
print('fromtimestamp(t):', datetime.date.fromtimestamp(t))
```
此示例说明了fromordinal（）和fromtimestamp（）使用的不同值类型。

```
$ python3 datetime_date_fromordinal.py

o               : 733114
fromordinal(o)  : 2008-03-13
t               : 1521404434.262209
fromtimestamp(t): 2018-03-18
```
与time一样，可以使用min和max属性确定支持的date值范围。

```python
#datetime_date_minmax.py
import datetime

print('Earliest  :', datetime.date.min)
print('Latest    :', datetime.date.max)
print('Resolution:', datetime.date.resolution)
```
日期的分辨率是整天。

```
$ python3 datetime_date_minmax.py

Earliest  : 0001-01-01
Latest    : 9999-12-31
Resolution: 1 day, 0:00:00
```
创建新date实例的另一种方法是使用现有date的replace（）方法。

```python
#datetime_date_replace.py
import datetime

d1 = datetime.date(2008, 3, 29)
print('d1:', d1.ctime())

d2 = d1.replace(year=2009)
print('d2:', d2.ctime())
```
此示例更改年份，使日期和月份保持不变。

```
$ python3 datetime_date_replace.py

d1: Sat Mar 29 00:00:00 2008
d2: Sun Mar 29 00:00:00 2009
```

### timedeltas

可以使用两个datetime对象的基本算数运算或通过将datetime与timedelta组合来计算未来和过去的日期。日期相减会产生timedelta，并且可以从date中添加或减去timedelta以产生另一个date。timedelta的内部值以days，seconds和microseconds存储。

```python
#datetime_timedelta.py
import datetime

print('microseconds:', datetime.timedelta(microseconds=1))
print('milliseconds:', datetime.timedelta(milliseconds=1))
print('seconds     :', datetime.timedelta(seconds=1))
print('minutes     :', datetime.timedelta(minutes=1))
print('hours       :', datetime.timedelta(hours=1))
print('days        :', datetime.timedelta(days=1))
print('weeks       :', datetime.timedelta(weeks=1))
```
传递给构造函数的中间级别值将转换为天，秒和微秒。

```
$ python3 datetime_timedelta.py

microseconds: 0:00:00.000001
milliseconds: 0:00:00.001000
seconds     : 0:00:01
minutes     : 0:01:00
hours       : 1:00:00
days        : 1 day, 0:00:00
weeks       : 7 days, 0:00:00
```
可以使用total_seconds（）将timedelta的完整持续时间检索为秒数。

```python
#datetime_timedelta_total_seconds.py
import datetime

for delta in [datetime.timedelta(microseconds=1),
              datetime.timedelta(milliseconds=1),
              datetime.timedelta(seconds=1),
              datetime.timedelta(minutes=1),
              datetime.timedelta(hours=1),
              datetime.timedelta(days=1),
              datetime.timedelta(weeks=1),
              ]:
    print('{:15} = {:8} seconds'.format(str(delta), delta.total_seconds()) )
```
返回值是浮点数，以适应以秒为单位的持续时间。

```
$ python3 datetime_timedelta_total_seconds.py

0:00:00.000001  =    1e-06 seconds
0:00:00.001000  =    0.001 seconds
0:00:01         =      1.0 seconds
0:01:00         =     60.0 seconds
1:00:00         =   3600.0 seconds
1 day, 0:00:00  =  86400.0 seconds
7 days, 0:00:00 = 604800.0 seconds
```

### Date Arithmetic

日期数学使用标准算术运算符。

```python
#datetime_date_math.py
import datetime

today = datetime.date.today()
print('Today    :', today)

one_day = datetime.timedelta(days=1)
print('One day  :', one_day)

yesterday = today - one_day
print('Yesterday:', yesterday)

tomorrow = today + one_day
print('Tomorrow :', tomorrow)

print()
print('tomorrow - yesterday:', tomorrow - yesterday)
print('yesterday - tomorrow:', yesterday - tomorrow)
```
此date对象示例说明了使用timedelta对象计算新日期，并date实例相减以生成timedeltas（包括负delta值）。

```
$ python3 datetime_date_math.py

Today    : 2018-03-18
One day  : 1 day, 0:00:00
Yesterday: 2018-03-17
Tomorrow : 2018-03-19

tomorrow - yesterday: 2 days, 0:00:00
yesterday - tomorrow: -2 days, 0:00:00
```
timedelta对象还支持与整数，浮点数和其他timedelta实例进行算术运算。

```python
#datetime_timedelta_math.py
import datetime

one_day = datetime.timedelta(days=1)
print('1 day    :', one_day)
print('5 days   :', one_day * 5)
print('1.5 days :', one_day * 1.5)
print('1/4 day  :', one_day / 4)

# assume an hour for lunch
work_day = datetime.timedelta(hours=7)
meeting_length = datetime.timedelta(hours=1)
print('meetings per day :', work_day / meeting_length)
```
在此示例中，计算一天的几个倍数，得到的timedelta保持适当的天数或小时数。最后一个示例演示了如何通过组合两个timedelta对象来计算值。 在这种情况下，结果是浮点数。

```
$ python3 datetime_timedelta_math.py

1 day    : 1 day, 0:00:00
5 days   : 5 days, 0:00:00
1.5 days : 1 day, 12:00:00
1/4 day  : 6:00:00
meetings per day : 7.0
```

### Comparing Values

可以使用标准比较运算符比较日期和时间值，以确定哪个更早或更晚。

```python
#datetime_comparing.py
import datetime
import time

print('Times:')
t1 = datetime.time(12, 55, 0)
print('  t1:', t1)
t2 = datetime.time(13, 5, 0)
print('  t2:', t2)
print('  t1 < t2:', t1 < t2)

print()
print('Dates:')
d1 = datetime.date.today()
print('  d1:', d1)
d2 = datetime.date.today() + datetime.timedelta(days=1)
print('  d2:', d2)
print('  d1 > d2:', d1 > d2)
```
支持所有比较运算符。

```
$ python3 datetime_comparing.py

Times:
  t1: 12:55:00
  t2: 13:05:00
  t1 < t2: True

Dates:
  d1: 2018-03-18
  d2: 2018-03-19
  d1 > d2: False
```

### Combining Dates and Times

使用datetime类来保存由日期和时间组件组成的值。与date一样，有几种方便的类方法可以从其他常用值创建datetime实例。

```python
#datetime_datetime.py
import datetime

print('Now    :', datetime.datetime.now())
print('Today  :', datetime.datetime.today())
print('UTC Now:', datetime.datetime.utcnow())
print()

FIELDS = [
    'year', 'month', 'day',
    'hour', 'minute', 'second',
    'microsecond',
]

d = datetime.datetime.now()
for attr in FIELDS:
    print('{:15}: {}'.format(attr, getattr(d, attr)))
```
正如所料，datetime实例具有date和time对象的所有属性。

```
$ python3 datetime_datetime.py

Now    : 2018-03-18 16:20:34.811583
Today  : 2018-03-18 16:20:34.811616
UTC Now: 2018-03-18 20:20:34.811627

year           : 2018
month          : 3
day            : 18
hour           : 16
minute         : 20
second         : 34
microsecond    : 811817
```
与date一样，datetime为创建新实例提供了方便的类方法。它还包括fromordinal（）和fromtimestamp（）。

```python
#datetime_datetime_combine.py
import datetime

t = datetime.time(1, 2, 3)
print('t :', t)

d = datetime.date.today()
print('d :', d)

dt = datetime.datetime.combine(d, t)
print('dt:', dt)
```
combine（）从一个date和一个timie实例创建datetime实例。

```
$ python3 datetime_datetime_combine.py

t : 01:02:03
d : 2018-03-18
dt: 2018-03-18 01:02:03
```

### Formatting and Parsing

datetime对象的默认字符串表示形式使用ISO-8601格式（YYYY-MM-DDTHH：MM：SS.mmmmmm）。可以使用strftime（）生成替换的格式。

```python
#datetime_datetime_strptime.py
import datetime

format = "%a %b %d %H:%M:%S %Y"

today = datetime.datetime.today()
print('ISO     :', today)

s = today.strftime(format)
print('strftime:', s)

d = datetime.datetime.strptime(s, format)
print('strptime:', d.strftime(format))
```
使用datetime.strptime（）将格式化字符串转换为datetime实例。

```
$ python3 datetime_datetime_strptime.py

ISO     : 2018-03-18 16:20:34.941204
strftime: Sun Mar 18 16:20:34 2018
strptime: Sun Mar 18 16:20:34 2018
```
Python的字符串格式化迷你语言可以使用相同的格式代码，通过在格式字符串的字段说明符中把格式代码放置在冒号(:)后面，如下所示：

```python
#datetime_format.py
import datetime

today = datetime.datetime.today()
print('ISO     :', today)
print('format(): {:%a %b %d %H:%M:%S %Y}'.format(today))
```
每个datetime格式代码仍必须以％为前缀，后续冒号将作为文字字符处理，以包含在输出中。

```
$ python3 datetime_format.py

ISO     : 2018-03-18 16:20:35.006116
format(): Sun Mar 18 16:20:35 2018
```
下表显示了美国/东部时区2016年1月13日下午5:00的所有格式代码。

strptime/strftime format codes

Symbol	 | Meaning	                                              | Example
:---     | :---                                                   | :---
%a	     | Abbreviated weekday name	                              | 'Wed'
%A	     | Full weekday name	                                    | 'Wednesday'
%w	     | Weekday number – 0 (Sunday) through 6 (Saturday)	      | '3'
%d	     | Day of the month (zero padded)	                        | '13'
%b	     | Abbreviated month name	                                | 'Jan'
%B	     | Full month name	                                      | 'January'
%m	     | Month of the year	                                    | '01'
%y	     | Year without century	                                  | '16'
%Y	     | Year with century	                                    | '2016'
%H	     | Hour from 24-hour clock	                              | '17'
%I	     | Hour from 12-hour clock	                              | '05'
%p	     | AM/PM	                                                | 'PM'
%M	     | Minutes	                                              | '00'
%S	     | Seconds	                                              | '00'
%f	     | Microseconds	                                          | '000000'
%z	     | UTC offset for time zone-aware objects	                | '-0500'
%Z	     | Time Zone name	                                        | 'EST'
%j	     | Day of the year	                                      | '013'
%W	     | Week of the year	                                      | '02'
%c	     | Date and time representation for the current locale	  | 'Wed Jan 13 17:00:00 2016'
%x	     | Date representation for the current locale	            | '01/13/16'
%X	     | Time representation for the current locale	            | '17:00:00'
%%	     | A literal % character	                                | '%'

### Time Zones

在datetime中，时区由tzinfo的子类表示。由于tzinfo是一个抽象基类，因此应用程序需要定义一个子类，并为一些方法提供适当的实现以使其有用。
datetime确实在类timezone中包含一个有点不成熟的实现，它使用UTC的固定偏移量，并且不支持一年中不同日期的不同偏移值，例如夏令时适用的地方，或UTC的偏移量会随着时间的推移改变的地方。

```python
#datetime_timezone.py
import datetime

min6 = datetime.timezone(datetime.timedelta(hours=-6))
plus6 = datetime.timezone(datetime.timedelta(hours=6))
d = datetime.datetime.now(min6)

print(min6, ':', d)
print(datetime.timezone.utc, ':', d.astimezone(datetime.timezone.utc))
print(plus6, ':', d.astimezone(plus6))

# convert to the current system timezone
d_system = d.astimezone()
print(d_system.tzinfo, '      :', d_system)
```
要将datetime值从一个时区转换为另一个时区，请使用astimezone（）。 在上面的示例中，显示了UTC两侧6小时的两个独立时区，而datetime.timezone中的utc实例也用于参考。 最终输出行显示系统时区中的值，通过不带参数调用astimezone（）获取。

```
$ python3 datetime_timezone.py

UTC-06:00 : 2018-03-18 14:20:35.123594-06:00
UTC : 2018-03-18 20:20:35.123594+00:00
UTC+06:00 : 2018-03-19 02:20:35.123594+06:00
EDT       : 2018-03-18 16:20:35.123594-04:00
```
>注意  
第三方模块pytz是一个更好的时区实现。它支持命名时区，并且随着世界各地政治机构做出的变化，偏移数据库保持最新。

## calendar — Work with Dates

目的：日历模块实现用于处理日期的类，以管理面向年/月/周的值。  
calendar模块定义了Calendar类，它封装了诸如给定月份或年份中周的日期等值的计算。此外，TextCalendar和HTMLCalendar类可以生成预格式化的输出。

### Formatting Examples

prmonth（）方法是一个简单的函数，可以生成一个月的格式化文本输出。

```python
#calendar_textcalendar.py
import calendar

c = calendar.TextCalendar(calendar.SUNDAY)
c.prmonth(2017, 7)
```
该示例将TextCalendar配置为按照美国惯例在星期日开始周。默认是使用星期一开始一周的欧洲惯例。
输出看起来像这样：

```
$ python3 calendar_textcalendar.py

     July 2017
Su Mo Tu We Th Fr Sa
                   1
 2  3  4  5  6  7  8
 9 10 11 12 13 14 15
16 17 18 19 20 21 22
23 24 25 26 27 28 29
30 31
```
可以使用HTMLCalendar和formatmonth（）生成类似的HTML表。呈现的输出看起来与纯文本版本大致相同，但是用HTML标记包装。每个表格单元格都有一个与星期几相对应的类属性，因此HTML可以通过CSS设置样式。  
要以不同于其中一个可用默认值的格式生成输出，请使用calendar计算日期并将值组织为周和月范围，然后迭代结果。Calendar的weekheader（），monthcalendar（）和yeardays2calendar（）方法对此特别有用。  
调用yeardays2calendar（）会产生一系列“月份行”列表。每个列表包括月份作为另一个周列表。这几周是由日期编号（1-31）和工作日编号（0-6）组成的元组列表。超出月份的天数为0。

```python
#calendar_yeardays2calendar.py
import calendar
import pprint

cal = calendar.Calendar(calendar.SUNDAY)

cal_data = cal.yeardays2calendar(2017, 3)
print('len(cal_data)      :', len(cal_data))

top_months = cal_data[0]
print('len(top_months)    :', len(top_months))

first_month = top_months[0]
print('len(first_month)   :', len(first_month))

print('first_month:')
pprint.pprint(first_month, width=65)
```
调用yeardays2calendar（2017,3）返回2017年的数据，每行三个月组织。

```
$ python3 calendar_yeardays2calendar.py

len(cal_data)      : 4
len(top_months)    : 3
len(first_month)   : 5
first_month:
[[(1, 6), (2, 0), (3, 1), (4, 2), (5, 3), (6, 4), (7, 5)],
 [(8, 6), (9, 0), (10, 1), (11, 2), (12, 3), (13, 4), (14, 5)],
 [(15, 6), (16, 0), (17, 1), (18, 2), (19, 3), (20, 4), (21, 5)],
 [(22, 6), (23, 0), (24, 1), (25, 2), (26, 3), (27, 4), (28, 5)],
 [(29, 6), (30, 0), (31, 1), (0, 2), (0, 3), (0, 4), (0, 5)]]
```
这相当于formatyear（）使用的数据。

```python
#calendar_formatyear.py
import calendar

cal = calendar.TextCalendar(calendar.SUNDAY)
print(cal.formatyear(2017, 2, 1, 1, 3))
```
对于相同的参数，formatyear（）产生以下输出：

```
$ python3 calendar_formatyear.py

                              2017

      January               February               March
Su Mo Tu We Th Fr Sa  Su Mo Tu We Th Fr Sa  Su Mo Tu We Th Fr Sa
 1  2  3  4  5  6  7            1  2  3  4            1  2  3  4
 8  9 10 11 12 13 14   5  6  7  8  9 10 11   5  6  7  8  9 10 11
15 16 17 18 19 20 21  12 13 14 15 16 17 18  12 13 14 15 16 17 18
22 23 24 25 26 27 28  19 20 21 22 23 24 25  19 20 21 22 23 24 25
29 30 31              26 27 28              26 27 28 29 30 31

       April                  May                   June
Su Mo Tu We Th Fr Sa  Su Mo Tu We Th Fr Sa  Su Mo Tu We Th Fr Sa
                   1      1  2  3  4  5  6               1  2  3
 2  3  4  5  6  7  8   7  8  9 10 11 12 13   4  5  6  7  8  9 10
 9 10 11 12 13 14 15  14 15 16 17 18 19 20  11 12 13 14 15 16 17
16 17 18 19 20 21 22  21 22 23 24 25 26 27  18 19 20 21 22 23 24
23 24 25 26 27 28 29  28 29 30 31           25 26 27 28 29 30
30

        July                 August              September
Su Mo Tu We Th Fr Sa  Su Mo Tu We Th Fr Sa  Su Mo Tu We Th Fr Sa
                   1         1  2  3  4  5                  1  2
 2  3  4  5  6  7  8   6  7  8  9 10 11 12   3  4  5  6  7  8  9
 9 10 11 12 13 14 15  13 14 15 16 17 18 19  10 11 12 13 14 15 16
16 17 18 19 20 21 22  20 21 22 23 24 25 26  17 18 19 20 21 22 23
23 24 25 26 27 28 29  27 28 29 30 31        24 25 26 27 28 29 30
30 31

      October               November              December
Su Mo Tu We Th Fr Sa  Su Mo Tu We Th Fr Sa  Su Mo Tu We Th Fr Sa
 1  2  3  4  5  6  7            1  2  3  4                  1  2
 8  9 10 11 12 13 14   5  6  7  8  9 10 11   3  4  5  6  7  8  9
15 16 17 18 19 20 21  12 13 14 15 16 17 18  10 11 12 13 14 15 16
22 23 24 25 26 27 28  19 20 21 22 23 24 25  17 18 19 20 21 22 23
29 30 31              26 27 28 29 30        24 25 26 27 28 29 30
                                            31
```
`day_name`，`day_abbr`，`month_name`和`month_abbr`模块属性可用于生成自定义格式化输出（即，在HTML输出中包含链接）。 它们会针对当前区域设置自动正确配置。

### Locales

要生成为当前默认区域以外的区域设置格式化的日历，请使用LocaleTextCalendar或LocaleHTMLCalendar。

```python
#calendar_locale.py
import calendar

c = calendar.LocaleTextCalendar(locale='en_US')
c.prmonth(2017, 7)

print()

c = calendar.LocaleTextCalendar(locale='fr_FR')
c.prmonth(2017, 7)
```
一周的第一天不是locale设置的一部分，并且该值从参数获取到日历类，就像使用常规TextCalendar一样。

```
$ python3 calendar_locale.py

     July 2017
Mo Tu We Th Fr Sa Su
                1  2
 3  4  5  6  7  8  9
10 11 12 13 14 15 16
17 18 19 20 21 22 23
24 25 26 27 28 29 30
31

    juillet 2017
Lu Ma Me Je Ve Sa Di
                1  2
 3  4  5  6  7  8  9
10 11 12 13 14 15 16
17 18 19 20 21 22 23
24 25 26 27 28 29 30
31
```

### Calculating Dates

虽然日历模块主要侧重于以各种格式打印完整日历，但它还提供了以其他方式处理日期的有用功能，例如计算重复事件的日期。例如，Python亚特兰大用户组在每个月的第二个星期四开会。要计算一年的会议日期，请使用monthcalendar（）的返回值。

```python
#calendar_monthcalendar.py
import calendar
import pprint

pprint.pprint(calendar.monthcalendar(2017, 7))
```
有些日子有0值。这些是与给定月份重叠的一周中的日子，但这是另一个月的一部分。

```
$ python3 calendar_monthcalendar.py

[[0, 0, 0, 0, 0, 1, 2],
 [3, 4, 5, 6, 7, 8, 9],
 [10, 11, 12, 13, 14, 15, 16],
 [17, 18, 19, 20, 21, 22, 23],
 [24, 25, 26, 27, 28, 29, 30],
 [31, 0, 0, 0, 0, 0, 0]]
```
一周的第一天默认为星期一。可以通过调用setfirstweekday（）来更改它，但由于日历模块包含用于索引monthcalendar（）返回的日期范围的常量，因此在这种情况下跳过该步骤会更方便。  
要计算一年的组会议日期，假设它们始终在每个月的第二个星期四，请查看monthcalendar（）的输出以查找星期四下降的日期。本月的第一周和最后一周填充0值作为前一个月或后一个月的天数的占位符。 例如，如果一个月在星期五开始，则星期四位置第一周的值将为0。

```python
#calendar_secondthursday.py
import calendar
import sys

year = int(sys.argv[1])

# Show every month
for month in range(1, 13):

    # Compute the dates for each week that overlaps the month
    c = calendar.monthcalendar(year, month)
    first_week = c[0]
    second_week = c[1]
    third_week = c[2]

    # If there is a Thursday in the first week,
    # the second Thursday is # in the second week.
    # Otherwise, the second Thursday must be in
    # the third week.
    if first_week[calendar.THURSDAY]:
        meeting_date = second_week[calendar.THURSDAY]
    else:
        meeting_date = third_week[calendar.THURSDAY]

    print('{:>3}: {:>2}'.format(calendar.month_abbr[month],
                                meeting_date))
```
所以该年的会议日程是：

```
$ python3 calendar_secondthursday.py 2017

Jan: 12
Feb:  9
Mar:  9
Apr: 13
May: 11
Jun:  8
Jul: 13
Aug: 10
Sep: 14
Oct: 12
Nov:  9
Dec: 14
```

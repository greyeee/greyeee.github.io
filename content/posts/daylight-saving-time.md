+++
title = '夏令时'
date = 2024-04-24T22:07:33+08:00
draft = false

+++

## 夏令时是什么？

夏令时英文叫**Daylight Saving Time**，简称**DST**。高纬度地区由于夏季太阳升起时间明显比冬季早，很多国家为了充分利用夏季的太阳光照，节约照明用电，在天亮早的**夏季**会人为将时间**调快一小时**，实行夏时制。而当冬季来临，这些国家会将时针再拨回原来的**标准时间（Standard Time）**。

美国夏令时一般在**3月第二个周日凌晨2AM**（当地时间）开始，将时钟调到3点，拨快1小时，俗称“**Spring Forward 1 Hour**”；而在**11月第一个周日凌晨2AM**（当地时间）夏令时结束，要将时钟调到1点，拨慢1小时，俗称“**Fall Back 1 Hour**”。

![](https://imgcache.dealmoon.com/thumbimg.dealmoon.com/dealmoon/66e/93e/4ba/f6b51807f8b92d79ea6a082.jpg_1080_0_3_15d1.jpg)

## IANA时区

IANA Time Zone Database，是一组表示地球上各地的时间历史的代码和数据，由互联网号码分配机构（Internet Assigned Numbers Authority，IANA）维护。该数据库包含了全球各国的时间信息，包括时区边界、UTC（世界标准时间）和夏令时等规则。IANA会根据各地政体的变化而定期更新关于时区边界、UTC和夏令时等的规则。

zoneinfo是IANA Time Zone Database的一种格式，用于表示世界各个国家和地区的时区和夏令时信息。在 Linux 系统中中位于：/usr/share/zoneinfo，每个文件都对应着一个时区。这些文件的名称通常基于地理位置或国家名称，例如 America/New_York 和 Asia/Shanghai 等。

嵌入式设备通过 `TZ`环境变量指定时区。支持三种格式

1. `std offset`，当当地时区没有夏令时时使用，比如`EST+5`，指北美东部标准时间Eastern Standard Time与 UTC 的偏移量为 5 小时。
2. `std offset dst [offset],start[/time],end[/time]`，当有夏令时时使用，比如`EST+5EDT,M3.2.0/2,M11.1.0/2`，表示北美东部标准时间 (EST) 与 UTC 的正常偏移量为 5 小时。夏令时从 3 月的第二个星期日凌晨 2:00 开始，进入东部夏令时间 (EDT) ，到 11 月的第一个星期日凌晨 2:00 结束。

3. `:characters`，比如`America/New_York`，库会查找`/usr/share/zoneinfo/America/New_York`文件。

第1和第2种格式称为POSIX格式，第3种格式称为Olson格式。

如果嵌入式设备资源有限，无法引入整个时区数据库，可以采用POSIX格式，只采用最新的夏令时规则，历史的夏令时规则不管，缺点是历史时间用最新的夏令时规则解析会不对。

每个时区文件的最后一行就是时区偏移和最新的夏令时规则，POSIX格式，比如`/usr/share/zoneinfo/America/New_York`最后一行是`EST5EDT,M3.2.0,M11.1.0`

## 代码实现注意点

1. 通信接口使用 UTC 并格式化为[ISO 8601](http://en.wikipedia.org/wiki/ISO_8601)字符串，例如：`2024-04-24T09:14:57Z`。或者使用 Unix 时间，例如：`1455821369`。

2. app时间展示根据设备的时区渲染成本地时间。

   ```golang
   package main
   
   import (
   	"fmt"
   	"time"
   )
   
   func main() {
   	utcTimeA, _ := time.Parse(time.RFC3339, "2024-11-03T05:30:00Z")
   	utcTimeB, _ := time.Parse(time.RFC3339, "2024-11-03T06:30:00Z")
   	location, _ := time.LoadLocation("America/New_York")
   	localTimeA := utcTimeA.In(location)
   	localTimeB := utcTimeB.In(location)
   	fmt.Println(localTimeA, localTimeB) // 2024-11-03 01:30:00 -0400 EDT 2024-11-03 01:30:00 -0500 EST
     // 2024-11-03 2:00:00结束夏令时，回退1小时，所以两个不同的UTC时间展示的本地时间相同
   }
   ```

3. 有时间范围查询的接口，app要将用户选择的本地时间转换为UTC时间。比如`America/New_York`时区查询`2024-03-10 00:00:00` ~ `2024-03-11 00:00:00`范围的事件列表。

   ```go
   package main
   
   import (
   	"fmt"
   	"time"
   )
   
   func main() {
   	location, _ := time.LoadLocation("America/New_York")
   	startTime := time.Date(2024, 3, 10, 0, 0, 0, 0, location).UTC()
   	endTime := time.Date(2024, 3, 11, 0, 0, 0, 0, location).UTC()
   	fmt.Println(startTime, endTime)             // 2024-03-10 05:00:00 +0000 UTC 2024-03-11 04:00:00 +0000 UTC
   	fmt.Println(endTime.Sub(startTime).Hours()) // 可以看到夏令时开始这天只有23个小时
   }
   ```

4. 定时任务

   1. 进入夏令时，时间拨快1小时，即消失了1小时，如果任务落在消失的1小时内，任务将被跳过。夏令时结束时，时间回退1小时，即这1小时重复2次，任务要保证仅执行一次并且不会重复执行。
   2. 用户设置的定时比如每天8点是本地时间的概念，要用明天8点的本地时间减去当前本地时间计算任务的延时，或者明天8点的本地时间转换为时间戳提交任务。
   3. 当更改时区时，`America/New_York`的明天8点和`Europe/London`的明天8点肯定不同，延时要重新计算并更新。

5. 夏令时结束的前后一小时，展示上会有回退和本地时间重复，比如事件列表顺序展示会出现

   1. 1:30:00，检测到人移动
   2. 1:40:00，检测到人移动
   3. 1:58:00，检测到人移动
   4. 1:09:00，检测到人移动
   5. 1:30:00，检测到人移动

6. 统计数据，比如智能喂食器的每日进食克数，如果在切换时区后也能展示正确，可以以15分钟为粒度统计，因为时区偏移最小有30min，45min，1h等，比如尼泊尔时区是**UTC+5:45**。以查询`America/New_York`时区最近7天每天进食克数为例。

   ```sql
   SELECT * FROM feeding_statistics ORDER BY time
   ```

   ```
   time, total_grams
   '2024-04-19 00:15:00.000000 +08:00', 4
   '2024-04-19 01:30:00.000000 +08:00', 9
   '2024-04-20 02:45:00.000000 +08:00', 16
   '2024-04-20 08:30:00.000000 +08:00', 3
   '2024-04-21 04:15:00.000000 +08:00', 9
   '2024-04-21 06:45:00.000000 +08:00', 7
   '2024-04-22 05:15:00.000000 +08:00', 18
   '2024-04-22 23:45:00.000000 +08:00', 6
   '2024-04-23 07:00:00.000000 +08:00', 12
   '2024-04-24 04:15:00.000000 +08:00', 4
   '2024-04-24 12:30:00.000000 +08:00', 9
   '2024-04-25 01:00:00.000000 +08:00', 8
   '2024-04-25 01:15:00.000000 +08:00', 3
   '2024-04-25 01:45:00.000000 +08:00', 6
   ```

   ```sql
   SELECT date(time AT TIME ZONE 'America/New_York') AS date, SUM(total_grams) AS total_grams
   FROM feeding_statistics
   WHERE time >= NOW() AT TIME ZONE 'America/New_York' - INTERVAL '7 days'
   GROUP BY date
   ORDER BY date ASC;
   ```

   ```
   date, total_grams
   '2024-04-18', 13
   '2024-04-19', 19
   '2024-04-20', 16
   '2024-04-21', 18
   '2024-04-22', 18
   '2024-04-23', 4
   '2024-04-24', 26
   ```

   切换成`Asia/Shanghai`时区再查询，结果不同。

   ```sql
   SELECT date(time AT TIME ZONE 'Asia/Shanghai') AS date, SUM(total_grams) AS total_grams
   FROM feeding_statistics
   WHERE time >= NOW() AT TIME ZONE 'Asia/Shanghai' - INTERVAL '7 days'
   GROUP BY date
   ORDER BY date ASC;
   ```

   ```
   '2024-04-19', 13
   '2024-04-20', 19
   '2024-04-21', 16
   '2024-04-22', 24
   '2024-04-23', 12
   '2024-04-24', 13
   '2024-04-25', 17
   ```

   

参考链接

https://www.dealmoon.com/guide/931030

https://www.iana.org/time-zones

https://developer.ibm.com/articles/au-aix-posix/

https://www.zeitverschiebung.net/en/timezone/america--new_york

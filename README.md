# 说明

先说说写这个脚本的初衷，源于某条监控 SQL。事实上，一开始关于 MySQL 的监控是通过 zabbix agent 进行的，自己写 shell、Python 等脚本执行 SQL，然后 zabbix agent 调用这些脚本来获取执行结果，这也是大多数人的做法。

但是有一天我拿到了一条需要监控的 SQL，但是这个 SQL 的执行时间超过了 30s，已经超出 zabbix agent 的最大超时时长，因此这个监控总是会显示执行超时，而无法获取值。没有办法，有问题就要解决。既然无法通过 agent，那就只能自己发送给 zabbix 了。于是我开始了解 zabbix sender，研究该协议，然后就有了该脚本。

该脚本会连接各个 MySQL 服务器（支持多实例），定时执行 SQL 语句，并将执行的结果发送到 zabbix。要完成这些，需要知道 MySQL 服务器的地址、端口、用户名、密码、数据库、zabbix 服务器的地址（如果该 MySQL 由某个 proxy 监控，需要 proxy 地址，而不是 server 的）、需要执行的 SQL、多久执行一次，以及对应的 zabbix key。这些需要在 sql.yml 中配置。

由于该脚本会主动将其采集到的数据发送给 server，无需 zabbix agent，因此需要在 zabbix 上事先创建对应的 key，并且监控项的类型必须是**zabbix 采集器**（trapper item）。

由于该脚本支持多实例，可能存在一台机器跑五六个实例的情况，如果一个个的创建这些 key 会显得很麻烦，因此该脚本本身就是基于低级发现（Low Level Discovery）的，你需要创建一个自动发现的监控项（类型同样是 zabbix 采集器），通过不同端口区别多实例。变量名称为 `{#PORT}`。

## 用法

golang 程序只对 glibc 版本有要求，因此该程序所有 Linux 发行版都可以使用。并不清楚它要求 glibc 最低版本是多少，但是 CentOS6+ 测试通过。

```sh
wget https://github.com/maxadd/MySQLMonitorForZbx/releases/download/1.0/linux
chmod +x linux
./linux -f xxx.yml -p HTTP_PORT
```

必须给它一个配置文件，配置文件格式下面会提到。http 端口是可选项，它的默认值为 1025。它用于暴露程序的内存指标，以及通过它来运行中修改程序的 debug 级别，也就是动态修改日志级别。

关于其他选项下面会提到。

## 配置文件

用于监控的 SQL 通过配置文件指定，由于存在主从监控、低级发现监控以及普通的监控，它们的配置会有一些差别，但是无论针对哪种，以下配置都是一样的：

```yml
10.2.2.2: # mysql ip
  # 监控该 mysql 的 server，如果由 proxy 监控，需要写 proxy 地址
  zbx_addr: 10.3.3.3:10051
  instances:
    # 实例之一
    3306:
      user: hehe
      password: abc@123
      database: 666
      # 在这个下面配置 SQL 相关内容
      _sql: ...
    3307:
      user: hehe
      password: abc@123
      database: 666
      _sql: ...
10.2.2.3: ...
```

## 主从监控

一般而言，一条 SQL 语句只会产出一个值。但是 `show slave status` 会输出很多列，并且根据 MySQL 版本的不同，列数也不同。也就是说执行普通的 SQL 语句，只会产生一个值，这种情况非常好处理，但是主从监控就比较麻烦了，它会输出多个值，并且每个公司关注的列都可能不一样。并且有些列的值类型是字符串，有些则是数字，这样还无法通过低级发现来创建监控项，因为低级发现生成的监控项必须是同一种类型。

由于存在多实例，所以主从监控必须用到低级发现。低级发现的原理这里就不赘述了，首先直接创建一条自动发现规则：

![](https://github.com/maxadd/MySQLMonitorForZbx/blob/master/images/ms_01.png)

通过这个规则还生成多个实例的主从监控，如果你不是多实例，也不影响。然后就要创建这个规则的监控项原型了，你要关注 `show slave status` 输出的哪些列，就建几个监控项原型（需要注意值的类型）。拿我们公司关注的 `Slave_IO_Running`、`Slave_SQL_Running` 和 `Seconds_Behind_Master` 其中的 `Slave_IO_Running` 举例：

![](https://github.com/maxadd/MySQLMonitorForZbx/blob/master/images/ms_02.png)

`{#PORT}` 这个变量是写死的，通过它来生成多实例。key 的第一个参数是固定的，第二个参数就是你要关注的列，它必须和列名一致。需要注意的是：

1. 第一个参数没有使用引号，第二个参数用了引号；
2. 两个参数之间不光有逗号，还有一个空格。

至于 key 的名字（这里是 ms.monitor）是可以自定义的。剩下要关注的列按照这种方式继续添加即可。模板建好，套在对应的主机上后，就可以修改配置文件了：

```yml
10.2.2.2:
  zbx_addr: 10.3.3.3:10051
  instances:
    3306:
      user: hehe
      password: abc@123
      database: 666
      _sql:
        # 这是低级发现的 key
        db.instance:
          # 执行的 SQL 语句
          sql: show slave status
          # 多久执行一次，单位是秒
          frequency: 30
          # 表示这是主从监控，另外还有 lld 和 normal 两种
          flag: m/s
          # 对于主从监控，它表示需要关注哪些列，多个列之间使用双冒号分隔
          # 这里有几项，监控项原型就需要有几个，它们需要一一对应
          items: Slave_IO_Running::Slave_SQL_Running::Seconds_Behind_Master
          # 监控项原型的 key
          pt_key: ms.monitor
    3307:
      user: hehe
      password: abc@123
      database: 666
      _sql:
        db.instance:
          sql: show slave status
          frequency: 30
          flag: m/s
          items: Slave_IO_Running::Slave_SQL_Running::Seconds_Behind_Master
          pt_key: ms.monitor
```

所有主从监控的机器只需要这一个模板就可以搞定。

## 低级发现

某些 SQL 可以返回多个值，比如：

    a 12
    b 232
    c 3234
    d 35

zabbix 监控项只支持一对一，这时要想监控这些值就只能将 SQL 拆成四个了。这是非常蛋疼的做法，还好 zabbix 支持低级发现来做这样的事。

对于低级发现，SQL 的执行结果可以有多行，但是只能有两列，就像上面那样。也就是说返回的结果有两个字段，并不是要求它们的显示效果是使用空格分隔的两字段，而是它们只能是数据库中的两个字段（两列），每个字段中都可以包含多个空格。且第一列中不能包含双引号。

因为有多实例的存在，同样的 SQL 可能需要在多个实例中都执行，如果使用同一个模板肯定会造成 key 的冲突（同一台机器上的 key 必须唯一）。但是为同一个 SQL 建多个模板就显得多余，一旦一个机器上跑了 10 个实例就要建 10 个模板，这是非常不合理的行为。

为了做到方便使用，还得使用低级发现。我的做法是这样的，首先建立一个低级发现的规则。

![](https://github.com/maxadd/MySQLMonitorForZbx/blob/master/images/lld_01.png)

然后给它定义一个监控项原型。这个监控项原型接收两个参数：第一个是 `{#PORT}`，表示它属于哪个实例；第二个为 `"{#KEY}"`，这是 SQL 执行结果的第一列。由于它的外面使用了双引号，因此它的结果中不能有双引号。就像这样：

![](https://github.com/maxadd/MySQLMonitorForZbx/blob/master/images/lld_02.png)

需要注意的是，`lld01[{#PORT, "{#KEY}"]` 逗号的后面必须要有一个空格，且这两个变量的名称是写死的，请不要更改。由于同一个实例可能存在有多个低级监控的情况，为了方便给它们进行分类，可以给它们加上一个应用组，当然这随意。

一个模板就这么定义好了，它会根据 SQL 的执行结果来生成相应的监控项。执行的结果有多少行，它就生成多少个监控项。接下来就是配置文件中的定义了，也很简单：

```yml
10.1.1.13:
  zbx_addr: 10.1.2.1:10051
  instances:
    3306:
      user: x
      password: x
      database: x
      _sql:
        db.lld.01: # 低级发现的 key
          sql: xxx
          frequency: 30
          flag: lld # flag 不要错了
          pt_key: lld01 # 监控项原型的 key，中括号及其内容不要写
```

可能需要为每一个低级发现的监控单独建立一个模板。

## 正常执行

正常监控就是一对一的，也是最简单的：

```yml
_sql:
  key1:
    sql: select xxx
    frequency: 30
    # 可以不写，默认就是 normal
    flag: normal
```

## 监控指标

该脚本内置一个 http 服务器，默认监听 1025 端口。访问 `http://127.0.0.1:1025/monitor?uptime` 会得到该程序运行至今的秒数。目前只支持这一种监控指标。

## 参数

使用 `-h` 会打印目前所有支持的参数：

```sh
Usage of /root/mysqlMonitor:
  -alsologtostderr
    	log to standard error as well as files
  -f string
    	config file
  -log_backtrace_at value
    	when Logging hits line file:N, emit a stack trace
  -log_dir string
    	If non-empty, write log files in this directory
  -logtostderr
    	log to standard error instead of files
  -stderrthreshold value
    	logs at or above this threshold go to stderr
  -v value
    	log level for V logs
  -vmodule value
    	comma-separated list of pattern=N settings for file-filtered Logging
```

除了 `-f` 之外，其他都由 glog 模块提供。glog 是谷歌提供的一个日志框架，它有些意思。默认情况下，它会将日志写入到内存，每 30s 刷新一次到磁盘。程序每次执行都会为每个日志级别生成一个日志，总共四个，日志文件为脚本名+主机名+用户名+级别+日期。这样看起来很费劲，因此它提供四个软链接文件，文件名为 `脚本名.级别`，它始终指向当前使用的日志文件。

日志文件格式为：

    [IWEF]mmdd hh:mm:ss.uuuuuu threadid file:line] msg

第一个字符表示级别（共四个），然后就是时间。threadid 表示进程 id，然后就是源文件的文件名和行号，最后就是消息内容了。默认所有日志文件都会输出到 /tmp 目录下，可以通过参数来控制。日志文件还有一个特点就是高级别日志不仅存在于此级别的日志文件中，所有低于它级别的日志文件中都会记录。也就是 error 级别的日志，会出现 error、warning、info 文件中。

它除了支持 info、warning、error、fatal 这四个级别外（不支持 debug）还实现了一种非常特殊的 V 级别，这个 V 级别就是加强版的 debug。debug 只是一个级别，如果我需要 debug 的信息还要分级呢？光一个 debug 级别是不够的。因此 V1 打印一级的 debug 信息，v2 打印二级的，等等。

默认的级别通过 -v 参数指定，数据越大，信息越详细。但是超过 3 没有意义，因为该脚本只支持到 v3。支持运行时修改级别，通过访问 `http://127.0.0.1:PORT/admin?verbose=x` 来修改。

参数解释：

- `-alsologtostderr`：是否将日志输出到标准错误输出，默认 false；
- `log_dir`：指定日志目录，默认在 `/tmp` 目录下；
- `-logtostderr`：将日志输出到标准错误输出而非文件，默认 false；
- `-stderrthreshold`：什么级别的日志会输出到标准错误输出，默认为 error，也就是所有 error 级别的日志都会输出到标准错误输出；
- `-v`：指定 debug 级别，默认为 0，不会输出。


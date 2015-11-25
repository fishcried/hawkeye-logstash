# What's hawkeye?

hawkEye是一套依托于ELKstask对openstack日志进行可视化的系统。

**目标**

hawkeye监视的日志主要为openstack平台，核心目标：
- 对openstack日志进行可视化，通过dashboard devops可以了解系统整体状态。
- devops debug平台。devops只需要通过hawkeye即可对openstack问题进行定位，尽量不需要登录各个子系统。

**组成**

hawkeye主要通过logstash进行日志收集与解析工作，通过elastacsearch进行存储索引，通过kibana进行展示。这里将hawkeye拆分为多个子项目。

- hawkeye-logstash
   logstash shipper configs.
- hawkeye-kibana
   可视化魔板,相关的search,visualization,dashboard.
- hawkeye-ansible
   通过ansible部署hawkeye

# What's hawkeye-logstash?

hawkeye-logstash专注于openstack的日志解析。目录结构如下：

    ├── indexer
    │   └── openstack-index.conf
    ├── patterns
    │   └── extra
    ├── README.md
    └── shipper
        ├── input-1101-[service].conf
        ├── filter-3000-first.conf
        ├── filter-3100-second-common.conf
        ├── filter-3101-[service].conf
        ├── filter-3501-auth.conf
        └── output-7000-general.conf

- indexer 为logstash作为indexer时的配置
- patterns openstack log patterns
- shipper 针对各个服务的配置文件

## shipper配置标准化

logstash的配置主要为`input | filter | output`几个stage.由于配置文件的之间的加载存在顺序问题，
所以配置文件的命名格式需要统一.当前规定为`[stage]-[order]-[service].conf`

- stage: input,filter,output
- order: 由于logstash加载配置时根据文件名的字母顺序加载，这里就用4位数字简单位置顺序。
  - 1000-1999分配给input
  - 3000-4999分配给filter
  - 7000分配给output
- service 主要是用于识别该配置针对那个service,such as nova,neutron and so on.

## filter配置规则

filter stage主要用于解析日志.这里将解析分为两个阶段：

**第一阶段: `stage1: log = timestamp + message`**

通常来讲一条日志可以解析成上面的结构。timestamp为日志的基础信息，通常还能提取出日志级别，
模块等基础信息;而message为这条日志真正的内容。所以第一阶段的比较通用，全部写在`filter-3000-first.conf`
文件内。第一次通用解析.


    $ cat filter-3000-first.conf
    filter {
            mutate {
                    gsub => ['path', "/.+/", ""]
            }

            if "oslofmt" in [tags] {
                    grok {
                            match => { "message" => "%{OPENSTACK_NORMAL}%{GREEDYDATA:message}"}
                            overwrite => ["message"]
                    }
            }

            if "rabbitmq" in [tags] {
                    grok {
                            match => { "message" => "%{RABBITMQ_PREFIX}%{GREEDYDATA:message}" }
                            overwrite => ["message"]
                    }
            }

            if "libvirtd" in [tags] {
                    grok {
                            match => { "message" => "%{LIBVIRTD_PREFIX}%{GREEDYDATA:message}" }
                            overwrite => ["message"]
                    }
            }

            if "auth" in [tags] {
                    grok {
                            match => { "message" => "%{SYS_AUTH_PREFFIX}%{GREEDYDATA:message}" }
                            overwrite => ["message"]
                    }
            }

            if "horizon" in [tags] {
                    if "horizon-access" in [tags] {
                            grok {
                                    match => { "message" => "%{HORIZON_ACCESS_ENTRY}"}
                                    add_field => ["api", "horizon"]
                                    add_field => ["loglevel", "INFO"]
                                    add_tag => ["apimetrics"]
                            }
                    }
                    if "horizon-error" in [tags] {
                            grok {
                                    match => { "message" => "%{HORIZON_ERROR_PREFIX}%{GREEDYDATA:message}" }
                                    overwrite => "message"
                            }
                            mutate {
                                    uppercase => ["loglevel"]
                            }
                    }
            }

            date {
                    match => ["logdate", "yyyy-MM-dd HH:mm:ss.SSS",
                                                             "EEE MMM dd HH:mm:ss.SSSSSS yyyy",
                                                             "dd/MMM/yyyy:HH:mm:ss",
                                                             "dd-MMM-yyyy::HH:mm:ss",
                                                             "MMM dd HH:mm:ss",
                                                             "MMM  dd HH:mm:ss",
                                                             "yyyy-MM-dd HH:mm:ss.SSS"  ]
            }

            if [loglevel] in ["WARNING","WARN","TRACE", "ERROR"] {
                    mutate {
                            add_tag => ["something_wrong"]
                    }
            }
    }

`filter-3000-first.conf`配置主要进行的工作如下：

1. 简化path.  `/var/log/neutron-server.log` -> `neutron-server.log`
2. 进行第一次通用解析.
3. 修成日志时间
4. 标记需要关注的日志级别

**第二阶段: 多message进行详细解析挖掘**

对message进行详细解析,提取详细的field.比如是一条web访问日志，通常可以提取sip,dip,method等。
这阶段每个service都有相应的个性化解析,文件为`filter-31xx-[service].conf`.

    $ cat filter-3102-glance.conf
    filter {
            if "glance-api" in [tags] {
                    if [module] == "eventlet.wsgi.server" {
                            grok {
                                    match => { "message" => "%{OPENSTACK_EVENTLET_WSGI}" }
                                    add_field => ["api", "glance"]
                                    add_tag => ["apimetrics"]
                            }
                    }
            }
    }

上面选取了glance的配置。message不是同一格式的，每个module的格式都不一样，所以可以根据module过滤关注的entry然后进行二次解析。提取相应的field.

## 操作场景
用户除了使用控制台配置日志采集，还可通过CRD的方式配置日志采集。CRD支持采集容器标准输出、容器文件和主机文件，支持多种日志采集格式。

## 创建CRD
您只需要定义LogConfig CRD即可创建采集配置，log-agent根据LogConfig CRD的变化修改相应的CLS日志主题，并设置绑定的机器组。CRD的格式如下：

apiVersion: cls.cloud.tencent.com/v1
kind: LogConfig                          ## 默认值
metadata:
  name: test                                ## CRD资源名，在集群内唯一
spec:
  clsDetail:
    topicId: xxxxxx-xx-xx-xx-xxxxxxxx       ## CLS日志主题的ID，日志主题需要在CLS中提前创建，且没有被其它采集配置占用
    logType: minimalist_log                 ## 日志采集格式，json_log代表 json 格式，delimiter_log代表分隔符格式，minimalist_log代表单行全文格式，multiline_log代表多行全文格式，fullregex_log代表完全正则格式
    extractRule:                            ## 提取、过滤规则
      ...
  inputDetail:
    type: container_stdout                  ## 采集日志的类型，包括container_stdout（容器标准输出）、container_file（容器文件）、host_file（主机文件）
    containerStdout:                        ## 容器标准输出
      namespace: default                    ## 采集容器的kubernetes命名空间，如果不指定，代表所有命名空间
      allContainers: false                  ## 是否采集指定命名空间中的所有容器的标准输出
      container: xxx                        ## 满足includeLabels的Pod中的容器名，只有在指定includeLabels时使用
      includeLabels:                         ## 采集包含指定label的Pod
        k8s-app: xxx                        ## 只采pod标签中配置"k8s-app=xxx"的pod产生的日志，与workloads、allContainers=true不能同时指定
      workloads:                            ## 要采集的容器的Pod所属的kubernetes workload
      - namespace: prod                     ## workload的命名空间
        name: sample-app                    ## workload的名字
        kind: deployment                    ## workload类型，支持deployment、daemonset、statefulset、job、cronjob
        container: xxx                      ## 要采集的容器名，如果不指定，代表workload Pod中的所有容器
    containerFile:                          ## 容器内文件
      namespace: default                    ## 采集容器的kubernetes命名空间
      container: xxx                        ## 采集容器名
      includeLabels:                         ## 采集包含指定label的Pod
        k8s-app: xxx                        ## 只采pod标签中配置"k8s-app=xxx"的pod产生的日志，与workload不能同时指定
      workload:                             ## 要采集的容器的Pod所属的kubernetes workload
        name: sample-app                    ## workload的名字                  
        kind: deployment                    ## workload类型，支持deployment、daemonset、statefulset、job、cronjob
      logPath: /opt/logs                    ## 日志文件夹，不支持通配符
      filePattern: app_*.log                ## 日志文件名，支持通配符 * 和 ? ，* 表示匹配多个任意字符，? 表示匹配单个任意字符
    hostFile:                               ## 主机文件
      logPath: /opt/logs                    ## 日志文件夹，支持通配符
      filePattern: app_*.log                ## 日志文件名，支持通配符 * 和 ? ，* 表示匹配多个任意字符，? 表示匹配单个任意字符
      customLablels
        k1: v1

## 日志输入类型
### 单行全文格式
单行全文日志是指一行日志内容为一条完整的日志。日志服务在采集的时候，将使用换行符\n来作为一条日志日志的结束符。为了统一结构化管理，每条日志都会存在一个默认的键值__CONTENT__，但日志数据本身不再进行日志结构化处理，也不会提取日志字段，日志属性的时间项由日志采集的时间决定。
LogConfig配置参考示例如下：
apiVersion: cls.cloud.tencent.com/v1
kind: LogConfig
spec:
  clsDetail:
    topicId: xxxxxx-xx-xx-xx-xxxxxxxx
    #单行日志
    logType: minimalist_log
### 多行全文格式
多行全文日志是指一条完整的日志数据可能跨占多行（例如 Java  stacktrace）。在这种情况下，以换行符\n 为日志的结束标识符就显得有些不合理，为了能让日志系统明确区分开每条日志，采用首行正则的方式进行匹配，当某行日志匹配上预先设置的正则表达式，就认为是一条日志的开头，而下一个行首出现作为该条日志的结束标识符。
多行全文也会设置一个默认的键值__CONTENT__，但日志数据本身不再进行日志结构化处理，也不会提取日志字段，日志属性的时间项由日志采集的时间决定。
LogConfig配置的参考如下：
apiVersion: cls.cloud.tencent.com/v1
kind: LogConfig
spec:
  clsDetail:
    topicId: xxxxxx-xx-xx-xx-xxxxxxxx
    #多行日志
    logType: multiline_log
    extractRule:
      #只有以日期时间开头的行才被认为是新一条日志的开头，否则就添加换行符\n并追加到当前日志的尾部
      beginningRegex: \d{4}-\d{2}-\d{2}\s\d{2}:\d{2}:\d{2},\d{3}\s.+
### 完全正则格式
完全正则格式通常用来处理结构化的日志，指将一条完整日志按正则方式提取多个 key-value 的日志解析模式。
LogConfig配置的参考如下：
apiVersion: cls.cloud.tencent.com/v1
kind: LogConfig
spec:
  clsDetail:
    topicId: xxxxxx-xx-xx-xx-xxxxxxxx
    #完全正则格式
    logType: fullregex_log
    extractRule:
      #正则表达式，会根据()捕获组提取对应的value
      logRegex: (\S+)[^\[]+(\[[^:]+:\d+:\d+:\d+\s\S+)\s"(\w+)\s(\S+)\s([^"]+)"\s(\S+)\s(\d+)\s(\d+)\s(\d+)\s"([^"]+)"\s"([^"]+)"\s+(\S+)\s(\S+).*
      beginningRegex: (\S+)[^\[]+(\[[^:]+:\d+:\d+:\d+\s\S+)\s"(\w+)\s(\S+)\s([^"]+)"\s(\S+)\s(\d+)\s(\d+)\s(\d+)\s"([^"]+)"\s"([^"]+)"\s+(\S+)\s(\S+).*
      #提取的key列表，与提取的value的一一对应
      keys:  ['remote_addr','time_local','request_method','request_url','http_protocol','http_host','status','request_length','body_bytes_sent','http_referer','http_user_agent','request_time','upstream_response_time']
### JSON格式
JSON 格式日志会自动提取首层的 key 作为对应字段名，首层的 value 作为对应的字段值，以该方式将整条日志进行结构化处理，每条完整的日志以换行符\n为结束标识符。
LogConfig配置的参考如下：
apiVersion: cls.cloud.tencent.com/v1
kind: LogConfig
spec:
  clsDetail:
    topicId: xxxxxx-xx-xx-xx-xxxxxxxx
    #JSON格式日志
    logType: json_log
### 分隔符格式
分隔符日志是指一条日志数据可以根据指定的分隔符将整条日志进行结构化处理，每条完整的日志以换行符\n为结束标识符。日志服务在进行分隔符格式日志处理时，您需要为每个分开的字段定义唯一的 key。
LogConfig配置的参考如下：
apiVersion: cls.cloud.tencent.com/v1
kind: LogConfig
spec:
  clsDetail:
    topicId: xxxxxx-xx-xx-xx-xxxxxxxx
    #分隔符日志
    logType: delimiter_log
    extractRule:
      #分隔符
      delimiter: ':::'
      #提取的key列表，与被分割的字段一一对应
      keys: ['IP','time','request','host','status','length','bytes','referer']

## 采集日志的类型
### 容器标准输出
示例1：
采集default命名空间中的所有容器的标准输出
apiVersion: cls.cloud.tencent.com/v1
kind: LogConfig
spec:
  inputDetail:
    type: container_stdout
    containerStdout:
      namespace: default
      allContainers: true
 ...
示例2:
采集production命名空间中属于ingress-gateway deployment的pod中的容器的标准输出
apiVersion: cls.cloud.tencent.com/v1
kind: LogConfig
spec:
  inputDetail:
    type: container_stdout
    containerStdout:
      allContainers: false
      workloads:
      - namespace: production
        name: ingress-gateway
        kind: deployment
  ...
示例3:
采集production命名空间中pod标签中包含“k8s-app=nginx”的pod中的容器的标准输出
apiVersion: cls.cloud.tencent.com/v1
kind: LogConfig
spec:
  inputDetail:
    type: container_stdout
    containerStdout:
      namespace: production
      allContainers: false
      includeLabels:
        k8s-app: nginx
  ...
### 容器文件
示例1:

采集production命名空间中属于ingress-gateway deployment的pod中的nginx容器中/data/nginx/log/路径下名为access.log的文件
apiVersion: cls.cloud.tencent.com/v1
kind: LogConfig
spec:
  topicId: xxxxxx-xx-xx-xx-xxxxxxxx
  inputDetail:
    type: container_file
    containerFile:
      namespace: production
      workload:
        name: ingress-gateway
        type: deployment
      container: nginx
      logPath: /data/nginx/log
      filePattern: access.log
  ...
示例2:

采集production命名空间中pod标签包含“k8s-app=ingress-gateway“的pod中的nginx容器中/data/nginx/log/路径下名为access.log的文件
apiVersion: cls.cloud.tencent.com/v1
kind: LogConfig
spec:
  inputDetail:
    type: container_file
    containerFile:
      namespace: production
      includeLabels:
        k8s-app: ingress-gateway
      container: nginx
      logPath: /data/nginx/log
      filePattern: access.log
  ...
### 主机文件
示例

采集主机/data/路径下所有.log文件
apiVersion: cls.cloud.tencent.com/v1
kind: LogConfig
spec:
  inputDetail:
    type: host_file
    hostFile:
      logPath: /data
      filePattern: *.log
  ...


## 查看CRD
执行。。。查看当前所有的CRD采集配置

执行。。。查看当前所有的CRD采集配置



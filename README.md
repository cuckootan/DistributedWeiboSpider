# 分布式新浪微博爬虫

---

## 1 简介

该程序用于爬取新浪微博的数据，主要用于学术研究。具体数据包括：

-   发布微博的作者的个人信息，包括用户ID，昵称，性别，地区；
-   作者的所有关注的人；
-   作者的所有粉丝；
-   作者发布的所有微博的微博ID，发布时间；
-   每条微博的文字；
-   每条微博的所有图片；
-   每条微博的所有评论者的昵称，评论的文字，以及评论的时间；
-   每条微博的所有转发者的昵称，以及转发的时间；
-   每条微博的所有点赞者的昵称，以及点赞的时间。

[新浪微博爬虫单机版](https://github.com/cuckootan/WeiboSpider) 的性能较低，在爬取的时候平均爬取 100 条微博的所有数据需要长达 6 个多小时。因此，有必要将其扩展为分布式结构，利用集群（采用单Master-多Slave结构）提高并发能力，从而提高爬取性能。

大概思路如下：

-   scrapy 中的 scheduler 是一个 request 调度器，用以响应 engine 发出的指令而调度多个 request 给 downloader。spider 每生成一个 request 就会存入到 scheduler 的这个队列中。如果有相同的 request，scrapy 会自动去重。由于去重后的 request 之间是相互独立的，因此要提高并发能力，只需要将 scheduler 分离开，放到一个 master 中，多个 slave 每次访问这个 master 上 scheduler 中的队列即可。
-   可以利用 scrapy-redis 包实现这一点。它利用 redis 作为队列，实现了 scheduler 的功能。redis 是一个高性能的 key-value 数据库，而且 redis 客户可以方便地远程访问 redis 服务端。其具体使用只需要在项目中的 settings.py 中进行配置即可。
-   分布式爬虫一般都会碰到去重的问题。每个 request 在进入队列之前，都需要判断是否与已有的所有 request 重复。如果不重复则才放到队列里。由于这里用了 scrapy-redis 定义的 scheduler，因此也需要用 scrapy-redis 定义好的去重工具。同样在 settings.py 中进行配置。
-   爬取好的 item 存储到 Master 上的 PostgreSQL 的数据库里。

---

## 2 Slave 依赖环境

在 Linux, Mac OSX 下测试通过（Windows 没有测试，应该是可以的）。下面以 Ubuntu 为例搭建环境。

-   Python 3.5+

    `sudo apt-get install python3-dev`

    `sudo apt-get install python3-pip`
-   Python下的 scrapy包，requests包，rsa包，scrapy-redis包，PostgreSQL 在 Python3 下的驱动 psycopg2

    `sudo python3 -m pip install -U requests`

    `sudo python3 -m pip install -U rsa`

    `sudo apt-get install libxml2-dev libxslt1-dev libffi-dev libssl-dev`

    `sudo python3 -m pip install -U scrapy`
    
    `sudo python3 -m pip install -U scrapy-redis`

    `sudo apt-get install libpq-dev`

    `sudo python3 -m pip install -U psycopg2`

---

## 3 Master 依赖环境

同样以 Ubuntu 为例。

-   Python 3.5+

    `sudo apt-get install python3-dev`

    `sudo apt-get install python3-pip`
   
-   PostgreSQL

    `sudo apt-get install postgresql`

-   Redis

    `sudo apt-get install redis-server`

## 4 安装及运行

下载到各个 Slave 上：

`git clone https://github.com/cuckootan/DistributedWeiboSpider.git`

然后在 Master 上的 PostgreSQL 里用以存储爬取的数据库中创建各个表。创建完成后将这些表的名字写入到各个 Slave 中的 settings.py 中。

最后进入各个 Slave 的项目根目录，执行如下命令即可运行（前提是要对该项目配置完成，见下面）：

`scrapy crawl weibo`

>   本项目中的 scrapy 项目名为 **weibo**。

---

## 5 配置说明

1.  选用 Pycharm 作为开发及调试工具；
    
    打开 **Run -> Edit Configurations**，点击左上角的 **+** 添加配置信息。
    
    -   将 **Script** 字段填写为 **/usr/local/bin/scrapy**；
    -   将 **Script parameters** 字段填写为 **crawl weibo**；
    -   将 **Python interpreter** 字段填写为 python3 解释器的路径；
    -   将 **Working directory** 字段填写为该项目的根目录的路径。比如：**/home/username/Project/WeiboSpider**；
    -   取消 **Add content roots to PYTHONPATH** 以及 **Add source roots to PYTHONPATH**。
2.   配置 PostgreSQL 并建立数据库。打开 **/etc/postgresql/9.5/main/pg_hba.conf**，添加如下字段到更改用户权限的相应位置。
    
    **host    all    your_username    所有的 salve 所在的网段    trust**
3.  配置 Redis。打开 **/etc/redis/redis-conf**，注释掉 bind 所在行。
4.  程序中用到的所有配置都写在了项目中的 **settings.py** 里，因此将项目下载到本地后，只需配置更改其中的相应内容即可，无序修改其他源程序。
    主要包括：

    ```python
        # Your whole weibo username and password pairs.
        WEIBO_LOGIN_INFO_LIST = [('your username_1', 'your password_1'), ('your username_2', 'your password_2')]
        # Each name of tables can be defined here (each value of items). These keys are not changeable.
        TABLE_NAME_DICT = {
            'user_info': 'user_info_table_name',
            'follow': 'follow_table_name',
            'fan': 'fan_table_name',
            'post_info': 'post_info_table_name',
            'text': 'text_table_name',
            'image': 'image_table_name',
            'comment': 'comment_table_name',
            'forward': 'forward_table_name',
            'thumbup': 'thumbup_table_name'
        }
        
        # Replace default scheduler with scrapy-redis scheduler.
        SCHEDULER = 'scrapy_redis.scheduler.Scheduler'
        DUPEFILTER_CLASS = 'scrapy_redis.dupefilter.RFTDupeFilter'
        # If this value below is set to False, when spider is closed normally, all the data in redis will be cleared.
        SCHEDULER_PERSIST = True
        SCHEDULER_QUEUE_CLASS = 'scrapy_redis.queue.SpiderQueue'
        # Sometimes the queue is empty, but the task is not finisher. So you'd better set the value below.
        SCHEDULER_IDLE_BEFORE_CLOSE = 300

        REDIS_HOST = 'your redis host'
        REDIS_PORT = 'your redis port'

        # Your postgresql username.
        POSTGRESQL_USERNAME = 'your postgresql username'
        # Your postgresql password.
        POSTGRESQL_PASSWORD = 'your postgresql password'
        # Your postgresql databaes.
        POSTGRESQL_DATABASE = 'your database name'

        # The user id you want to crawl.
        CRAWLED_WEIBO_ID_LIST = ['123456789', '246812345']
    ```
    其中，各个表的所有列的字段及数据类型分别为（它们不能被改变，表名可以改变）：
    
    -   user_info 对应表的结构为： **(user_id varchar(20), user_name text, gender varchar(5), district text)**
    -   follow 对应表的结构为： **(user_id varchar(20), follow_list text[])**
    -   fan 对应表的结构为： **(user_id varchar(20), fan_list text[])**
    -   post_info 对应表的结构为： **(user_id varchar(20), post_id varchar(20), publist_time text)**
    -   text 对应表的结构为： **(user_id varchar(20), post_id varchar(20), text text)**
    -   image 对应表的结构为： **(user_id varchar(20), post_id varchar(20), image_list text[])**
    -   comment 对应表的结构为： **(user_id varchar(20), post_id varchar(20), comment_list json)**
    -   forward 对应表的结构为： **(user_id varchar(20), post_id varchar(20), forward_list json)**
    -   thumbup 对应表的结构为： **(user_id varchar(20), post_id varchar(20), thumbup_list json)**

    还有一些其他配置项，详见 settings.py。

## 6 数据的导出

进入 **setting.py** 中指定的数据库，对每个表执行如下命令：

`\copy table_name TO $ABSOLUTE_PATH`

其中，**$ABSOLUTE_PATH** 为每个表对应输出文件的 **绝对路径**。

对于表中 json 类型的字段，在输出到文件后用 python3 中的 json 包进行处理即可。

## TODO

-   添加用于实时查看爬虫信息的图形化界面（用 Graphite 实现）；

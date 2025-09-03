黑马点评+谷粒商城分布式高级篇

一、 redis基础：
1. cd /usr/local/src/redis-xx
2. 配置/etc/redis/redis.conf:
	bind 0.0.0.0
    daemonize yes
    requirepass 123123

    port 6379
    maxmemory 512mb
    dir .
    logfile "redis.log"
3. 命令：
sudo systemctl status redis-server
sudo systemctl start redis-server
sudo systemctl restart redis-server




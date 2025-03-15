Alpin Linux



服务自启



path: /etx/init.d/xray

```shell
#!/sbin/openrc-run

# 定义服务的名称变量
name=xray
pidfile=/run/xray.pid
cfgfile=/etc/xray/config.jsonc
command=/usr/bin/xray
command_args="-c $cfgfile"
command_background=true
command_user=root
require_files="$cfgfile"

# 提供服务的描述信息
description="Custom service for ${name}"

# 定义服务的依赖关系
depend() {
    # 指定该服务需要网络（net 服务）支持
    need net

    # after 后置强依赖条件
    # 指定该服务应在 sshd 服务之后启动
    after sshd

    # use 非强依赖
    ## 如果 目标服务存在且已运行，那么当前服务会优先使用它，目标服务不存在未运行，服务依然会启动
    # 指定该服务要在 logger 服务之后启动
    use logger
    # 指定该服务可以使用 audit
    use audit
    # 指定该服务可以使用 loadkeys
    use loadkeys

}

# 在启动服务之前的预处理函数
start_pre() {
    # 打印启动前的消息
    # 可以在这里添加任何启动服务之前的准备工作
    ebegin "Preparing to start ${name}"
}
```

```shell
rc-update add xray default

service xray start
```



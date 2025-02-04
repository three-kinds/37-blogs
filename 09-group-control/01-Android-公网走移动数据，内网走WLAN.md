# Android-公网走移动数据，内网走WLAN

* 在群控领域，很可能会出现每个设备都需要一个公网出口的场景；同时很多时候，我们不仅要访问公网，同时还要访问内网
* 宽带有限，策略路由很难尽情发挥
* 最简单的方式，是使用3G/4G路由，但这种方法会增加一些额外的成本

## 现状

* 手机关闭WLAN、打开移动数据开关，就可以实现使用移动数据网络上公网；但却无法访问内网
* 手机打开WLAN开关，就可以实现使用WLAN访问内网；但这时移动数据网络就无法使用了，陷入了两难境地 

## 解决思路

1. 手机设备类似一个有2张网卡的电脑，一个是移动数据网卡，一个是WLAN网卡
2. 通常这2个网卡都不是同时工作，需要让这2个网卡同时工作
3. 通过配置默认路由，让默认流量全部走移动数据网络
4. 通过配置静态路由，让指定内网网段走WLAN网络

## 核心难题

### 1. 如何让2个网卡同时工作

* 当前的手机系统，通常都支持让2个网卡同时工作
* 如MIUI：设置 -> WLAN -> 网络加速 | WLAN助理 -> 打开`数据加速` -> 加速模式`并发`

### 2. 如何配置默认路由

* 需要知道工作中的移动数据网卡`CCMNI_DEV`与其IP地址`CCMNI_IP`
* 通过就可以设置默认路由了`ip route add default via ${CCMNI_IP} dev ${CCMNI_DEV}`
* 测试`ip route get 114.114.114.114`，发现还是走的WLAN
* 推测与路由策略有关（ip rule），但我不懂android的默认路由策略，它有很多条
* 我仿照电脑的2条路由策略，配置了一下，发现启效了，如下（如有更好的办法，欢迎老哥指点）
* `ip rule add pref 9000 from all table main`
* `ip rule add pref 9001 from all table default`
* 无论是`ip route get 114.114.114.114`，还是使用网页访问IP发现都走的移动数据
* 需要root权限

### 3. 如何配置静态路由

* 添加普通的静态路由即可`ip route add 172.18.0/24 via 192.168.0.1 dev wlan0 proto static`
* 无论是`ip route get 172.18.0.100`，还是使用网页访问内网服务，发现都是通的
* 需要root权限

### 4. 网络可能会发生变化、特别是移动数据网络，如何及时更新路由

* 测试发现，当网络发生变化时、无论什么网络变化，路由策略都会自动更新
* 所以只要通过`ip monitor rule`去监控路由策略，当其发生变化时，就能及时更新路由
* 需要root权限

## 流程梳理

```shell
#!/system/bin/sh

DIR=${0%/*}
# 如果有这个文件，则开启debug模式，会输出日志到文件
FLAG_FILE="/data/local/tmp/ccmni-and-wlan.debug"
LOG_FILE="/data/local/tmp/ccmni-and-wlan.log"
DEBUG=false

if [ -r $FLAG_FILE ]; then
  DEBUG="true"
fi

# shell log
slog() {
    if [ $DEBUG = "true" ]; then
        echo "$(date '+%Y-%m-%d %H:%M:%S') $1" >> $LOG_FILE
    fi
}

LAST_WLAN0_DEV=""
update_wlan0_route() {
    WLAN0_DEV=$(ip -o -4 addr show | grep wlan0 | awk '{print $2}' | cut -d':' -f1)

    if [ "$WLAN0_DEV" == "$LAST_WLAN0_DEV" ]; then
        slog "wlan0没有变化"
    else
        if [ "$WLAN0_DEV" == "" ]; then
            slog "[关闭]wlan0"
        else
            slog "[打开]wlan0，配置内网相关静态路由"
            # todo: 添加内网相关静态路由
            # ip route add 172.18.0/24 via 192.168.0.1 dev wlan0 proto static
        fi
    fi

    LAST_WLAN0_DEV=$WLAN0_DEV    
}

LAST_CCMNI_DEV=""
LAST_CCMNI_IP=""
update_ccmni_route() {
    # 获取当前工作中的 ccmni 设备
    CCMNI_DEV=$(ip -o -4 addr show | grep ccmni | awk '{print $2}' | cut -d':' -f1)
    CCMNI_IP=""
    if [ -n "$CCMNI_DEV" ]; then
        CCMNI_IP=$(ip addr show $CCMNI_DEV | grep "inet " | awk '{print $2}' | cut -d/ -f1)
    fi

    if [[ "$CCMNI_DEV" == "$LAST_CCMNI_DEV" && "$CCMNI_IP" == "$LAST_CCMNI_IP" ]]; then
        slog "ccmni没有变化"
    else
        if [[ "$CCMNI_IP" == "" ]]; then
            slog "[关闭]ccmni"
        else
            slog "[打开]$CCMNI_DEV，配置相关路由"
            # 添加默认路由，指定设备
            ip route del default
            ip route add default via ${CCMNI_IP} dev ${CCMNI_DEV}
        fi
    fi

    LAST_CCMNI_DEV=$CCMNI_DEV
    LAST_CCMNI_IP=$CCMNI_IP
}

# 当只有wlan在工作时，则通过 lo 锁定默认路由，达到锁网效果
check_need_lock_network() {
    if [[ "$LAST_WLAN0_DEV" != "" && "$LAST_CCMNI_IP" == "" ]]; then
        if [[ "$(ip route get 8.8.8.8 | grep "dev lo src")" == "" ]]; then
            slog "[锁网]只有wlan0"
            ip route del default
            ip route add default dev lo
        fi
    fi
}

# 添加初始路由策略
ip rule add pref 9000 from all table main
ip rule add pref 9001 from all table default

# 无限循环，持续监控 ip rule 的变化
while true; do
    slog "正在检查 ip rule 变化..."
    # 使用 ip monitor 监听 rule 变化
    ip monitor rule | while read -r line; do
        # 检查是否匹配到关注的规则
        if  [[ "$line" == *"from all iif lo oif wlan0 uidrange 0-0 lookup"* ]] || [[ "$line" == *"from all iif lo oif ccmni"* ]]; then
            slog "匹配到规则: $line"
            update_wlan0_route
            update_ccmni_route
            # 可选：当不想让设备通过WLAN访问公网时启用
            check_need_lock_network
        fi
    done
    # 避免过多占用 CPU
    sleep 1
done

```

* 为了方便部署，可以将这个脚本封装为KernelSU/Magisk的模块

================================================================================
           家庭宽带公网访问方案 — 华硕路由器 + 阿里云 IPv6 DDNS
================================================================================

一、方案概述
────────────────────────────────────────────────────────────────

  适用于家庭宽带的公网访问方案，解决以下问题：
  - 运营商分配大内网 IPv4（多层 NAT），无公网 IPv4 地址
  - 利用 IPv6 公网地址实现外网访问家中设备
  - 运营商可能动态更换 IPv6 前缀，导致设备地址变化
  - 路由器重启后定时任务丢失

  核心流程：
  1. 脚本定时检测路由器及内网设备的 IPv6 地址
  2. 通过阿里云 DNS API 自动更新 AAAA 记录的解析地址
  3. 同时自动更新 ip6tables 防火墙规则（IPv6 下每个设备都有公网地址，防火墙至关重要）

  适用环境：华硕路由器（AsusWRT/Merlin 固件） + 阿里云域名


二、文件说明
────────────────────────────────────────────────────────────────

  文件名                                用途
  ────────────────────────────────────────────────────────────────
  aliddns_devices.conf                  多设备 DDNS 配置文件
  asus_single_device_ddns_update.txt    单设备 DDNS 更新脚本
  asus_multi_devices_ddns_update.txt    多设备 DDNS 更新脚本
  firewall_devices.conf                 防火墙设备端口配置文件
  firewall_rules_for_device.txt         ip6tables 防火墙规则管理脚本


三、前置准备
────────────────────────────────────────────────────────────────

  3.1 阿里云账号配置

      1. 登录阿里云控制台，在「AccessKey 管理」页面创建 AccessKey
         获取 AccessKeyId 和 AccessKeySecret
      2. 在阿里云 DNS 解析中添加你的域名（如 example.com）
      3. 将域名的 DNS 服务器修改为阿里云提供的 NS 地址

  3.2 路由器准备

      1. 路由器需刷入 Merlin（梅林）固件或官改固件（AsusWRT）
      2. 开启 SSH 访问（路由器管理页面 → 系统管理 → 服务 → 启用 SSH）
      3. 启用 IPv6（路由器管理页面 → IPv6 → 开启，连接类型选 Native 或 Passthrough）
      4. 确保路由器已安装 openssl（Merlin 固件自带）
      5. /jffs 分区需要可读写（需启用 JFFS 分区）

  3.3 网络环境确认

      1. 确认路由器 WAN 口能获取到公网 IPv6 地址
         ssh 登录路由器执行: ip -6 addr show | grep "scope global"
         如看到 2409: 或 2408: 开头的地址，说明已有公网 IPv6
      2. 确认内网设备也获取到了公网 IPv6 地址


四、单设备 DDNS 脚本 (asus_single_device_ddns_update.txt)
────────────────────────────────────────────────────────────────

  4.1 适用场景

      只需要将路由器的 IPv6 地址映射到一个子域名。

  4.2 配置修改

      编辑脚本，修改以下三个变量：

        AccessKeyId=""          # 填入阿里云 AccessKey ID
        AccessKeySecret=""      # 填入阿里云 AccessKey Secret
        DomainName=""           # 主域名，如 example.com
        SubDomain=""            # 子域名前缀，如 home（最终解析为 home.example.com）

  4.3 部署到路由器

      1. 将脚本内容上传到路由器: /jffs/scripts/aliddns.sh
      2. ssh 登录路由器，赋予执行权限:
           chmod +x /jffs/scripts/aliddns.sh
      3. 手动运行测试:
           /jffs/scripts/aliddns.sh
      4. 设置定时任务（每10分钟检查一次）:
           cru a aliddns "*/10 * * * * /jffs/scripts/aliddns.sh"

  4.4 工作原理

      1. 从路由器 nvram/ifconfig 获取当前 WAN 口的公网 IPv6 地址
      2. 与本地缓存对比，如果 IP 没变化则直接退出（不产生日志）
      3. 如果 IP 变化，调用阿里云 API 查询当前 AAAA 记录
      4. 根据查询结果自动决定创建新记录或更新已有记录
      5. 更新成功后写入本地缓存

  4.5 日志和缓存

      日志文件: /tmp/aliddns.log
      IP 缓存:  /tmp/aliddns_ip_cache
      查看日志: tail -20 /tmp/aliddns.log


五、多设备 DDNS 脚本 (asus_multi_devices_ddns_update.txt)
────────────────────────────────────────────────────────────────

  5.1 适用场景

      需要将多个设备（路由器 + 内网设备）的 IPv6 地址分别映射到不同子域名。

  5.2 配置修改

      编辑脚本，修改以下三个变量：

        AccessKeyId=""          # 填入阿里云 AccessKey ID
        AccessKeySecret=""      # 填入阿里云 AccessKey Secret
        DomainName=""           # 主域名，如 example.com
        CONFIG_FILE="/jffs/scripts/aliddns_devices.conf"  # 设备配置文件路径

  5.3 配置文件 (aliddns_devices.conf)

      格式: 设备描述|MAC地址|子域名

      规则:
      - 使用竖线 | 分隔三个字段
      - MAC 地址必须大写，用冒号分隔（如 00:11:22:33:44:55）
      - 路由器的 MAC 地址留空
      - 以 # 开头的行为注释
      - 子域名为实际完整解析的 RR，无需包含主域名部分

      示例:
        # 路由器自身
        主路由器||route

        # 内网设备
        台式电脑|00:11:22:33:44:55|pc
        NAS存储|AA:BB:CC:DD:EE:FF|nas
        树莓派|11:22:33:44:55:66|pi

      以上配置将产生:
        route.example.com  → 路由器 IPv6 地址
        pc.example.com     → 台式电脑 IPv6 地址
        nas.example.com    → NAS IPv6 地址
        pi.example.com     → 树莓派 IPv6 地址

  5.4 部署到路由器

      1. 将脚本上传到: /jffs/scripts/aliddns_multi.sh
      2. 将配置文件上传到: /jffs/scripts/aliddns_devices.conf
      3. ssh 登录路由器:
           chmod +x /jffs/scripts/aliddns_multi.sh
      4. 设置定时任务:
           cru a aliddns_multi "*/10 * * * * /jffs/scripts/aliddns_multi.sh"

  5.5 可用命令

      运行方式: /jffs/scripts/aliddns_multi.sh [命令]

        (无参数)    运行所有设备的 DDNS 检查
        list        列出配置文件中所有设备
        test <子域名>  单独测试某个子域名
        log         查看最近的运行日志
        config      显示当前配置文件内容

  5.6 内网设备 IPv6 地址获取方式

      脚本通过以下途径获取内网设备的 IPv6 地址:
      1. ARP 邻居表 (ip -6 neigh show) — 按 MAC 地址匹配
      2. DHCP 租约文件 (/var/lib/misc/dnsmasq.leases) — 备用查询
      3. 自动过滤掉 fe80: 开头的本地链路地址

  5.7 注意事项

      - 每个设备处理之间间隔 2 秒，避免触发阿里云 API 频率限制
      - IP 地址有缓存机制，未变化时不会调用 API，也不产生日志
      - 日志文件最大保留 100 行，超过会自动截断


六、防火墙规则管理脚本 (firewall_rules_for_device.txt)
────────────────────────────────────────────────────────────────

  6.1 为什么需要防火墙

      在 IPv6 网络中，每个设备都有独立的公网 IPv6 地址（不再像 IPv4 那样
      共享路由器的公网地址做 NAT）。这意味着如果没有防火墙，内网设备将
      直接暴露在公网上，存在严重安全隐患。本脚本的作用是:

      - 只对指定端口放行公网入站流量
      - 当设备 IPv6 地址变化时自动更新规则
      - 确保旧规则被清理，避免残留无效规则

  6.2 配置文件 (firewall_devices.conf)

      上传到: /jffs/scripts/firewall_devices.conf

      格式: 设备描述|MAC地址|端口列表

      端口规则:
      - 单个端口: 80、443、22
      - 端口范围: 9000-9010（使用连字符 -）
      - 多个端口用逗号分隔: 80,443,8080,9000-9010
      - 路由器的 MAC 地址留空
      - 以 # 开头的行为注释

      示例:
        # 路由器自身 (MAC地址留空表示路由器)
        主路由器||22,80,443,8080

        # 内网设备
        台式电脑|00:11:22:33:44:55|3389,8080,9000-9010
        NAS存储|AA:BB:CC:DD:EE:FF|22,80,443,5000-5010
        手机|11:22:33:44:55:66|9999

  6.3 部署

      1. 将脚本上传到: /jffs/scripts/aliddns_firewall.sh
      2. 将配置文件上传到: /jffs/scripts/firewall_devices.conf
      3. ssh 登录路由器:
           chmod +x /jffs/scripts/aliddns_firewall.sh

  6.4 可用命令

      运行方式: /jffs/scripts/aliddns_firewall.sh [命令]

        update         更新所有设备的防火墙规则（默认，仅在IP变化时更新）
        status         查看当前防火墙规则和IP缓存状态
        setup          设置定时任务（每10分钟）和开机自启动
        clean          清理本脚本创建的所有防火墙规则
        test <MAC>     测试指定 MAC 设备（如 test 00:11:22:33:44:55）
        config         显示当前配置文件
        example        生成配置文件示例
        log            查看最近 50 行日志
        help           显示帮助信息

  6.5 自动执行机制

      执行 setup 命令会自动完成:
      1. 通过 cru 添加每 10 分钟运行的 cron 定时任务
      2. 将更新命令追加到 /jffs/scripts/firewall-start（防火墙启动时执行）
      3. 将更新命令追加到 /jffs/scripts/services-start（延迟30秒执行，等网络就绪）

      注意: 路由器重启后，cron 定时任务会丢失（见第八节），但 firewall-start
      和 services-start 中的脚本会在重启后执行一次。

  6.6 防火墙规则说明

      每条规则都带有注释标记（ALIDDNS_FW_MAC_xxxxx），方便识别和管理。
      脚本自动添加的规则包括:
      - TCP 端口放行规则（按配置的端口/端口范围）
      - ICMPv6 放行（允许 ping）
      - 已建立连接和相关连接的放行规则

      可通过 status 命令查看当前生效的所有本脚本创建的规则。


七、推荐部署架构
────────────────────────────────────────────────────────────────

  完整的推荐部署方案如下：

  路由器文件清单:
    /jffs/scripts/aliddns_multi.sh          多设备 DDNS 脚本
    /jffs/scripts/aliddns_devices.conf      设备-子域名映射配置
    /jffs/scripts/aliddns_firewall.sh       防火墙规则管理脚本
    /jffs/scripts/firewall_devices.conf     设备-端口映射配置

  定时任务 (通过 cru 命令，注意重启后需重建):
    */10 * * * * /jffs/scripts/aliddns_multi.sh         # DDNS更新
    */5  * * * * /jffs/scripts/aliddns_firewall.sh update  # 防火墙更新

  开机启动 (services-start 或 firewall-start):
    /jffs/scripts/aliddns_firewall.sh update

  各脚本运行顺序:
  1. 路由器开机 → 获取 IPv6 地址
  2. 防火墙脚本先运行，打开必要的端口
  3. DDNS 脚本运行，将最新 IPv6 地址更新到阿里云 DNS
  4. 此后各脚本按照 cron 定时检查并更新


八、注意事项与常见问题（雷点）
────────────────────────────────────────────────────────────────

  8.1 内网设备的临时 IPv6 地址干扰

      问题: Windows / Linux 等设备默认开启隐私扩展，会生成随机临时 IPv6 地址。
            脚本可能获取到临时地址，而非稳定的 SLAAC/DHCPv6 地址。

      解决:
      - 确认脚本获取到的是正确的稳定地址
      - 可在路由器端将目标设备的 MAC 绑定到固定 IPv6 后缀
      - 或在设备端关闭 IPv6 隐私扩展:
        Windows: netsh interface ipv6 set privacy state=disable
        Linux:   sysctl net.ipv6.conf.all.use_tempaddr=0

  8.2 路由器重启后定时任务丢失

      问题: AsusWRT/Merlin 的 cru（cron）任务在重启后会丢失。

      解决:
      - 方式一: 使用 /jffs/scripts/services-start 脚本重新注册定时任务。
        在此脚本内容中加入:
          cru a aliddns "*/10 * * * * /jffs/scripts/aliddns_multi.sh"
          cru a aliddns_fw "*/5 * * * * /jffs/scripts/aliddns_firewall.sh update"
      - 方式二: 使用 init-start 脚本在系统启动阶段注册 cron（某些固件支持）
      - 务必执行 chmod +x /jffs/scripts/services-start

  8.3 IPv6 前缀飘移

      问题: 运营商可能不定期更换分配的 IPv6 前缀（如 2409:xxxx:xxxx::/64 的前 64 位
            变化），导致所有内网设备的 IPv6 地址同时变化。

      影响:
      - 所有 AAAA 记录需要批量更新
      - 所有防火墙规则需要重建

      缓解:
      - DDNS 脚本设置为较短的检查间隔（5-10 分钟）
      - 防火墙脚本也设置为较短间隔
      - 这两个脚本完美应对前缀变化场景

  8.4 API 调用限制

      问题: 阿里云 DNS API 有调用频率限制。

      解决:
      - 多设备脚本中每个设备处理间隔 2 秒
      - IP 缓存机制：IP 未变化时不会发起 API 调用
      - 建议 cron 间隔不要短于 3 分钟

  8.5 openssl 签名问题

      问题: 部分固件的 openssl 版本较老，可能不支持某些参数。

      解决:
      - Merlin 固件（384 以上版本）自带的 openssl 通常兼容
      - 如果签名失败，检查是否安装了完整版 openssl:
          opkg install openssl-util

  8.6 Debug 调试

      单设备脚本:
        ssh 登录路由器后直接执行 /jffs/scripts/aliddns.sh
        查看日志: tail -f /tmp/aliddns.log

      多设备脚本:
        /jffs/scripts/aliddns_multi.sh              # 全部运行
        /jffs/scripts/aliddns_multi.sh test nas     # 单独测试 nas 子域名
        /jffs/scripts/aliddns_multi.sh list         # 列出设备
        /jffs/scripts/aliddns_multi.sh log          # 查看日志

      防火墙脚本:
        /jffs/scripts/aliddns_firewall.sh status    # 查看规则状态
        /jffs/scripts/aliddns_firewall.sh log       # 查看日志
        /jffs/scripts/aliddns_firewall.sh test 00:11:22:33:44:55  # 测试单设备

  8.7 防火墙规则清理

      如果规则混乱需要重置:
        /jffs/scripts/aliddns_firewall.sh clean     # 清理所有本脚本创建的规则

      手动检查 ip6tables 规则:
        ip6tables -L INPUT -n --line-numbers
        ip6tables -L FORWARD -n --line-numbers


九、扩展：配合 Nginx Proxy Manager (NPM)
────────────────────────────────────────────────────────────────

  DDNS 解析到了 IPv6 地址后，直接访问还需要加端口号（如 https://nas.example.com:5001），
  不太美观。推荐在内网部署 Nginx Proxy Manager（Docker 或直接安装）:
  - 安装后暴露 80/443 端口
  - 在 NPM 中添加反向代理，指向内网设备的 IPv6 + 端口
  - 配合 Let's Encrypt 自动签发 SSL 证书
  - 最终实现: https://nas.example.com → 反向代理到 [设备IPv6]:5001
  - 此时防火墙只需放行 NPM 所在设备的 80/443 端口

================================================================================
  项目地址: 本仓库 / 家庭宽带公网访问 /
  适用平台: 华硕路由器 (AsusWRT / Merlin) + 阿里云域名
================================================================================

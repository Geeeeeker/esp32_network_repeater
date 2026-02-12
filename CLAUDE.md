# CLAUDE.md - 项目记忆上下文

## 项目概述

**ESP32 WiFi 中继器 (NAT Router)** - 基于 ESP32 的 WiFi 中继/NAT 路由器。设备同时运行 STA（客户端）和 AP（热点）模式，通过 NAT 将 AP 客户端的流量转发到上游 WiFi 网络，实现网络扩展和共享。

## 技术栈

- **芯片**: ESP32 / ESP32-S3 / ESP32-C3
- **框架**: ESP-IDF v5.4+
- **构建**: CMake 3.16+
- **语言**: C (ESP-IDF 风格)
- **网络**: lwIP (启用 NAPT)、esp_http_server
- **存储**: NVS Flash (非易失性存储)
- **OS**: FreeRTOS + pthread

## 目录结构

```
├── main/
│   ├── esp32_nat_router.c    # 主入口: WiFi初始化、NAT配置、事件处理、GPIO控制
│   ├── http_server.c         # HTTP服务器: 强制门户、DNS劫持、Web配置API
│   ├── pages.h               # 嵌入式HTML页面声明
│   └── cmd_decl.h            # 命令声明头文件
├── components/
│   ├── cmd_router/           # 路由器CLI命令 (set_sta, set_ap, portmap等)
│   │   ├── cmd_router.c      # 路由器命令实现
│   │   └── router_globals.h  # 全局变量和函数声明
│   ├── cmd_nvs/              # NVS存储操作命令
│   │   └── cmd_nvs.c         # NVS读写/擦除/列表命令
│   └── cmd_system/           # 系统命令 (restart, version, heap等)
│       └── cmd_system.c
├── www/
│   └── index.html            # 现代Web配置界面 (HTML/CSS/JS)
├── sdkconfig.defaults        # SDK默认配置 (NAPT/IP转发已启用)
├── partitions_example.csv    # Flash分区表
└── CMakeLists.txt            # 根构建配置
```

## 核心模块功能

### main/esp32_nat_router.c (~793行)
- `app_main()`: 主入口 - NVS初始化 → 加载配置 → WiFi初始化 → 启动线程 → NAT启用 → 控制台
- `wifi_init()`: 双模WiFi初始化 (APSTA模式), DHCP配置, 事件注册
- `wifi_event_handler()`: WiFi事件回调 - 连接/断连/IP获取处理
- `led_status_thread()`: LED状态指示灯线程 (GPIO44/ESP32-S3, GPIO2/ESP32)
- `boot_button_monitor_thread()`: BOOT按键监控 - 长按5秒恢复出厂
- `factory_reset()`: 清除NVS所有配置并重启
- `add_portmap()` / `del_portmap()`: 端口映射管理

### main/http_server.c (~658行)
- `start_webserver()`: 启动HTTP服务器, 注册URI处理器
- `config_post_handler()`: POST /config - JSON配置提交处理
- `captive_portal_handler()`: 强制门户重定向
- `connectivity_check_handler()`: Android/iOS/Windows连接检测响应
- `dns_server_task()`: DNS服务器 - 所有域名解析到192.168.4.1
- `modern_index_handler()`: 提供嵌入式index.html页面

### components/cmd_router/cmd_router.c (~621行)
- `set_sta()`: 设置上游WiFi (SSID/密码/企业认证)
- `set_ap()`: 设置热点 (SSID/密码)
- `set_ap_mac()` / `set_sta_mac()`: MAC地址配置
- `get_config_param_str()` / `get_config_param_int()`: NVS配置读取
- `preprocess_string()`: URL编码/十六进制解码

## NVS存储 (命名空间: "esp32_nat")

| 键 | 类型 | 默认值 | 用途 |
|---|------|--------|------|
| ssid | String | "" | 上游WiFi SSID |
| passwd | String | "" | 上游WiFi密码 |
| ent_username | String | "" | 企业WiFi用户名 |
| ent_identity | String | "" | 企业WiFi身份 |
| static_ip | String | "" | STA静态IP |
| subnet_mask | String | "" | 子网掩码 |
| gateway_addr | String | "" | 网关地址 |
| ap_ssid | String | "ESP32_NAT_Router" | 热点SSID |
| ap_passwd | String | "12345678" | 热点密码 |
| ap_ip | String | "192.168.4.1" | AP IP地址 |
| mac / ap_mac | Blob(6) | NULL | STA/AP MAC地址 |
| portmap_tab | Blob | 空 | 端口映射表 |
| lock | String | "0" | 配置锁定标志 |

## 关键全局变量 (router_globals.h)

```c
extern char* ssid, *passwd, *ap_ssid, *ap_passwd;
extern char* ent_username, *ent_identity;
extern char* static_ip, *subnet_mask, *gateway_addr;
extern uint16_t connect_count;  // AP已连接客户端数
extern bool ap_connect;         // STA是否已连接上游
extern uint32_t my_ip, my_ap_ip;
extern esp_netif_t* wifiAP, *wifiSTA;
```

## HTTP API 端点

| 路径 | 方法 | 功能 |
|------|------|------|
| `/` | GET | 主配置页面 (index.html) |
| `/config` | POST | JSON配置提交 |
| `/generate_204` | GET | Android连接检测 → 302重定向 |
| `/hotspot-detect.html` | GET | iOS连接检测 → 302重定向 |
| `/ncsi.txt` | GET | Windows连接检测 → 302重定向 |
| `/*` | GET | 通配符 → 强制门户重定向 |

## 硬件引脚

- **LED**: GPIO44 (ESP32-S3) / GPIO2 (ESP32)
- **BOOT按键**: GPIO0 (低电平有效, 长按5秒恢复出厂)

## 构建命令

```bash
idf.py build              # 编译
idf.py -p [PORT] flash    # 烧录
idf.py monitor             # 串口监视
idf.py build flash monitor # 一键编译烧录监视
```

## 应用启动流程

1. NVS初始化 → 加载所有配置参数
2. WiFi APSTA模式初始化 → 注册事件处理器
3. 启动LED线程 + BOOT按键监控线程
4. 启用NAT (ip_napt_enable) → 应用端口映射规则
5. 启动HTTP服务器 + DNS服务器 (强制门户)
6. 初始化CLI控制台 → 进入命令循环

## 配置更新流程

Web界面提交 → POST /config (JSON) → 解析参数 → 调用 set_sta()/set_ap() → 写入NVS → 5秒后自动重启

## 注意事项

- SDK配置中 `CONFIG_LWIP_IPV4_NAPT=y` 和 `CONFIG_LWIP_IP_FORWARD=y` 是NAT功能的关键开关
- AP默认网段 192.168.4.x/24, 最大8个客户端连接
- Flash分区: nvs(24KB) + phy_init(4KB) + factory(1200KB) + storage(740KB)
- 企业WiFi使用 esp_eap_client API
- Web前端使用 Fetch API 发送 JSON, 内嵌在固件中 (通过 CMake embed 机制)

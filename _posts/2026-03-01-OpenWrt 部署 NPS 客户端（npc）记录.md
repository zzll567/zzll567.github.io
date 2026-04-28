# OpenWrt 部署 NPS 客户端（npc）记录

## 一、目标

在路由器上部署 **npc 客户端**，连接公网服务器 **nps 服务端**，实现：

- 内网穿透
- 远程桌面
- SSH访问
- NAS访问

系统：

- 路由系统：OpenWrt
- 穿透工具：NPS

------

# 二、网络架构

```
公网服务器
     │
     │ nps服务端
     │ 8024
     │
OpenWrt路由器
     │
     │ npc客户端
     │
家庭内网设备
(Windows / NAS / SSH)
```

访问路径：

```
外网 → 服务器 → npc → 内网设备
```

------

# 三、确认路由器架构

登录路由器：

```
ssh root@192.168.31.1
```

查看CPU架构：

```
uname -m
```

输出：

```
mips
```

说明：

```
MIPS架构CPU
```

------

# 四、下载 npc

进入程序目录：

```
cd /usr/bin
```

下载客户端：

```
wget https://github.com/ehang-io/nps/releases/download/v0.26.10/linux_mipsle_client.tar.gz
```

解压：

```
tar -zxvf linux_mipsle_client.tar.gz
```

赋权：

```
chmod +x npc
```

运行测试：

```
./npc
```

------

# 五、第一个坑：Illegal instruction

运行结果：

```
Illegal instruction
```

原因：

官方编译版本使用 **硬浮点（hardfloat）**，而很多 MIPS 路由器不支持。

解决方法：

重新编译 npc：

```
GOMIPS=softfloat
```

------

# 六、使用 WSL 编译 npc

使用：

Windows Subsystem for Linux

安装 Go：

```
sudo apt update
sudo apt install golang -y
```

下载源码：

```
git clone https://github.com/ehang-io/nps.git
cd nps
```

编译：

```
GOOS=linux GOARCH=mipsle GOMIPS=softfloat go build -o npc ./cmd/npc
```

生成：

```
npc
```

------

# 七、上传到 OpenWrt

上传：

```
scp npc root@192.168.31.1:/usr/bin/
```

赋权：

```
chmod +x /usr/bin/npc
```

测试：

```
npc
```

成功输出：

```
the version of client is 0.26.10
```

------

# 八、配置 npc

创建配置目录：

```
mkdir -p /usr/bin/conf
```

编辑配置：

```
vi /usr/bin/conf/npc.conf
```

配置内容：

```
server_addr=服务器IP:8024
vkey=你的VKEY
conn_type=tcp
auto_reconnection=true
```

示例：

```
server_addr=8.162.8.67:8024
vkey=123456
conn_type=tcp
auto_reconnection=true
```

------

# 九、启动 npc

前台运行：

```
npc -config=/usr/bin/conf/npc.conf
```

成功日志：

```
Successful connection with server
```

------

# 十、后台运行 npc

## 方法1：简单后台运行

```
npc -config=/usr/bin/conf/npc.conf &
```

查看进程：

```
ps | grep npc
```

示例：

```
20603 root /usr/bin/npc -config=/usr/bin/conf/npc.conf
```

------

## 方法2：nohup后台运行

防止 SSH 断开：

```
nohup npc -config=/usr/bin/conf/npc.conf > /tmp/npc.log 2>&1 &
```

说明：

```
> /tmp/npc.log   保存日志
2>&1             合并错误输出
```

------

# 十一、查看 npc 日志

实时查看：

```
tail -f /tmp/npc.log
```

查看最后日志：

```
tail -n 50 /tmp/npc.log
```

查看全部日志：

```
cat /tmp/npc.log
```

------

# 十二、进程管理

查看 npc：

```
ps | grep npc
```

查看端口：

```
netstat -ntlp | grep npc
```

停止 npc：

```
killall npc
```

重启 npc：

```
killall npc
npc -config=/usr/bin/conf/npc.conf &
```

------

# 十三、开机自启配置

创建启动脚本：

```
vi /etc/init.d/npc
```

写入：

```
#!/bin/sh /etc/rc.common

START=99

start() {
    echo "Starting npc..."
    /usr/bin/npc -config=/usr/bin/conf/npc.conf > /tmp/npc.log 2>&1 &
}

stop() {
    echo "Stopping npc..."
    killall npc
}
```

赋权：

```
chmod +x /etc/init.d/npc
```

启用：

```
/etc/init.d/npc enable
```

启动：

```
/etc/init.d/npc start
```

停止：

```
/etc/init.d/npc stop
```

------

# 十四、NPS 面板配置

登录 Web 面板：

```
http://服务器IP:8080
```

新增隧道。

远程桌面示例：

| 参数       | 示例          |
| ---------- | ------------- |
| 类型       | tcp           |
| 目标IP     | 192.168.1.100 |
| 目标端口   | 3389          |
| 服务器端口 | 40001         |

------

# 十五、远程桌面连接

Windows 打开：

Remote Desktop Connection

输入：

```
服务器IP:40001
```

即可远程访问内网电脑。

------

# 十六、常见问题排查

### 1 npc 未连接服务器

检查：

```
server_addr
vkey
```

查看日志：

```
tail -f /tmp/npc.log
```

------

### 2 Illegal instruction

原因：

```
CPU架构不匹配
```

解决：

```
GOMIPS=softfloat
```

------

### 3 端口被占用

错误：

```
bind: address already in use
```

解决：

```
killall npc
```

------

### 4 默认连接 127.0.0.1

修改配置：

```
server_addr=公网服务器IP
```

------

# 十七、最终运行结构

```
公网服务器
    │
    │ nps
    │
    │ 8024
    │
OpenWrt (npc)
    │
    │
内网设备
```

实现：

- 内网穿透
- 远程桌面
- NAS访问
- SSH访问

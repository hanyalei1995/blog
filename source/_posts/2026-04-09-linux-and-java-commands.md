---
title: Linux 与 Java 常用命令速查手册
date: 2026-04-09 10:00:00
tags:
  - Linux
  - Java
  - 命令行
  - 运维
categories:
  - 后端开发
description: 整理常用的 Linux 系统命令与 Java 开发命令，涵盖文件操作、进程管理、网络排查及 JVM 调优等场景，适合开发与运维日常查阅。
top_img:
cover:
password:
message:
---

日常开发和运维中，熟练使用 Linux 命令和 Java 相关命令能大幅提升效率。本文整理了最常用的命令，按场景分类，方便快速查阅。

---

## 一、Linux 常用命令

### 文件与目录操作

```bash
# 查看目录内容（详细信息 + 隐藏文件）
ls -la

# 切换目录
cd /var/log

# 创建目录（含父级）
mkdir -p /data/app/logs

# 删除文件 / 目录
rm -f file.txt          # 强制删除文件
rm -rf /tmp/test/       # 递归删除目录（慎用）

# 复制 / 移动
cp -r src/ dst/
mv old_name.txt new_name.txt

# 查找文件
find /home -name "*.log" -mtime -7   # 7天内修改过的 .log 文件

# 查看文件内容
cat app.log
tail -f app.log          # 实时追踪日志
tail -n 100 app.log      # 最后 100 行
head -n 50 app.log       # 前 50 行

# 文件内容搜索
grep -rn "ERROR" /var/log/
grep -i "exception" app.log | grep -v "NullPointer"
```

### 文本处理

```bash
# 统计行数 / 字数
wc -l file.txt

# 排序与去重
sort file.txt | uniq -c | sort -rn

# 截取字段（以冒号为分隔符，取第一列）
cut -d: -f1 /etc/passwd

# 流式编辑（替换字符串）
sed -i 's/old/new/g' config.txt

# 按列处理
awk '{print $1, $3}' access.log
awk -F',' '$3 > 100 {print $0}' data.csv
```

### 进程与系统资源

```bash
# 查看进程
ps aux | grep java
pgrep -a java

# 实时监控
top
htop                     # 需安装，更友好

# 杀进程
kill -9 <PID>
pkill -f "myapp.jar"

# 查看端口占用
ss -tulnp | grep 8080
lsof -i :8080

# 磁盘使用
df -h                    # 各挂载点磁盘使用
du -sh /var/log/*        # 各子目录大小

# 查询目录下占用磁盘大小（排序）
du -sh /data/*           # 查看 /data 下各子目录/文件大小
du -sh *                 # 查看当前目录下各条目大小
du -ah /data/ | sort -rh | head -20   # 按大小降序，显示最大的20个
du -h --max-depth=1 /var/log/         # 只展开一层目录

# 内存使用
free -h

# 系统负载与运行时间
uptime
vmstat 1 5               # 每秒刷新，共5次
```

### 网络排查

```bash
# 测试连通性
ping -c 4 google.com
traceroute google.com

# 查看网络连接
netstat -antp
ss -s                    # 连接统计汇总

# 下载文件
curl -O https://example.com/file.zip
wget https://example.com/file.zip

# 发送 HTTP 请求
curl -X POST http://localhost:8080/api/user \
  -H "Content-Type: application/json" \
  -d '{"name":"tom"}'

# 查看 DNS 解析
nslookup example.com
dig example.com
```

### 权限与用户

```bash
# 修改文件权限
chmod 755 script.sh
chmod -R 644 /data/files/

# 修改文件所有者
chown user:group file.txt

# 查看当前用户
whoami
id

# 切换用户
su - deploy
sudo systemctl restart nginx
```

### 压缩与解压

```bash
# tar.gz
tar -czvf archive.tar.gz /data/app/
tar -xzvf archive.tar.gz -C /tmp/

# zip
zip -r archive.zip /data/app/
unzip archive.zip -d /tmp/
```

---

## 二、Java 常用命令

### 编译与运行

```bash
# 编译单个文件
javac Hello.java

# 编译并指定输出目录
javac -d out/ src/**/*.java

# 运行 class 文件
java -cp out/ com.example.Main

# 运行 jar 包
java -jar myapp.jar

# 指定 JVM 堆内存
java -Xms512m -Xmx2g -jar myapp.jar

# 传入系统参数
java -Dspring.profiles.active=prod -jar myapp.jar
```

### JVM 调优参数

```bash
# 堆内存设置
-Xms512m          # 初始堆大小
-Xmx2g            # 最大堆大小
-Xss256k          # 每个线程栈大小

# GC 选择
-XX:+UseG1GC                    # G1 垃圾回收器（JDK 9+ 默认）
-XX:+UseZGC                     # ZGC（低延迟，JDK 15+）
-XX:+UseShenandoahGC            # Shenandoah GC

# GC 日志（JDK 9+）
-Xlog:gc*:file=gc.log:time,uptime:filecount=5,filesize=20m

# OOM 时自动 dump
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/tmp/heapdump.hprof
```

### JDK 自带诊断工具

```bash
# 查看所有 Java 进程
jps -lv

# 查看 JVM 参数与系统属性
jinfo -flags <PID>
jinfo -sysprops <PID>

# 堆内存统计（按类排名）
jmap -histo <PID> | head -30

# 生成堆转储文件（用于 MAT 分析）
jmap -dump:format=b,file=heap.hprof <PID>

# 查看线程栈（排查死锁/CPU 飙高）
jstack <PID>
jstack <PID> | grep -A 20 "BLOCKED"

# 实时 JVM 性能监控
jstat -gcutil <PID> 1000        # 每秒输出 GC 统计
jstat -gc <PID> 2000 10         # 每2秒，共10次

# 图形化监控工具（需图形界面）
jconsole
jvisualvm
```

### Maven 常用命令

```bash
# 清理 + 打包（跳过测试）
mvn clean package -DskipTests

# 运行测试
mvn test

# 安装到本地仓库
mvn install

# 查看依赖树
mvn dependency:tree

# 分析未使用的依赖
mvn dependency:analyze

# 指定模块构建
mvn -pl module-name -am clean package
```

### Gradle 常用命令

```bash
# 构建
./gradlew build

# 跳过测试
./gradlew build -x test

# 查看所有任务
./gradlew tasks

# 查看依赖
./gradlew dependencies

# 运行指定任务
./gradlew bootRun
```

### JAR 包操作

```bash
# 查看 jar 包内容
jar -tf myapp.jar | grep "Main"

# 解压 jar 包
jar -xf myapp.jar

# 查看 MANIFEST 信息
unzip -p myapp.jar META-INF/MANIFEST.MF

# 查看 jar 中某个 class 的反编译（需 javap）
jar -xf myapp.jar com/example/Main.class
javap -c -verbose com/example/Main.class
```

---

## 三、实战：线上 Java 服务排查流程

当线上 Java 服务出现问题时，通常按以下步骤排查：

```bash
# 1. 找到 Java 进程 PID
jps -l
# 或
ps aux | grep java

# 2. 查看 CPU / 内存占用
top -p <PID>

# 3. 查看 GC 情况（频繁 Full GC？）
jstat -gcutil <PID> 1000

# 4. 线程分析（CPU 飙高时定位热点线程）
# 先找 CPU 最高的线程 TID
top -H -p <PID>
# 将十进制 TID 转十六进制
printf '%x\n' <TID>
# 在 jstack 中搜索对应 nid
jstack <PID> | grep -A 30 "nid=0x<hex>"

# 5. 内存溢出时 dump 堆
jmap -dump:format=b,file=/tmp/heap.hprof <PID>
# 用 MAT 或 VisualVM 打开分析
```

---

## 总结

| 场景 | 推荐命令 |
|------|----------|
| 实时日志 | `tail -f` |
| 端口占用 | `ss -tulnp` / `lsof -i` |
| 文本搜索 | `grep -rn` |
| Java 进程 | `jps -lv` |
| GC 监控 | `jstat -gcutil` |
| 线程分析 | `jstack` |
| 堆分析 | `jmap -dump` + MAT |
| 打包 | `mvn clean package` |

熟悉这些命令，是每个后端开发和运维人员的必备技能。建议在本地或测试环境多练习，遇到问题时才能从容应对。

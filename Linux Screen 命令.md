# Linux Screen 命令

## 概念

scree Linux 下的离线运行程序,在终端下运行定时任务时,需要一直开着终端才能运行,已关闭终端,程序也会自动关闭,Screen类似一个后台运行的命令,即使终端关闭,程序也会继续运行

## 安装

Ubuntu下安装Screen,只需要一行命令

```bash
sudo apt-get install screen
```

## 语法

````bash
screen [-AmRvx -ls -wipe][-d <作业名称>][-h <行数>][-r <作业名称>][-s ][-S <作业名称>]
````

## 参数


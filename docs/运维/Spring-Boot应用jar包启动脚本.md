#!/bin/bash
#这里可替换为你自己的执行程序，其他代码无需更改
APP_NAME=iota.jar

#检查程序是否在运行
is_exist(){
	pid=`ps -ef|grep $APP_NAME|grep -v grep|awk '{print $2}' `
	#如果不存在返回1，存在返回0
	if [ -z "${pid}" ]; then
		return 1
	else
		return 0
	fi
}

#启动方法
start(){
	is_exist
	if [ $? -eq "0" ]; then
		echo "${APP_NAME} is already running. pid=${pid} ."
	else
		nohup java -jar -server -Xms128m -Xmx128m -XX:SurvivorRatio=6 -XX:+UseG1GC -XX:MaxGCPauseMillis=40 -XX:InitiatingHeapOccupancyPercent=35 -XX:+DisableExplicitGC -Dspring.profiles.active=dev $APP_NAME>/data/log/iota.log 2>&1 &
		echo "${APP_NAME} start success"
	fi
}

#停止方法
stop(){
	is_exist
	if [ $? -eq "0" ]; then
		echo "kill ${APP_NAME} pid"
		kill -9 $pid
	else
		echo "${APP_NAME} is not running"
	fi
}

#输出运行状态
status(){
	is_exist
	if [ $? -eq "0" ]; then
		echo "${APP_NAME} is running. Pid is ${pid}"
	else
		echo "${APP_NAME} is NOT running."
	fi
}

#重启
restart(){
	stop
	start
}

# 执行重启方法
restart



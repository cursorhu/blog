---
title: Linux Shell笔记：实现芯片固件的批量编译
date: 2020-08-12 14:58:37
tags: shell
categories: linux
---

# 背景
某软件有不同的配置参数，实现不同功能版本的编译
批量测试需要批量编译各种版本，实现方式为：
1.将编译参数组合，生成大量配置文件
2.编译过程遍历这些配置文件，依次编译对应版本
3.有参数加入，修改，删除，只需要更新这些配置文件
如何实现这些配置文件的更新？

# 实例
某芯片的Firmware批量编译实现：
Firmware代码为C, 配置参数用宏实现，后缀为.def
目录结构如下

|--project_folder
　　|--config
　　　|--build.def
　　　|--defs
　　　　　|--1.def 2.def ... n.def
　　|--src
　　|--Makefile
　　|--build_All.sh
　　|--update.sh
    
## 批量编译脚本
批量编译脚本如下
基本过程：
1.依次拷贝def文件夹中的每个def，替换默认的build.def
2.编译，接受所有编译参数
3.拷贝编译输出的image到包含git tag, def名，时间等信息的文件夹

    #!/bin/bash
    
    echo "Batch build support args:"
    echo "1. functin version:"
    echo "verargs=mp_fpga"
    echo "verargs=mpw_asic"
    echo "2. boot debug:"
    echo "bootargs=debug"
    
    OUTPUT=batch_build_$1$2
    
    mkdir -p ${OUTPUT}
    rm -rf ./batch_build_*
    
    build_time=`date +%Y%m%d%H%M%S`
    
    #commit_id=`git rev-parse HEAD`
    
    tag_name=`git describe --exact-match --tags 2>/dev/null`
    
    if [ -z "${tag_name}" ]; then
    	tag_name="NO_TAG"
    fi
    
    mv ./config/build.def ./config/build.def.bak 
    
    for file in `ls ./config/defs/*.def`;
    do
    	file_name=${file##*/}
    	config_name=${file_name%.def}
    	
    	cp -rf ${file} ./config/build.def
    	make clean
    	make -j4 $@
    	#mv ./build/image ./batch_build/${tag_name}_${config_name}_time_${build_time}_cid_${commit_id}
    	mkdir -p ./${OUTPUT}/${tag_name}_${config_name}_time_${build_time}
    	mv ./build/image/* ./${OUTPUT}/${tag_name}_${config_name}_time_${build_time}
    done
    
    mv ./config/build.def.bak ./config/build.def

def文件内的.def文件即各种参数配置文件，.def文件名即参数功能的别名组合
例如：

    CQ_emmc_two_card_enhance_hs400_always_l0_MSIx_dllphase14_tuning_on.def

对应的内容是：
    
    /*0: Non-CQ mode 1:CQ mode enable*/
    #define BB_CQ_MODE_ENABLE 1
    /*the card number support emmc#0:0 emmc#1:1 two card:2*/
    #define BB_CARD_NUMBER 2
    /*0:legacy 1:High Speed 50MHz 2:HS200 3:HS400 4:Enhance HS400*/
    #define BB_MAX_TRANSFER_MODE 4
    /* power mode management: 0: Low Power 1:Balance 2:High Performance 3:Direct HP 4:Always L0*/
    #define POWER_MANAGEMENT_MODE 4
    /* INT_MODE: 0:MSI_X 1:INTx 2:MSI_MULTIPLE 3:MSI_SINGLE */
    #define INT_MODE 0
    /* The selection of DLL PHASE COUNT is 11 or 14 */
    #define DLL_phase_cnt 14
    /* 0: fixed output phase  1: auto output tuning */
    #define AUTO_OUTPUT_TUNING 1

## 批量编辑配置文件
配置文件def有两个属性
1.文件名每个词代表一个功能，各词用下划线“_"分隔
2.内部用宏定义实现功能配置，宏定义的值要和外部文件名匹配

基于以上属性，编辑脚本需求为：
1.新增：增加一个宏定义，并增加对应的功能缩写到文件名
2.修改：修改一个已存在的宏定义，并修改对应的功能缩写到文件名
3.删除：删除一个已存在的宏定义，并删除对应的功能缩写到文件名
4.其他功能，如直接删除含某缩写的文件，备份原配置文件

shell实现为update.sh,如下:

    #!/bin/bash
    
    DEFS_PATH="./config/defs"
    DEFS_BACKUP_PATH="./config/defs_backup"
    DEFS_TEMP_PATH="./config/defs_temp"
    
    if [ $# -lt 1 ];then
    		echo "usage: ./update.sh [option] [args]"
    
    		echo "example 0:"
    		echo "		backup defs files:"
    		echo "		./update.sh -bf"
    		echo ""
    
    		echo "example 1:"
    		echo "		add a macro name and macro value to defs, and add file postfix:"
    		echo "		./update.sh -b"
    		echo "		./update.sh -a balance POWER_MANAGEMENT_MODE 1 "
    		echo "		./update.sh -a high_performance POWER_MANAGEMENT_MODE 2 "
    		echo "		add other values..."
    		echo ""
    
    		echo "example 2:"
    		echo "		update a macro name and macro value to defs, and update file postfix:"
    		echo "		./update.sh -u balance lowpower POWER_MANAGEMENT_MODE 1 "
    		echo ""
    
    		echo "example 3:"
    		echo "		delete a macro name and macro value of defs, and delete file postfix:"
    		echo "		./update.sh -d lowpower POWER_MANAGEMENT_MODE"
    		echo ""
    
    		echo "example 4:"
    		echo "		delete target files:"
    		echo "		./update.sh -df lowpower"
    		echo ""
    
    		echo "example 5:"
    		echo "		clean backup defs files:"
    		echo "		./update.sh -cf"
    		echo ""
    
    		exit;
    	fi
    
    if [ $1 = "-bf" ];then #backup defs
    	mkdir -p $DEFS_BACKUP_PATH
    	mv $DEFS_PATH/*.def $DEFS_BACKUP_PATH
    
    elif [ $1 = "-cf" ];then #clear backup defs
    	rm -rf $DEFS_BACKUP_PATH


​    
​    #add a macro name and macro value to defs, and add file postfix
​    elif [ $1 = "-a" ];then
​    
    	if [ $# != 4 ];then
    		echo "usage: ./update.sh -a FILE_POSTFIX MACRO_NAME MACRO_VALUE"
    		exit;
    	fi
    
    	mkdir -p $DEFS_TEMP_PATH && cp -rf $DEFS_BACKUP_PATH/*.def $DEFS_TEMP_PATH
    	
    	FILE_POSTFIX=$2
    	MACRO_NAME=$3
    	MACRO_VALUE=$4
    	# sed -i makes change on original file, otherwise on stream
    	# xargs transfer multiple output from stream to multiple args to sed
    	find ${DEFS_TEMP_PATH} -name '*.def' | xargs sed -i '$a\#define\ '"$MACRO_NAME"'\ '"$MACRO_VALUE"''
    	
    	for file in `ls ${DEFS_TEMP_PATH}/*.def`
    	do
    	 mv $file `echo $file | sed 's/\(.*\)\(\..*\)/\1_'"$FILE_POSTFIX"'\2/g'`
    	done
    	
    	cp -rf $DEFS_TEMP_PATH/*.def $DEFS_PATH
    	rm -rf $DEFS_TEMP_PATH
    
    #update a macro name and macro value to defs, and update file postfix
    elif [ $1 = "-u" ];then
    
    	if [ $# != 5 ];then
    		echo "usage: ./update.sh -u ORIGIN_POSTFIX UPDATED_POSTFIX MACRO_NAME MACRO_UPDATED_VALUE"
    		exit;
    	fi
    
    	ORIGIN_POSTFIX=$2
    	UPDATED_POSTFIX=$3
    	MACRO_NAME=$4
    	MACRO_UPDATED_VALUE=$5
    
    	#replace all lines that pattern matches $MACRO_NAME
    	find ${DEFS_PATH} -name '*.def' | grep $ORIGIN_POSTFIX | xargs sed -i 's/.*'"$MACRO_NAME"'.*/#define\ '"$MACRO_NAME"'\ '"$MACRO_UPDATED_VALUE"'/g'
    	#update file postfix
    	for file in `ls ${DEFS_PATH}/*$ORIGIN_POSTFIX*.def`
    	do
    	 	mv $file `echo $file | sed 's/'"$ORIGIN_POSTFIX"'/'"$UPDATED_POSTFIX"'/g'`
    	done


​    
​    #delete a macro name and macro value of defs, and delete file postfix
​    elif [ $1 = "-d" ];then
​    	
    	if [ $# != 3 ];then
    		echo "usage: ./update.sh -d DELETE_POSTFIX MACRO_NAME"
    		exit;
    	fi
    
    	DELETE_POSTFIX=$2
    	MACRO_NAME=$3
    	#delete all lines that contain $MACRO_NAME
    	find ${DEFS_PATH} -name '*.def' | grep $DELETE_POSTFIX | xargs sed -i '/'"$MACRO_NAME"'/d'
    	#delete file postfix
    	for file in `ls ${DEFS_PATH}/*.def`
    	do
    	 	mv $file `echo $file | sed 's/_'"$DELETE_POSTFIX"'//g'`
    	done
    
    #delete target file by postfix
    elif [ $1 = "-df" ];then
    	
    	if [ $# != 2 ];then
    		echo "usage: ./update.sh -df DELETE_POSTFIX"
    		exit;
    	fi
    
    	DELETE_POSTFIX=$2
    	rm -f ${DEFS_PATH}/*$DELETE_POSTFIX*.def
    
    fi

**重点讲下其中的几个sed和文件操作**
1.多个文件，每个文件最后一行追加内容

    find ${DEFS_TEMP_PATH} -name '*.def' | xargs sed -i '$a\#define\ '"$MACRO_NAME"'\ '"$MACRO_VALUE"''


 - xargs的作用： find <path> -name <string> 输出的是多个文件名，不能直接传给sed， **xargs将多个文件名转化成多个参数**，每个参数是一个文件名，sed可以接收
 - -i的作用：sed本身是在文件的拷贝上操作，不直接在文件本身修改，-i（insert）使sed的操作在文件上生效，**如果不加-i，源文件不会被修改**
 - $：表示最后一行，sed 'a\string'是基础格式
 - 注意sed怎么用带空格和变量的字符串：**空格用转义'\ '表示，变量是单引号内加双引号**，即'"\$ARG"'

2.找到包含字符串A的所有文件，替换内容：将字符串B替换为C

    #replace all lines that pattern matches $MACRO_NAME
    find ${DEFS_PATH} -name '*.def' | grep $ORIGIN_POSTFIX | xargs sed -i 's/.*'"$MACRO_NAME"'.*/#define\ '"$MACRO_NAME"'\ '"$MACRO_UPDATED_VALUE"'/g'

 - find | grep 是常用套路，先找在过滤，注意find -name 可以使用\*， grep不要用\*，否则grep会把它当成要匹配的字符
 - sed 's/stringB/stringC'是基础格式，g表示全局，注意要-i

3.找到包含字符串A的所有文件，删除内容：包含字符串B的行

    #delete all lines that contain $MACRO_NAME
    find ${DEFS_PATH} -name '*.def' | grep $DELETE_POSTFIX | xargs sed -i '/'"$MACRO_NAME"'/d'

4.对多个文件的文件名，增加，修改，删除特定字符串

    #在原文件名最后，后缀之前，增加字符串“_$FILE_POSTFIX”
    for file in `ls ${DEFS_TEMP_PATH}/*.def`
    	do
    	 mv $file `echo $file | sed 's/\(.*\)\(\..*\)/\1_'"$FILE_POSTFIX"'\2/g'`
    	done
    
    #替换文件名中匹配的字符
    for file in `ls ${DEFS_PATH}/*$ORIGIN_POSTFIX*.def`
    do
     	mv $file `echo $file | sed 's/'"$ORIGIN_POSTFIX"'/'"$UPDATED_POSTFIX"'/g'`
    done
    
    #删除文件名指定字符
    for file in `ls ${DEFS_PATH}/*.def`
    do
     	mv $file `echo $file | sed 's/_'"$DELETE_POSTFIX"'//g'`
    done

 - for < args > in \`ls < path >\`是把多个文件名依次写入变量args的常用操作，类似于xargs，不过是用循环每次处理一个文件名变量
 - mv \$file \`echo \$file | sed 's/stringA/stringB/g'`实际是两个步骤：先用echo把当前处理的文件名传给sed, sed处理完的输出文件名，作为mv的目标文件名，覆盖了原文件
 - 注意，文件名xargs传给sed,处理的是文件内容，for-in-do一个个mv，才是处理文件名本身

# 相关文章
[使用 sed 命令查找和替换文件中的字符串的 16 个示例](https://linux.cn/article-11367-1.html)
[sed引入变量的几种方法](https://blog.csdn.net/weixin_40572607/article/details/90812959)
[sed 批量替换文件内容](https://blog.csdn.net/elong490/article/details/52587171)

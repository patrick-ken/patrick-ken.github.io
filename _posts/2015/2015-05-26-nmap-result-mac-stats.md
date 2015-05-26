---
layout: post
title: "nmap 统计各厂商终端数的shell脚本"
description: ""
category: 编程开发
tags: [nmap, shell, mac]
---

上篇Blog《nmap nmap-mac-prefixes 生成脚本》解决了nmap的MAC地址数据库过旧的问题，只要使用nmap之前先更新一下`nmap-mac-prefixes`，设备厂商识别不完整的问题就基本被干掉了。但是，还有两个问题需要解决：


1. **不能上网的环境下识别结果不完整** 例如死话访问不了IEEE官网（咳咳，我这是指网速慢，不要老想着#长#城）。
2. **每个厂商统计敲一遍命令，过于繁琐** 我关注的厂商无非就那几个，每次扫描敲一遍是很蛋疼的事情，必须简化操作！


如上两个问题，可以浓缩成一个方案解决：实现一个脚本，输入多个MAC地址和需要统计的厂商，输出各厂商的MAC地址数量。需求明确，动手实现，不难，几十行代码的事，脚本（`mac_stat.sh`）代码如下：

    #!/bin/sh
    #
    # brief  : a simple device statistics of nmap scan result.
    # author : patrick_ken#163.com
    # create : 2015/05/26
    #
    
    NMAP_SCAN_RESULT="$1"
    NMAP_MAC_PREFIXES="nmap-mac-prefixes"
    MAC_SUM_LIST="mac_sum.txt"
    MAC_DETECTED_LIST="mac_detected.txt"
    
    
    cat $NMAP_SCAN_RESULT | grep 'MAC Address' | awk '{ print $3 }' | awk -F ':' '{ print $1$2$3 }' > $MAC_SUM_LIST
    
    cat $MAC_SUM_LIST | xargs -i grep '{}' $NMAP_MAC_PREFIXES > $MAC_DETECTED_LIST

    MAC_SUM=`cat $MAC_SUM_LIST | wc -l`
    
    # trim prefix/suffix blanks
    MAC_SUM=$(echo $MAC_SUM)
    
    DETECTED=`cat $MAC_DETECTED_LIST | wc -l`
    
    DETECTED=$(echo $DETECTED)
    
    let "UNKNOWN=MAC_SUM-DETECTED"
    
    echo "[+] "
    echo "[+] $MAC_SUM total, $DETECTED detected, $UNKNOWN unknown."
    echo "[+] "
    
    printf "[+] %-10s %8s %8s \n" "Vendor" "Number" "Percent"
    echo "[+] ----------------------------------"
    
    
    VENDOR_LIST="APPLE SAMSUNG XIAOMI HUAWEI MEIZU BBK OPPO NOKIA LENOVO ZTE HTC GIONEE VIVO INTEL YULONG SMARTISAN"
    
    for v in $VENDOR_LIST
    do
        NUM=`grep -i "$v" $MAC_DETECTED_LIST | wc -l`
        let "PCT=($NUM*100)/MAC_SUM"
        printf "[+] %-10s %8s %8s %%\n" "$v" "$NUM" "$PCT"
    done
    
    echo "[+] "
    echo "[+] ----------------------------------"

使用步骤很简单，先将nmap扫描结果保存成文件，然后执行`mac_stat.sh <NMAP_SCAN_RESULT>`即可。

    $ nmap -sP 192.168.1.0/24 > nmap_result.txt
    $ mac_stats.sh nmap_result.txt
    
统计结果输出：

    $ sh mac_stat.sh nmap_result.txt
    [+]
    [+] 274 total, 273 detected, 1 unknown.
    [+]
    [+] Vendor       Number  Percent
    [+] ----------------------------------
    [+] APPLE           123       44 %
    [+] SAMSUNG          36       13 %
    [+] XIAOMI           24        8 %
    [+] HUAWEI           21        7 %
    [+] MEIZU             6        2 %
    [+] BBK               7        2 %
    [+] OPPO              7        2 %
    [+] NOKIA             3        1 %
    [+] LENOVO            3        1 %
    [+] ZTE               5        1 %
    [+] HTC               3        1 %
    [+] GIONEE            2        0 %
    [+] VIVO              4        1 %
    [+] INTEL             6        2 %
    [+] YULONG            3        1 %
    [+] SMARTISAN         1        0 %
    [+]
    [+] ----------------------------------

这是根据我被困合肥机场时候的一次扫描结果生成的统计信息，可以看到：当时总共有274台终端接入机场的Wi-Fi网络，其中apple终端多达123台，占比44%；第二到四名分别是三星（13%），小米（8%）和华为（7%）。

-DONE-

---
layout: post
title: "nmap nmap-mac-prefixes 生成脚本"
description: ""
category: 
tags: [shell nmap mac]
---
### 事情背景

5月中旬到合肥出差，返程时因天气原因被困于新桥机场。百无聊赖之际，对机场的Wi-Fi简单分析了一番，包括Wi-Fi设备提供商、无线信号、Portal论证和在线客户端信息（嗯，足可见当时是多少无聊）。在线客户端统计初始是使用手机的 Fing 应用，结果发现速率太慢且较多终端无法识别出来。因此直接打开老ThinkPad祭出nmap大法，速率问题倒是解决了，但仍然存在较多MAC地址无法精确识别：

    MAC Address: 4C:21:D0:6E:5A:A8 (Sony Mobile Communications AB)
    Nmap scan report for 196.170.0.16
    Host is up (0.0020s latency).
    MAC Address: A4:3D:78:4E:FC:AA (Guangdong Oppo Mobile Telecommunications)
    Nmap scan report for 196.170.0.18
    Host is up (0.020s latency).
    MAC Address: 88:53:95:06:EC:7A (Apple)
    Nmap scan report for 196.170.0.26
    Host is up (0.12s latency).
    MAC Address: 70:14:A6:E5:03:74 (Unknown)
    Nmap scan report for 196.170.0.29
    Host is up (0.011s latency).
    
因为刚扫描完飞机就开始登机（17点的机票，22点才登机，我也是醉了），所以当时没来得及解决这个“部分MAC无法识别”问题。本周末终于有了点空闲时间，可以尝试解决一下。

### 初步分析

MAC具有全球唯一性，根据它可以精确识别出其设备厂商，这已是路人皆知，就不废话了。“部分MAC无法识别”问题就基本可定位为nmap的MAC地址数据库没有更新到最新版本所导致。直接进入nmap安装目录，就发现MAC地址数据库就静静躺在那了：nmap-mac-prefixes 。

    # $Id: nmap-mac-prefixes 33507 2014-08-13 21:18:10Z dmiller $ generated with make-mac-prefixes.pl
    # Original data comes from http://standards.ieee.org/regauth/oui/oui.txt
    # These values are known as Organizationally Unique Identifiers (OUIs)
    # See http://standards.ieee.org/faqs/OUI.html
    # We have added a few unregistered OUIs at the end.
    000000 Xerox
    000001 Xerox
    000002 Xerox
    ...

数据库格式简单粗暴，由多行 `<mac-prefix vendor>` 组成，例如 `88:53:95:06:EC:7A` 取其高3个字节得到 `885395` ， 匹配到行 `885395 Apple`，因此 `88:53:95:06:EC:7A` 这台设备是 apple 公司生产的终端设备。 `# ...`行为注释行，说明了本文件的生成过程：通过一个 `make-mac-prefiex.pl` perl脚本，将 http://standards.ieee.org/regauth/oui/oui.txt 文件转换生成了 `nmap-mac-prefixes`。

> MAC地址是由IEEE机构统一分配的，IEEE将已分配的MAC地址公布在 http://standards.ieee.org/regauth/oui/oui.txt 。

我们获取了一份最新的oui.txt，发现其最近一次更新是 2015-05-24 ，而 nmap 的 `nmap-mac-prefixes` 则是 2014-08-13 就生成了的。因此问题原因就确定了，部分MAC无法识别是由于 `nmap-mac-prefixes` 所用的oui.txt并非是最新的，导致最分配的MAC地址无法识别。解决方案也非常简单，根据最新的oui.txt重新生成`nmap-mac-prefixes`即可。较可惜的是，注释中提到的 `make-mac-prefixes.pl` 脚本并没有包含在 nmap 源码包或工程中，因此自己操刀写一个！

### 解决步骤

oui.txt内容如下：

      Generated: Sun, 24 May 2015 05:00:02 -0400

      OUI/MA-L			Organization
      company_id			Organization
                    Address
      
      
      00-00-00   (hex)		XEROX CORPORATION
      000000     (base 16)		XEROX CORPORATION
                    M/S 105-50C
                    800 PHILLIPS ROAD
                    WEBSTER NY 14580
                    UNITED STATES

      00-00-01   (hex)		XEROX CORPORATION
      000001     (base 16)		XEROX CORPORATION
                    ZEROX SYSTEMS INSTITUTE
                    M/S 105-50C 800 PHILLIPS ROAD
                    WEBSTER NY 14580
                    UNITED STATES
      ...
                    
和 nmap-mac-prefixes 的内容对比一下，很容易发现：我们只需要将 `000000     (base 16)		XEROX CORPORATION` 这些行抽取出来，稍作转换即可生成`nmap-mac-prefixes`了。这个转换较为简单，无需出动perl或python等重型武器，用`sed/awk/grep`等小刀即可。

**[setp1]** 执行 `grep '(base 16)'` ，筛选出我们需要处理的行。
 
    ken@PC-KEN /f/project/nmap_mac_gen
    $ cat oui.txt | grep '(base 16)' > mac_base16.txt
    
    ken@PC-KEN /f/project/nmap_mac_gen
    $ cat mac_base16.txt | head -n 5
      000000     (base 16)          XEROX CORPORATION
      000001     (base 16)          XEROX CORPORATION
      000002     (base 16)          XEROX CORPORATION
      000003     (base 16)          XEROX CORPORATION
      000004     (base 16)          XEROX CORPORATION
      
    ken@PC-KEN /f/project/nmap_mac_gen
    $

**[setp2]** 执行`sed 's/^\s\+//'` ，去除每一行的首空格（left trim）。

    ken@PC-KEN /f/project/nmap_mac_gen
    $ cat mac_base16.txt | sed 's/^\s\+//' > mac_base16_left_trim.txt
    
    ken@PC-KEN /f/project/nmap_mac_gen
    $ cat mac_base16_left_trim.txt | head -n 5
    000000     (base 16)            XEROX CORPORATION
    000001     (base 16)            XEROX CORPORATION
    000002     (base 16)            XEROX CORPORATION
    000003     (base 16)            XEROX CORPORATION
    000004     (base 16)            XEROX CORPORATION
    
    ken@PC-KEN /f/project/nmap_mac_gen
    $


**[setp3]** 执行 `sed 's/\s\+(base 16)\s\+/ /'` ， 使用一个空格替换掉中间的`(base 16)`及其左右空格(和tab)，得到格式完全一致的`nmap-mac-prefixes`。

    ken@PC-KEN /f/project/nmap_mac_gen
    $ cat mac_base16_left_trim.txt | sed 's/\s\+(base 16)\s\+/ /' > mac_base16_finnal.txt
    
    ken@PC-KEN /f/project/nmap_mac_gen
    $ cat mac_base16_finnal.txt | head -n 5
    000000 XEROX CORPORATION
    000001 XEROX CORPORATION
    000002 XEROX CORPORATION
    000003 XEROX CORPORATION
    000004 XEROX CORPORATION
    
    ken@PC-KEN /f/project/nmap_mac_gen
    $

**[setp4]** 添加虚拟网卡的MAC地址前缀。

    ken@PC-KEN /f/project/nmap_mac_gen
    $ cat nmap-mac-prefixes | tail -n 5
    FCFFAA Ieee Registration Authority  - Please see MAL Public Listing for More Information.
    525400 QEMU Virtual NIC
    B0C420 Bochs Virtual NIC
    DEADCA PearPC Virtual NIC
    00FFD1 Cooperative Linux virtual NIC
    
    ken@PC-KEN /f/project/nmap_mac_gen
    $ echo '525400 QEMU Virtual NIC' >> mac_base16_finnal.txt
    
    ken@PC-KEN /f/project/nmap_mac_gen
    $ echo 'B0C420 Bochs Virtual NIC' >> mac_base16_finnal.txt
    
    ken@PC-KEN /f/project/nmap_mac_gen
    $ echo 'DEADCA PearPC Virtual NIC' >> mac_base16_finnal.txt
    
    ken@PC-KEN /f/project/nmap_mac_gen
    $ echo '00FFD1 Cooperative Linux virtual NIC' >> mac_base16_finnal.txt
    
    ken@PC-KEN /f/project/nmap_mac_gen
    $ cat mac_base16_finnal.txt | tail -n 5
    FCFFAA IEEE REGISTRATION AUTHORITY  - Please see MAL public listing for more information.
    525400 QEMU Virtual NIC
    B0C420 Bochs Virtual NIC
    DEADCA PearPC Virtual NIC
    00FFD1 Cooperative Linux virtual NIC
    
    ken@PC-KEN /f/project/nmap_mac_gen
    $

**[setp5]** 替换 nmap 目录下的`nmap-mac-prefixes` ， 大功告成。

    ken@PC-KEN /f/project/nmap_mac_gen
    $ cp mac_base16_finnal.txt "/d/Program Files (x86)/Nmap/nmap-mac-prefixes"
    
    ken@PC-KEN /f/project/nmap_mac_gen

### 遗留问题

**上述的分解动作可以使用管道合并起来，更为简单高效。**
  

    # PWD = /d/Program Files (x86)/Nmap/
    -------------------------------------------------
    $ wget http://standards.ieee.org/regauth/oui/oui.txt
    $ cat oui.txt | grep '(base 16)' | sed 's/^\s\+//' | sed 's/\s\+(base 16)\s\+/ /' > mac_base16_finnal.txt
    $ cat nmap-mac-prefixes | tail -n 4 | xargs -i echo '{}' >> mac_base16_finnal.txt
    $ cp mac_base16_finnal.txt nmap-mac-prefixes

**是否可根据MAC前缀识别出设备类型？** 

例如给定一个 apple MAC地址，是否能识别它是iPhone，还是iPad或MacBook？

**给出一份设备的MAC地址表，直接计算出各厂商的设备数量**

有了这个支持，`nmap-mac-prefixes`是否是最新的版本就不再是问题了：我们可以先将扫描结果保存成文件，后续通过分析文件来得到想要的统计信息。




---
date: 2024-11-08 17:14:10
title: DHCP 支持 PXE 启动的基本条件
tags:
  - "Linux"
  - "PXE"
  - "DHCP"
draft: false
---

这篇笔记用来记录在虚拟机环境下，如何实现 DHCP 服务支持 PXE 启动。

<!--more-->

```bash

                                       (@@) (  ) (@)  ( )  @@    ()    @     O     @     O      @
                                  (   )
                              (@@@@)
                           (    )

                         (@@@)
                       ====        ________                ___________
                   _D _|  |_______/        \__I_I_____===__|_________|
                    |(_)---  |   H\________/ |   |        =|___ ___|      _________________
                    /     |  |   H  |  |     |   |         ||_| |_||     _|                \_____A
                   |      |  |   H  |__--------------------| [___] |   =|                        |
                   | ________|___H__/__|_____/[][]~\_______|       |   -|                        |
                   |/ |   |-----------I_____I [][] []  D   |=======|____|________________________|_
                 __/ =| o |=-~O=====O=====O=====O\ ____Y___________|__|__________________________|_
                  |/-=|___|=    ||    ||    ||    |_____/~\___/          |_D__D__D_|  |_D__D__D_|
                   \_/      \__/  \__/  \__/  \__/      \_/               \_/   \_/    \_/   \_/

```

一般的 DHCP 协商过程保持着以下四个步骤：

1. client 发出 DHCP DISCOVER 寻找网络中的 DHCP 服务器，这个数据包应该是通过广播发送的。
2. server 接收到 DISCOVER 数据包后，同样以广播形式回复 DHCP OFFER 数据包。
3. client 接收到 OFFER 数据包，确定提供了这个服务提供了需要的信息后，应该发出 DHCP REQUEST 进行 IP 申请，同样是通过广播发送，因为此时仍未获取到 IP 地址。
4. server 接收到 REQUEST 数据包后，将正确的 IP 信息以 DHCP ACK 的数据包格式通过广播发送到请求方，等待 client 接收完成后，就能够以这个 IP 进行通信。

在 PXE 启动中， Proxy DHCP 可以在不改变原有 DHCP 服务配置的基础上，通过额外运行新的 DHCP 服务，补充原有服务中无法提供的 PXE 启动信息。

根据 Intel 的文档 Preboot Execution Environment (PXE) Specification ， Proxy DHCP 可以分成两种情况：

- DHCP 和 Proxy DHCP 共同运行在同一台机器上，此时 DHCP 应该保持在 67 端口进行监听，而 Proxy DHCP 则应该在 4011 端口进行监听。
- DHCP 和 Proxy DHCP 分别运行在不同的机器上，此时它们都应该在 67 端口进行监听，而 Proxy DHCP 还应该额外在 4011 端口进行监听。

实际上的关键在于 DHCP OFFER 数据包中的 DHCP Options ，对应的文档中也有着 `The PXE client knows to interrogate the Proxy DHCP service because the DHCPOFFER from the DHCP service contains an Option #60 "PXEClient" tag without corresponding Option #43 tags or a boot file name.` 的相关记录。

从上述的文档记录理解，涉及到三个重要 Option ，分别是 `Option #60 Vendor Class Identifier` ， `Option #43 Vendor-Specific Infomation` ，而关于 `boot file name` ，这里指的应该是 DHCP 报文中的 file 字段，但在当前字段被占用的情况下可以使用 `Option #67 Bootfile Name` 来替代。

根据实际的抓包结果和对应的文档，当 client 进行 PXE 启动时，首先发出的 DISCOVER 数据包中必定带有 `Option #60 Vendor Class Identifier` ，以标识自身要求获取更多的启动信息。此时任何的 DHCP 服务都可以回复 OFFER 数据包，是否在其中加入 `Option #60 Vendor Class Identifier` 则尤为关键。

假定 DHCP 和 Proxy DHCP 共同运行在同一台机器上，按照文档描述，当 DHCP 服务只设置 `Option #60 Vendor Class Identifier` 的时候，因为没有额外的 `Option #43 Vendor-Specific Infomation` 和 Bootfile Name 信息，除了默认向 DHCP 服务所在的 67 端口发起申请 IP 地址的 REQUEST 广播请求，还会在成功申请到 IP 地址之后额外向这台机器的 4011 端口发起新的 REQUEST 请求。

假定 DHCP 和 Proxy DHCP 分别运行在不同的机器上，理论上这时应该会接收到多个 OFFER 数据包，那么如何区别哪个数据包是 Proxy DHCP 发出的，就要看其中哪个设置 `Option #60 Vendor Class Identifier` 来确定，除了默认向 DHCP 服务所在的 67 端口发起申请 IP 地址的 REQUEST 广播请求，还会在成功申请到 IP 地址之后额外向设置了 `Option #60 Vendor Class Identifier` 的 Proxy DHCP 服务所在的 4011 端口发起新的 REQUEST 请求。

所以基本上可以认为设置了 `Option #60 Vendor Class Identifier` 的服务，总会是 PXE client 请求启动信息的目标。

在对比了 VMware ESXi 和 Oracle VM VirtualBox 两者的实际表现后，基本上都可以验证这个结论。

首先这里的条件是 Proxy DHCP 服务所在的 67 端口返回的 OFFER 数据包必定带有 TFTP 服务器地址和 Bootfile Name 信息，其中：

- TFTP 服务器地址使用 DHCP 报文中的 siaddr 字段填充。
- Bootfile Name 信息使用 DHCP 报文中的 file 字段或者 `Option #67 Bootfile Name` 填充。

可以参考这部分来自 iPXE 的处理逻辑：

```C
// https://github.com/ipxe/ipxe/blob/master/src/net/udp/dhcp.c
static int dhcp_has_pxeopts ( struct dhcp_packet *dhcppkt ) {

    /* Check for a next-server and boot filename */
    if ( dhcppkt->dhcphdr->siaddr.s_addr &&
         ( dhcppkt_fetch ( dhcppkt, DHCP_BOOTFILE_NAME, NULL, 0 ) > 0 ) )
        return 1;

    /* Check for a PXE boot menu */
    if ( dhcppkt_fetch ( dhcppkt, DHCP_PXE_BOOT_MENU, NULL, 0 ) > 0 )
        return 1;

    return 0;
}

static void dhcp_request_rx ( struct dhcp_session *dhcp,
                  struct dhcp_packet *dhcppkt,
                  struct sockaddr_in *peer, uint8_t msgtype,
                  struct in_addr server_id,
                  struct in_addr pseudo_id ) {
    /* ............................... */
    /* Perform ProxyDHCP if applicable */
    if ( dhcp->proxy_offer /* Have ProxyDHCP offer */ &&
         ( ! dhcp->no_pxedhcp ) /* ProxyDHCP not disabled */ ) {
        if ( dhcp_has_pxeopts ( dhcp->proxy_offer ) ) {
            /* PXE options already present; register settings
             * without performing a ProxyDHCPREQUEST
             */
            settings = &dhcp->proxy_offer->settings;
            if ( ( rc = register_settings ( settings, NULL,
                       PROXYDHCP_SETTINGS_NAME ) ) != 0 ) {
                DBGC ( dhcp, "DHCP %p could not register "
                       "proxy settings: %s\n",
                       dhcp, strerror ( rc ) );
                dhcp_finished ( dhcp, rc );
                return;
            }
        } else {
            /* PXE options not present; use a ProxyDHCPREQUEST */
            dhcp_set_state ( dhcp, &dhcp_state_proxy );
            return;
        }
    }

    /* Terminate DHCP */
    dhcp_finished ( dhcp, 0 );
}
```

Oracle VM VirtualBox 使用了 82540em 的网络适配器，实际上则使用的是 iPXE boot firmware ，只需要满足上述条件，无须额外监听 4011 端口就已经可以正常获取到 Bootfile 。

VMware ESXi 使用的是 E1000e 的网络适配器，具体的 firmware 不甚明确，可以看到带有 VMware 和 Intel 的 Copyright 信息。这里则必须明确设置 OFFER 数据包中 `Option #43 Vendor-Specific Infomation, Suboption #6 PXE_DISCOVERY_CONTROL` 的 bit 3 为 1 ，才能正常获取到 Bootfile ，否则会有 `PXE-E55: ProxyDHCP service did not reply to request on port 4011` 的报错信息。这就说明了即使已经获取到前述条件，这里的 client 依然需要向 4011 端口发起 REQUEST 请求。

根据 Intel 的文档，设置 PXE_DISCOVERY_CONTROL 的 bit 3 则是为了允许 client 直接使用 OFFER 阶段的 Bootfile Name ，并且禁用 PXE discovery 。这里推测是 VMware ESXi 对于是否发起 ProxyDHCPREQUEST 的判断条件不同于 iPXE ，如果没有填充任何 `Option #43` 信息则进入 ProxyDHCPREQUEST 状态。

但如果已经填充了 `Option #43 Vendor-Specific Infomation` ，并且没有明确禁用 PXE discovery ，那么则需要将服务器地址信息额外填充到 `Option #43 Vendor-Specific Infomation` 的子选项当中，包括了 `Suboption #7 DISCOVERY_MCAST_ADDR` ， `Suboption #8 PXE_BOOT_SERVERS` ， `Suboption #9 PXE_BOOT_MENU` ， `Suboption #10 PXE_MENU_PROMPT` 等选项，当这些信息正确填充到 ACK 信息后，客户端可以根据他们执行 PXE discovery 。

以现有的情况来看，使用 `Option #43 Vendor-Specific Infomation` 字段禁用 PXE discovery 并且满足携带两个基本的启动信息的情况下，应该可以满足大多数虚拟化场景的 PXE 启动。

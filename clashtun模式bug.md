## 记一次clash tun模式与oppo互联冲突焦头烂额



codex cli使用ccswitch路由方式老是传输失败

想使用clash tun转发后使用ccswitch直连模式，结果打开节点全超时，

网上找许多教程，结果全无法解决，gpt排查也无法定位到，

最终在https://github.com/clash-verge-rev/clash-verge-rev/issues/6990#issuecomment-4486093363

发现解决方法，

oppo互联的服务带了一个**wintun.dll**是罪魁祸首！！！！

卸载重启，tun模式正常^_^

后面询问gpt原因:

1. ```
   `wintun.dll` 是创建、操作 Windows 虚拟网卡的组件，让程序能够收发 IP 数据包。它本身并不代表 VPN，也不是只能由一个软件使用。两个程序带有同名 DLL，并不能证明发生了 DLL 文件冲突。([wintun.net](https://www.wintun.net/))
   
   结合你电脑之前的日志，最符合现象的机制是：
   
   1. **OPPO 互联建立虚拟网络通道**，其网络配置与 Clash TUN 同时存在。
   2. **Clash 开启 TUN 后接管系统流量**，涉及自动路由、出口网卡选择和 DNS 接管。([wiki.metacubex.one](https://wiki.metacubex.one/config/inbound/tun/))
   3. 如果两套配置相互干扰，Clash 发往节点或 DNS 服务器的流量可能又进入自己的 TUN，形成回环。你之前日志中的 `reject loopback connection`、`dns resolve failed`，正符合这种情况。其他用户也记录过 OPPO 互联、WLAN 转发与 Mihomo 回环同时出现的案例。
   ```

   尚不能精确到 `wintun.dll` 的某个版本或函数




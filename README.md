# config-codex
一、让远程服务器走本地代理
1.配置本地的.ssh/config文件：

'''
Host <随便起>
    HostName <远程主机ip名>
    User <远端用户名>
    # **注意**：这个xxx是什么要问gpt，跟着他的指引做
    IdentityFile C:/Users/<你的Windows用户名>/.ssh/id_xxx
    ServerAliveInterval 60
    ServerAliveCountMax 3
    # 关键改动：允许转发失败时连接继续，这样能并发开多个窗口/项目
    ExitOnForwardFailure no
    # 建立（或复用）远端 -> 本地的反向代理通道
    # **注意**：确认本地代理端口号，远程端口号也可以自定义
    RemoteForward 127.0.0.1:7890 127.0.0.1:7890
'''
    
2.编辑远程主机的~/.bashrc，把代理设置添加进去：

# === SSH 反向隧道本地代理(默认开启) ===
'''
proxy_on() {
  export HTTP_PROXY="http://127.0.0.1:7890"
  export HTTPS_PROXY="http://127.0.0.1:7890"
  export ALL_PROXY="http://127.0.0.1:7890"
  export NO_PROXY="127.0.0.1,localhost,*.local,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16"
}
proxy_off() {
  unset HTTP_PROXY HTTPS_PROXY ALL_PROXY NO_PROXY
}
# 只有当 127.0.0.1:7890 存在时自动开启（即隧道已建立）
if bash -c 'exec 3<>/dev/tcp/127.0.0.1/7890' 2>/dev/null; then
  exec 3>&-
  proxy_on
else
  proxy_off
fi
# 手动切换命令：proxy_on / proxy_off
'''
3.生效：source ~/.bashrc
4.验证：

'''
# 应显示你本地代理出口的公网 IP，表示走代理成功
curl -s https://ipinfo.io/ip
# 看到 7890 在监听也表示隧道正常
ss -lnpt | grep 7890
# 查看环境变量（应已设置）
env | grep -E 'HTTP_PROXY|HTTPS_PROXY|ALL_PROXY|NO_PROXY'
#
curl -I --connect-timeout 10
'''
以上就配置好了远程的终端代理，并且可以开多个ssh窗口。
不过，如果你想让插件也走代理，比如codex插件，那还需要对本地的vsc做额外的如下配置：
打开本地的vsc，按住ctrl+shift+p，输入Preferences: Open User Settings (JSON)。然后添加：

'''
{
  // 让远程扩展宿主进程自带代理环境变量（对所有远程窗口/项目生效）
  "remote.SSH.remoteEnv": {
    "HTTP_PROXY":  "http://127.0.0.1:7890",
    "HTTPS_PROXY": "http://127.0.0.1:7890",
    "ALL_PROXY":   "http://127.0.0.1:7890",
    "NO_PROXY":    "127.0.0.1,localhost,*.local,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16"
  },

  // 让 Vsc/扩展的 HTTP 请求也遵循代理（配合上面的环境变量）
  "http.proxy": "http://127.0.0.1:7890",
  "http.proxySupport": "on",
  "http.systemCertificates": true
}
'''
然后就可以了，如果不行的话，重新加载一下窗口。

二、远程服务器登录codex
先在本地登录，然后找到本地.codex目录以及服务器.codex目录，把如图选中的本地文件上传上去。建议上传之前重启电脑，然后先在本地退出重新登录codex之后再做。
<img width="1686" height="619" alt="image" src="https://github.com/user-attachments/assets/b35277a4-acd1-41bf-806b-6a3297c13a8f" />

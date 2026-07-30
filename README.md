# Server-network-ssh---codex
远程服务器通过本地代理上网配置说明

[] 基于vscode，其他的原理一样

背景与目标
远程开发服务器（10.6.36.210）无法直接访问外网，但本地电脑有可用的代理（127.0.0.1:7897）。通过 SSH 远程转发与 VSCode 设置，实现：
- VSCode 远程端可下载扩展、更新等
- 服务器终端可手动开启/关闭代理，让 curl、pip 等工具也能临时访问外网
  
整体数据流
远程服务器程序（VSCode / curl / pip）
        ↓
  http://127.0.0.1:9999  （代理请求）
        ↓
  SSH 远程转发隧道（加密）
        ↓
  本地 127.0.0.1:7897 （本地代理软件，如 Clash）
        ↓
  外网

各环节配置

1. 本机代理软件
- 监听端口：7897
- 提供 HTTP 代理 协议（非 SOCKS5）
- 需保持运行状态
  
2. SSH 配置（本地vscode的插件Remote - SSH配置 ~/.ssh/config）
Host 有线10_3_5090_ysj
  HostName 10.6.36.210
  User ysj
  RemoteForward 9999 127.0.0.1:7897
作用：将远程服务器的 9999 端口流量，通过 SSH 隧道转发到本地的 7897 端口。

3. VSCode 设置（settings.json）
在vscode的ctrl+shift+p选择：“首选项 打开远程配置(JSON)
[图片]
填写下面的
{
    "http.proxy": "http://127.0.0.1:9999",
    "http.noProxy": ["localhost", "127.0.0.1", "::1"],
    "http.proxySupport": "on"
}
作用：仅 VSCode 远程服务端自身使用该代理，用于扩展安装、更新等。

4. 服务器端 Shell 配置（~/.bashrc）
# 代理切换函数：输入 proxy 开启，再输入 proxy 关闭
proxy() {
    if [ -n "$http_proxy" ]; then
        unset http_proxy https_proxy no_proxy
        echo "❌ 代理已关闭"
    else
        export http_proxy="http://127.0.0.1:9999"
        export https_proxy="http://127.0.0.1:9999"
        export no_proxy="localhost,127.0.0.1,::1"
        echo "✅ 代理已开启：http/https -> 127.0.0.1:9999"
    fi
}


作用：
- proxy 函数：手动在终端中执行，开启/关闭当前 shell 的代理
- 方便快捷：后面每次使用，在终端输入proxy一次是打开，第二次是关闭。

使用方法
1. 本地启动代理软件（确保 7897 端口处于监听状态）。
2. 通过 VSCode 或终端连接远程服务器（SSH 隧道自动建立）。
3. VSCode 下载扩展：自动生效，无需额外操作。
4. 终端命令使用代理：
proxy                    # 开启代理
curl -I https://www.google.com
pip install some-package
proxy                    # 再次执行即关闭代理
  
注意事项
- 隧道必须存活：一旦 SSH 断开，9999 端口立即失效，所有依赖该端口的请求都会失败。
- 仅 HTTP 代理：隧道转发的是 HTTP 代理协议，如果本地 7897 是 SOCKS5，需转换或调整。
- 代理作用域：proxy 函数只影响当前终端，新开终端需重新执行。
- no_proxy 设置：避免本地地址走代理，防止 SSH 本身被误代理。
  
优化建议（可选）
为避免隧道未建立时误开代理，可将 proxy 函数增加端口检测：
proxy() {
    if [ -n "$http_proxy" ]; then
        unset http_proxy https_proxy no_proxy
        echo "❌ 代理已关闭"
    else
        if ss -tln | grep -q '127.0.0.1:9999'; then
            export http_proxy="http://127.0.0.1:9999"
            export https_proxy="http://127.0.0.1:9999"
            export no_proxy="localhost,127.0.0.1,::1"
            echo "✅ 代理已开启"
        else
            echo "⚠️  隧道未建立（127.0.0.1:9999 未监听），请先连接 SSH。"
            return 1
        fi
    fi
}

总结
通过 SSH 远程转发 + VSCode 设置 + Shell 函数，实现了远程服务器的按需上网。配置简单、安全，无需修改服务器全局网络，断开即失效，灵活可控。

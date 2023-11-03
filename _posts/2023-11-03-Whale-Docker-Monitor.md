---
title: CTFd 题目及平台日志监控
description: 摆脱cli，拥抱gui！
categories:
- CTFd
tags:
- CTFd-Whale
- Docker
date: 2023-11-03
photos: /ctfd/2023/11/03/Whale-Docker-Monitor/face.png
---
首先就是后台对应api的实现，直接用Whale插件处理好的dockerclient方便写代码，新增后端api如下
```python
{% raw %}
@page_blueprint.route("/admin/getLog")
@admins_only
def admin_get_log():
    id = request.args.get("id")
    tail = request.args.get("tail", 1000)
    docker_client = DockerUtils.client
    if id:
        try:
            container = docker_client.containers.get(id)
            logs_text = container.logs(stdout=True, stderr=True, stream=False, tail=tail)
            return {
                'success': True,
                'message': logs_text.decode('utf-8')
            }, 200
        except e:
            return {
                'success': False,
                'message': '日志获取失败：<br>' + str(e.__cause__)
            }, 200
    return {
        'success': False,
        'message': '日志获取失败'
    }, 200

@page_blueprint.route("/admin/docker")
@admins_only
def admin_list_docker():
    containers = DockerUtils.client.containers.list(all=True)
    return render_template("whale_docker.html",
                           plugin_name=plugin_name,
                           containers=containers)

@page_blueprint.route("/admin/viewLog")
@admins_only
def admin_view_log():
    return render_template("whale_log.html",
                           plugin_name=plugin_name)
{% endraw %}
```
然后就是前端实现上
```jinja2
{% raw %}
{% extends "whale_base.html" %}

{% block menu %}
    <li class="nav-item">
        <a class="nav-link" href="/plugins/ctfd-whale/admin/settings">🔗{{"Settings" if en else "设置"}}</a>
    </li>
    <li class="nav-item">
        <a class="nav-link" href="/plugins/ctfd-whale/admin/containers">🔗{{"Instances" if en else "实例"}}</a>
    </li>
    <li class="nav-item">
        <a class="nav-link" href="/plugins/ctfd-whale/admin/upload">🔗{{"Upload" if en else "上传"}}</a>
    </li>
    <li class="nav-item">
        <a class="nav-link active" href="#">{{"Containers" if en else "容器"}}</a>
    </li>
{% endblock %}

{% block panel %}
    <svg hidden>
        <symbol id="copy" viewBox="0 0 24 24">
            <path d="M16 1H4c-1.1 0-2 .9-2 2v14h2V3h12V1zm3 4H8c-1.1 0-2 .9-2 2v14c0 1.1.9 2 2 2h11c1.1 0 2-.9 2-2V7c0-1.1-.9-2-2-2zm0 16H8V7h11v14z"></path>
        </symbol>
    </svg>
    <div class="row">
        <div class="col-md-12">
            <table class="table table-striped border">
                <thead>
                <tr>
                    <th class="sort-col text-center"><b>ID</b></td>
                    <th class="text-center"><b>{{"Image Name" if en else "镜像名称"}}</b></td>
                    <th class="sort-col text-center"><b>{{"Container Name" if en else "容器名称"}}</b></td>
                    <th class="sort-col text-center"><b>{{"Container Satus" if en else "运行状态"}}</b></td>
                    <th class="text-center"><b>{{"Container Logs" if en else "容器日志"}}</b></td>
                </tr>
                </thead>
                <tbody>
                {% for container in containers %}
                    <tr>
                        <td class="text-center">
                            {{ container.id  | truncate(12)}}
                            <svg class="click-copy" data-copy="{{ container.id }}"
                                 height="24" width="24" style="cursor: pointer;">
                                <use xlink:href="#copy" />
                            </svg>
                        </td>
                        <td class="text-center">
                            {{ container.image.tags[0]  }}
                            <svg class="click-copy" data-copy="{{ container.image.tags[0] }}"
                                 height="24" width="24" style="cursor: pointer;">
                                <use xlink:href="#copy" />
                            </svg>
                        </td>
                        <td class="text-center">
                            {{ container.name }}
                        </td>
                        <td class="text-center">
                            {{ container.status }}&nbsp;
                        </td>
                        <td class="text-center">
                            <a class="view-log" container-id="{{ container.id }}" data-toggle="tooltip" data-placement="top"
                               title="{{'Check Logs' if en else '查看日志'}}">
                                <i class="fas fa-file"></i>
                            </a>
                        </td>
                    </tr>
                {% endfor %}
                </tbody>
            </table>
        </div>
    </div>

{% endblock %}

{% block scripts %}
    <script defer src="{{ url_for('plugins.ctfd-whale.assets', path='docker.js') }}"></script>
{% endblock %}
{% endraw %}
```
docker.js
```javascript
{% raw %}
const $ = CTFd.lib.$;

function language(en,zh) {
    const cookies = document.cookie.split('; ');
    for (const cookie of cookies) {
        const [cookieName, cookieValue] = cookie.split('=');
        if (cookieName === "Scr1wCTFdLanguage") {
            return (decodeURIComponent(cookieValue) === "en" ? en : zh);
        }
    }
    return zh;
}

function htmlentities(str) {
    return String(str).replace(/&/g, '&amp;').replace(/</g, '&lt;').replace(/>/g, '&gt;').replace(/"/g, '&quot;');
}

function copyToClipboard(event, str) {
    // Select element
    const el = document.createElement('textarea');
    el.value = str;
    el.setAttribute('readonly', '');
    el.style.position = 'absolute';
    el.style.left = '-9999px';
    document.body.appendChild(el);
    el.select();
    document.execCommand('copy');
    document.body.removeChild(el);

    $(event.target).tooltip({
        title: language("Copied!","复制完成!"),
        trigger: "manual"
    });
    $(event.target).tooltip("show");

    setTimeout(function () {
        $(event.target).tooltip("hide");
    }, 1500);
}

$(".click-copy").click(function (e) {
    copyToClipboard(e, $(this).data("copy"));
})

$(".view-log").click(function (e) {
    e.preventDefault();
    let container_id = $(this).attr("container-id");
    window.open("/plugins/ctfd-whale/admin/viewLog?id="+container_id,"_blank")
});
{% endraw %}
```
然后就是查看日志的页面，为了看起来好看用xterm.js渲染日志，并且允许关闭自动刷新和设置日志显示行数
```jinja2
{% raw %}
{% extends "admin/base.html" %}

{% if request.cookies.get('Scr1wCTFdLanguage') == 'en' %}
    {% set en = true %}
{% else %}
    {% set en = false %}
{% endif %}

{% block content %}
    <link rel="stylesheet" href="{{ url_for('plugins.ctfd-whale.assets', path='log.css') }}">
    <link rel="stylesheet" href="{{ url_for('plugins.ctfd-whale.assets', path='xterm.css') }}">
    <div class="container mt-4">
        <h1>容器日志</h1>
        <div class="row mb-2">
            <div class="col-md-6">
                <label for="logLines">日志行数:</label>
                <input type="number" id="logLines" class="form-control" value="500">
            </div>
            <div class="col-md-6">
                <label for="switch">自动刷新:</label>
                <br>
                <div id="switch" class="switch">
                    <input type="checkbox" id="autoRefresh" checked>
                    <span class="slider round"></span>
                </div>
            </div>
        </div>
        <div id="terminal"></div>
    </div>
{% endblock %}
{% block scripts %}
    <script defer src="{{ url_for('plugins.ctfd-whale.assets', path='xterm.js') }}"></script>
    <script defer src="{{ url_for('plugins.ctfd-whale.assets', path='xterm-addon-fit.js') }}"></script>
    <script defer src="{{ url_for('plugins.ctfd-whale.assets', path='bundle.js') }}"></script>
    <script>
        // bundle的源码
        // import {Terminal} from 'xterm'
        // import {FitAddon} from 'xterm-addon-fit';
        //
        // window.addEventListener('DOMContentLoaded', (event) => {
        //     function getCookieForLanguage(name) {
        //         const cookies = document.cookie.split('; ');
        //         for (const cookie of cookies) {
        //             const [cookieName, cookieValue] = cookie.split('=');
        //             if (cookieName === name) {
        //                 return decodeURIComponent(cookieValue);
        //             }
        //         }
        //         return null;
        //     }
        //     const terminal = new Terminal();
        //     const fitAddon = new FitAddon();
        //     terminal.loadAddon(fitAddon);
        //     terminal.open(document.getElementById('terminal'));
        //     fitAddon.fit();
        //
        //     // 获取当前页面地址中的id参数
        //     const urlParams = new URLSearchParams(window.location.search);
        //     const id = urlParams.get('id');
        //
        //     const logContent = document.getElementById('logContent');
        //     const logLines = document.getElementById('logLines');
        //     const autoRefresh = document.getElementById('autoRefresh');
        //
        //     // 获取日志内容的函数
        //     function getLog() {
        //         const tail = logLines.value;
        //         const url = `/plugins/ctfd-whale/admin/getLog?id=${id}&tail=${tail}`;
        //
        //         window.CTFd.fetch(url, {
        //             method: 'GET',
        //             credentials: 'same-origin',
        //             headers: {
        //                 'Accept': 'application/json',
        //             }
        //         }).then(function (response) {
        //             if (response.status === 429) {
        //                 // User was ratelimited but process response
        //                 return response.json();
        //             }
        //             if (response.status === 403) {
        //                 // User is not logged in or CTF is paused.
        //                 return response.json();
        //             }
        //             return response.json();
        //         }).then(function (response) {
        //             if (response.success) {
        //                 terminal.clear();
        //                 const messages = response.message.split('\n');
        //                 for (const message of messages) {
        //                     terminal.writeln(message);
        //                 }
        //             } else {
        //                 var e = new Object;
        //                 e.title = (getCookieForLanguage("Scr1wCTFdLanguage") === "en" ? "Log Auto Refresh Fail!" : "日志自动刷新失败！");
        //                 e.body = response.message;
        //                 CTFd.ui.ezq.ezToast(e)
        //             }
        //         });
        //     }
        //
        //     // 自动刷新日志内容
        //     let refreshInterval;
        //
        //     function startAutoRefresh() {
        //         refreshInterval = setInterval(getLog, 10000); // 10 seconds
        //     }
        //
        //     function stopAutoRefresh() {
        //         clearInterval(refreshInterval);
        //     }
        //
        //     // 切换自动刷新状态
        //     autoRefresh.addEventListener('change', () => {
        //         if (autoRefresh.checked) {
        //             startAutoRefresh();
        //         } else {
        //             stopAutoRefresh();
        //         }
        //     });
        //     getLog()
        //     startAutoRefresh()
        // })
    </script>
{% endblock %}
{% endraw %}
```
bundle的源码就是上面注释里的内容，引入xterm和xterm-fit-addon使用webpack打包。

效果如下：
![](docker-1.png)
![](docker-2.png)
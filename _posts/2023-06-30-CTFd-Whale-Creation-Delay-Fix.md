---
title: CTFd-Whale 镜像创建延迟修复
description: CTFd-Whale是CTFd的一个插件，用于部署容器实例作为靶场。在容器提示创建之后，会有很长一段时间（30秒-1分钟，因服务器性能而异），给予的题目链接无法访问。
categories:
- CTFd
tags:
- CTFd
- CTFd-Whale
- Docker
date: 2023-06-30
photos: /ctfd/2023/06/30/CTFd-Whale-Creation-Delay-Fix/image-20230630165128707.png
---
实际上，由于他创建的服务需要自行再去创建task，而task有一个prepare的阶段，这阶段就是这段等待时间。如果我们想要前端进行等待，需要修改两个地方：

```python
service = client.services.create(
    image=container.challenge.docker_image,
    name=f'{container.user_id}-{container.uuid}',
    env={'FLAG': container.flag}, dns_config=docker.types.DNSConfig(nameservers=dns),
    networks=[get_config("whale:docker_auto_connect_network", "ctfd_frp_containers")],
    resources=docker.types.Resources(
        mem_limit=DockerUtils.convert_readable_text(
            container.challenge.memory_limit),
        cpu_limit=int(container.challenge.cpu_limit * 1e9)
    ),
    labels={
        'whale_id': f'{container.user_id}-{container.uuid}'
    },  # for container deletion
    constraints=['node.labels.name==' + node],
    endpoint_spec=docker.types.EndpointSpec(mode='dnsrr', ports={})
)
```

在这之后，我们不让函数自己返回，而是加上一段判断：

```python
count = 0
while True:
    count += 1
    try:
        tasks = service.tasks()
        for task in tasks:
            current_state = task['Status']['State']
            if 'running'.lower() in current_state.lower():
                print("容器启动成功！")
                return
    except:
        pass

    if count > 960:
        service.remove()
        raise Exception("容器创建超时")
    time.sleep(0.5)  # 等待0.5秒后重新检查
```

通过获取task里的对象的state，变成running之后再返回，进行注册路由等操作，这样前端就会一直等待

![image-20230630165128707](image-20230630165128707.png)

但是有一个问题，如果用户等不及了，在此时刷新页面，那么就会出现一个问题，页面显示了domain:0的地址。

那么就在前端修改一下loadinfo函数。

![image-20230630165314205](image-20230630165314205.png)

```javascript
function loadInfo() {
    var challenge_id = $('#challenge-id').val();
    var url = "/api/v1/plugins/ctfd-whale/container?challenge_id=" + challenge_id;

    var params = {};

    CTFd.fetch(url, {
        method: 'GET',
        credentials: 'same-origin',
        headers: {
            'Accept': 'application/json',
            'Content-Type': 'application/json'
        }
    }).then(function (response) {
        if (response.status === 429) {
            // User was ratelimited but process response
            return response.json();
        }
        if (response.status === 403) {
            // User is not logged in or CTF is paused.
            return response.json();
        }
        return response.json();
    }).then(function (response) {
        if (window.t !== undefined) {
            clearInterval(window.t);
            window.t = undefined;
        }
        if (response.success) response = response.data;
        else CTFd.ui.ezq.ezAlert({
            title: "Fail",
            body: response.message,
            button: "OK"
        });
        if (response.remaining_time === undefined) {
            $('#whale-panel').html('<div class="card" style="width: 100%;">' +
                '<div class="card-body">' +
                '<h5 class="card-title">实例信息</h5>' +
                '<button type="button" class="btn btn-primary card-link" id="whale-button-boot" ' +
                '        onclick="CTFd._internal.challenge.boot()">' +
                '启动题目实例' +
                '</button>' +
                '</div>' +
                '</div>');
        } else {
            if (response.user_access.includes(":0"))
            {
                $('#whale-panel').html('<div class="card" style="width: 100%;">' +
                    '<div class="card-body">' +
                    '<h5 class="card-title">实例信息</h5>' +
                    '<button type="button" class="btn btn-primary card-link" id="whale-button-boot">' +
                    '正在启动容器，请等待。。。' +
                    '</button>' +
                    '</div>' +
                    '</div>');
                $('#whale-button-boot')[0].disabled = true;
                setTimeout(loadInfo,2000)
            }
            else
            {
                $('#whale-panel').html(
                    `<div class="card" style="width: 100%;">
                    <div class="card-body">
                        <h5 class="card-title">实例信息</h5>
                        <h6 class="card-subtitle mb-2 text-muted" id="whale-challenge-count-down">
                            剩余时间: ${response.remaining_time}秒
                        </h6>
                        <h6 class="card-subtitle mb-2 text-muted">
                            局域网域: ${response.lan_domain}
                        </h6>
                        <a id="user-access" class="card-text" target="_blank" href=""></a><br/><br/>
                        <button type="button" class="btn btn-danger card-link" id="whale-button-destroy"
                                onclick="CTFd._internal.challenge.destroy()">
                            关闭实例容器
                        </button>
                        <button type="button" class="btn btn-success card-link" id="whale-button-renew"
                                onclick="CTFd._internal.challenge.renew()">
                            延期实例容器
                        </button>
                    </div>
                </div>`
                );
                $('#user-access').html(response.user_access);
                function showAuto() {
                    const c = $('#whale-challenge-count-down')[0];
                    if (c === undefined) return;
                    const origin = c.innerHTML;
                    const second = parseInt(origin.split(": ")[1].split('s')[0]) - 1;
                    c.innerHTML = '剩余时间: ' + second + '秒';
                    if (second < 0) {
                        loadInfo();
                    }
                }
                window.t = setInterval(showAuto, 1000);

                var port = response.user_access.split(':');
                var host = document.location.host.split(':')[0];
                document.getElementById("user-access").href="http://"+host+":"+port[1];
            }
        }
    });
};
```

这样前端该等待的时候就会一直等待啦。（最后三行是为了实现链接可以点击，可加可不加）

PS：我们Scr1w战队二次开发的CTFd整合版地址：https://github.com/dlut-sss/CTFD-Public

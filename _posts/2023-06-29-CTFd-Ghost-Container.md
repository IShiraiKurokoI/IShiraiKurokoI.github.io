---
title: CTFd异常关闭时Whale插件的幽灵容器问题处理
description: CTFd-Whale是CTFd的一个插件，用于部署容器实例作为靶场，然而多多少少有点问题
categories:
- CTFd
tags:
- CTFd
- CTFd-Whale
- Docker
date: 2023-06-29
---
CTFd-Whale通过数据库来记录存在哪些容器和服务，以便进行开启和关闭，但如果承载CTFd的服务器或者平台出现故障异常重启，或平台异常关闭，有可能导致记录丢失，容器和服务变身无主容器。
有人可能就想问了，我直接 docker ps 然后 docker stop，之后docker rm不就移除了吗？
错误的捏，我们通过研究Whale的代码可以发现，他是通过以下代码来创建容器的。

```python
client.services.create(
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

也就是说，他创建了服务，然后服务再创建了容器，如果不消灭服务，那么就算你重启服务器，这个默认模式为replicated的服务也会不断给你创建容器，让你永远无法达到杀死容器的目的。

解决办法就是移除服务，举例如下

![幽灵容器处理办法](幽灵容器处理办法.png)

找到他创建的服务，直接rm掉服务，就可以解决问题了。

妈妈再也不用担心产生无主容器啦。

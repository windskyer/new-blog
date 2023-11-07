---
title: gpu压力测试
categories: Kubernetes
sage: true
date: 2022-01-14 11:43:48
tags: gpu
---

## gpu压力测试
### 执行gpu脚本
```shell
# 进入要压力测试gpu 容器
kubect -n aijob-system exec -it dddd-0 -- sh
# 编写测试脚本
cat << EOF > demo.py
import os
import numpy as np
import torch
from torchvision.models import resnet18
import time

if __name__ == '__main__':
    model = resnet18(pretrained=False)
    device = torch.device('cuda')
    model.eval()
    model.to(device)
    dump_input = torch.ones(1,3,224,224).to(device)

    # Warn-up
    for _ in range(500000):
        start = time.time()
        outputs = model(dump_input)
        torch.cuda.synchronize()
        end = time.time()
        print('Time:{}ms'.format((end-start)*1000))

    with torch.autograd.profiler.profile(enabled=True, use_cuda=True, record_shapes=False, profile_memory=False) as prof:
        outputs = model(dump_input)
    print(prof.table())
EOF
# 执行python 文件
python demo.py
```

### 查看gpu的使用率
```shell
# grah 地址查看
http://172.18.1.38:30901/graph?g0.expr=DCGM_FI_DEV_GPU_UTIL%7Bnamespace%3D%22aijob-system%22%2Cpod!%3D%22%22%7D&g0.tab=1&g0.stacked=0&g0.show_exemplars=0&g0.range_input=1h

# 监控指标
DCGM_FI_DEV_GPU_UTIL{namespace="aijob-system",pod!=""}
```

### 查看pod 的hpa
```shell
kubectl -n aijob-system get horizontalpodautoscaler.autoscaling  dddd(任务名称)
输出如下：
NAME                                          REFERENCE       TARGETS        MINPODS   MAXPODS   REPLICAS   AGE
horizontalpodautoscaler.autoscaling/dddd      AIJob/dddd      0/80           2         3         2          44s
# TARGETS gpu使用率阀值
# MINPODS 最小pod数量
# MAXPODS 最大pod数量
```

### 查看pod 数量是否正确
```shell
kubectl -n aijob-system get pod
```

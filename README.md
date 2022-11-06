# 测试使用 [ktunnel](https://github.com/omrikiei/ktunnel)

**ktunnel: A CLI tool that establishes a reverse tunnel between a kubernetes cluster and your local machine**

## 使用 k3d 启动一个 kubernetes 集群用于测试
1. 初始化容器配置
    `k3d config init -o kubernetes/k3d-config-5.yaml`
2. 修改配置 本次测试仅测试 `ktunnel` 配置仅修改`name` `agents:1` 其他不需要修改
3. 启动集群 
    `k3d cluster create -c kubernetes/k3d-config-5.yaml`
4. 本地配置修改
    - `k3d kubeconfig get k5 > kubernetes/k3d-cluster-5.yaml`
    - `cp kubernetes/k3d-cluster-5.yaml ~/.kube/conf/`
    - 刷新配置
    - `kubectx`
    - 测试访问
        ```sh
        % k get all        
        NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
        service/kubernetes   ClusterIP   10.43.0.1    <none>        443/TCP   3h10m
        ```

## 下载 ktunnel 并安装
https://github.com/omrikiei/ktunnel/releases

## 测试
1. 使用docker启动本地nginx
    - 
    ```sh
    docker run -d --name mynginx3 -p 8080:80 nginx
    ```
    - 访问 localhost:8080 nginx启动成功

2. 使用 `ktunnel expose` expose 本地的 8080 端口到 kubernetes
    ```% ktunnel expose mynginx 8080:8080
    INFO[0000] Exposed service's cluster ip is: 10.43.109.162 
    .INFO[0000] waiting for deployment to be ready           
    ........................
    INFO[0007] port forwarding to https://0.0.0.0:38241/api/v1/namespaces/default/pods/mynginx-fc8777cf4-55zq4/portforward 
    INFO[0007] Waiting for port forward to finish           
    INFO[0007] Forwarding from 127.0.0.1:28688 -> 28688
    Forwarding from [::1]:28688 -> 28688 
    INFO[2022-11-06 20:36:54.503] starting tcp tunnel from source 8080 to target 8080
    ```

3. 查看当前集群资源
    ```sh
    % k get all        
    NAME                          READY   STATUS    RESTARTS   AGE
    pod/mynginx-fc8777cf4-55zq4   1/1     Running   0          76s

    NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
    service/kubernetes   ClusterIP   10.43.0.1       <none>        443/TCP    3h13m
    service/mynginx      ClusterIP   10.43.109.162   <none>        8080/TCP   76s

    NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
    deployment.apps/mynginx   1/1     1            1           76s

    NAME                                DESIRED   CURRENT   READY   AGE
    replicaset.apps/mynginx-fc8777cf4   1         1         1       76s
    ```
    ```sh
    % k describe pod/mynginx-fc8777cf4-55zq4                       
    Name:             mynginx-fc8777cf4-55zq4
    Namespace:        default
    Priority:         0
    Service Account:  default
    Node:             k3d-k5-agent-0/172.24.0.3
    Start Time:       Sun, 06 Nov 2022 20:36:47 +0800
    Labels:           app.kubernetes.io/instance=mynginx
                    app.kubernetes.io/name=mynginx
                    pod-template-hash=fc8777cf4
    Annotations:      <none>
    Status:           Running
    IP:               10.42.1.6
    IPs:
    IP:           10.42.1.6
    Controlled By:  ReplicaSet/mynginx-fc8777cf4
    Containers:
    ktunnel:
        Container ID:  containerd://d1cc69b063a8cbf0e39dde9867928551f7c3f79e5723edf826afec26a62294e2
        Image:         quay.io/omrikiei/ktunnel:v1.4.8
        Image ID:      quay.io/omrikiei/ktunnel@sha256:75868dc1db4183df83e849b0aea47e82d2cb8ce5baad72b5c1202a7714060927
        Port:          8080/TCP
        Host Port:     0/TCP
        Command:
        /ktunnel/ktunnel
        Args:
        server
        -p
        28688
        State:          Running
        Started:      Sun, 06 Nov 2022 20:36:53 +0800
        Ready:          True
        Restart Count:  0
        Limits:
        cpu:     1
        memory:  1e9
        Requests:
        cpu:        500e-3
        memory:     100e6
        Environment:  <none>
        Mounts:
        /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-t8bqz (ro)
    Conditions:
    Type              Status
    Initialized       True 
    Ready             True 
    ContainersReady   True 
    PodScheduled      True 
    Volumes:
    kube-api-access-t8bqz:
        Type:                    Projected (a volume that contains injected data from multiple sources)
        TokenExpirationSeconds:  3607
        ConfigMapName:           kube-root-ca.crt
        ConfigMapOptional:       <nil>
        DownwardAPI:             true
    QoS Class:                   Burstable
    Node-Selectors:              <none>
    Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                                node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
    Events:
    Type    Reason     Age    From               Message
    ----    ------     ----   ----               -------
    Normal  Scheduled  2m31s  default-scheduler  Successfully assigned default/mynginx-fc8777cf4-55zq4 to k3d-k5-agent-0
    Normal  Pulling    2m31s  kubelet            Pulling image "quay.io/omrikiei/ktunnel:v1.4.8"
    Normal  Pulled     2m25s  kubelet            Successfully pulled image "quay.io/omrikiei/ktunnel:v1.4.8" in 5.768035606s
    Normal  Created    2m25s  kubelet            Created container ktunnel
    Normal  Started    2m25s  kubelet            Started container ktunnel
    ```

4. 使用 `curl` 在集群上测试联通性
    -
    ```sh
    % k run -it --rm --restart=Never --image alpine/curl -- curl mynginx:8080
    <!DOCTYPE html>
    <html>
    <head>
    <title>Welcome to nginx!</title>
    <style>
    html { color-scheme: light dark; }
    body { width: 35em; margin: 0 auto;
    font-family: Tahoma, Verdana, Arial, sans-serif; }
    </style>
    </head>
    <body>
    <h1>Welcome to nginx!</h1>
    <p>If you see this page, the nginx web server is successfully installed and
    working. Further configuration is required.</p>

    <p>For online documentation and support please refer to
    <a href="http://nginx.org/">nginx.org</a>.<br/>
    Commercial support is available at
    <a href="http://nginx.com/">nginx.com</a>.</p>

    <p><em>Thank you for using nginx.</em></p>
    </body>
    </html>
    pod "curl" deleted
    ```

    - ktunnel 的 terminal 打印出日志:
    ```sh
    INFO[2022-11-06 20:42:10.265] new connection                                port=8080 session=844663df-3199-466c-9cba-52e0a91857ed
    INFO[2022-11-06 20:42:10.266] closed session                                session=844663df-3199-466c-9cba-52e0a91857ed
    ```
## Prow 集群

主要需要完成的操作为：

- kind 集群创建
- prow 集群创建

### kind 集群创建

由于访问受限参考了《使用离线环境创建 K8S 集群》。

1. 拉取 kind release 时的默认镜像 `sudo docker pull kindest/node:v1.19.1@sha256:98cf5288864662e37115e362b23e4369c8c4a408f99cbc06e58ac30ddc721600`
2. 使用此镜像创建集群 `sudo kind create cluster --image kindest/node:v1.19.1@sha256:98cf5288864662e37115e362b23e4369c8c4a408f99cbc06e58ac30ddc721600 --name=prow-dev`（可以通过 `--name` 选项指定集群名，否则默认为 kind）
3. 检查：

   ```shell
   $ sudo kind get clusters
   prow-dev
   $ sudo kind get nodes --name=prow-dev
   prow-dev-control-plane
   $ sudo kubectl cluster-info --context kind-prow-dev
   Kubernetes control plane is running at https://127.0.0.1:41709
   KubeDNS is running at https://127.0.0.1:41709/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
   $ sudo kubectl get po -A
   NAMESPACE            NAME                                             READY   STATUS    RESTARTS   AGE
   kube-system          coredns-f9fd979d6-gprs2                          1/1     Running   0          7m41s
   kube-system          coredns-f9fd979d6-npp6s                          1/1     Running   0          7m41s
   kube-system          etcd-prow-dev-control-plane                      1/1     Running   0          7m46s
   kube-system          kindnet-h69cn                                    1/1     Running   0          7m41s
   kube-system          kube-apiserver-prow-dev-control-plane            1/1     Running   0          7m45s
   kube-system          kube-controller-manager-prow-dev-control-plane   1/1     Running   0          7m46s
   kube-system          kube-proxy-24twk                                 1/1     Running   0          7m41s
   kube-system          kube-scheduler-prow-dev-control-plane            1/1     Running   0          7m46s
   local-path-storage   local-path-provisioner-78776bfc44-2hgnx          1/1     Running   0          7m41s
   ```
   
请忽略掉访问 `https://127.0.0.1:41709` 时提示的 `"forbidden: User \"system:anonymous\" cannot get path \"/\""` 等问题，这不是目前关注的要点，当然也可以通过配置证书或关闭 HTTPS 来解决。
   
### Prow 集群创建

这里先使用一套简版的基础配置进行 prow 集群的创建和验证。配置文件参见 [config/sample/stater.yaml](./config/sample/stater.yaml) 。

#### GitHub 相关 Token 的创建

1. 新建机器人帐号/使用已有帐号，这里选择新建 psiace-robot 。
2. 将机器人帐号添加到测试仓库合作者中，以授予其访问权限。
2. 为机器人帐号生成 github token ：settings -> developer settings -> personal access tokens -> repo:status & public_repo 并保存 `echo "<your-token-here>" > oauth-token`
4. 生成一个用于 Github Webhook 认证的 hmac 令牌 `openssl rand -hex 20 > hmac-token`

#### 启动 Prow 集群

这里没有采用明文放入 token 的做法，而是使用 kubectl 创建对应的 secret 。

1. `sudo kubectl create secret generic oauth-token --from-file=oauth=./oauth-token`
2. `sudo kubectl create secret generic hmac-token --from-file=hmac=./hmac-token`
5. `sudo kubectl apply -f ./config/sample/stater.yaml` （已经使用 zhangsean/k8s-prow- 替换依赖的 gcr.io/k8s-prow/ 获得一定加速）

**注意** apply 时创建容器仍然可能很慢，所以为了节约生命，这里提供一个快速的办法：

1. 本地 `docker pull <images>` ，这里的 images 即 starter.yaml 中指定的若干 image ，之后按下述方法处理：
2. `sudo docker images` 列出所有已经 pull 到本地镜像
3. `sudo kind load docker-image <every-image-must-be-used> --name prow-dev`
4. 如果你使用的不是本配置，可以考虑将需要 apply 的 yaml 文件中的所有镜像加载策略为 Always 的一行删除

查看当前容器状态：

```
$ sudo kubectl get po -A
NAMESPACE            NAME                                             READY   STATUS      RESTARTS   AGE
default              deck-855b8f4bc5-pnwpx                            1/1     Running     0          71s
default              deck-855b8f4bc5-wsc77                            1/1     Running     0          71s
default              hook-588b5789f7-24fsw                            1/1     Running     0          70s
default              hook-588b5789f7-wkj2q                            1/1     Running     0          70s
default              horologium-587bb9548d-kszbh                      1/1     Running     0          70s
default              plank-54bf98c7cd-bmhnn                           1/1     Running     0          71s
default              sinker-7c88c4f846-p64rc                          1/1     Running     0          70s
default              statusreconciler-84d7c86d7c-d9tnj                1/1     Running     0          70s
default              tide-84486c87c-g5l78                             1/1     Running     0          70s
kube-system          coredns-f9fd979d6-gprs2                          1/1     Running     0          5h29m
kube-system          coredns-f9fd979d6-npp6s                          1/1     Running     0          5h29m
kube-system          etcd-prow-dev-control-plane                      1/1     Running     0          5h29m
kube-system          kindnet-h69cn                                    1/1     Running     0          5h29m
kube-system          kube-apiserver-prow-dev-control-plane            1/1     Running     0          5h29m
kube-system          kube-controller-manager-prow-dev-control-plane   1/1     Running     0          5h29m
kube-system          kube-proxy-24twk                                 1/1     Running     0          5h29m
kube-system          kube-scheduler-prow-dev-control-plane            1/1     Running     0          5h29m
local-path-storage   local-path-provisioner-78776bfc44-2hgnx          1/1     Running     0          5h29m
test-pods            671179d4-5e1a-11eb-bd89-1685c8e9b23b             0/1     Completed   0          39s
```

#### 访问 Prow

访问时的网址应该类似这样：

```
deck: http://prow.example.com/
hook: http://prow.example.com/hook/
```

由于这次是本地部署，没有域名，所以是通过 `127.0.0.1` 访问。

1. 查看 deck 和 hook 服务的 NodePort 端口
   ```
   $ sudo kubectl get svc
   NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
   deck         NodePort    10.103.186.73    <none>        80:31112/TCP     3m26s
   hook         NodePort    10.107.106.113   <none>        8888:30699/TCP   3m26s
   kubernetes   ClusterIP   10.96.0.1        <none>        443/TCP          5h32m
   tide         NodePort    10.99.240.243    <none>        80:32236/TCP     3m26s
   ```
2. 查看 Ingress
   ```
   $ sudo kubectl get ing
   NAME   CLASS    HOSTS   ADDRESS   PORTS   AGE
   ing    <none>   *                 80      3m2s
   $ sudo kubectl describe ingress ing
   Name:             ing
   Namespace:        default
   Address:
   Default backend:  deck:80 (10.244.0.33:8080,10.244.0.35:8080)
   Rules:
       Host        Path  Backends
       ----        ----  --------
       *
                   /       deck:80 (10.244.0.33:8080,10.244.0.35:8080)
                   /hook   hook:8888 (10.244.0.39:8888,10.244.0.40:8888)
   Annotations:  <none>
   Events:       <none>
   ```
   **注意** 这里 deck 对应的 Path 已经是 `/` 了，所以不需要修改，如果为 `/*`，则参考 [Prow 快速入门向导](https://www.servicemesher.com/blog/prow-quick-start-guide/)，
   使用 `sudo kubectl get ingress ing -o yaml | sed 's|path: /\*|path: /|g' | kubectl apply -f -` 进行适配。
3. 由于使用 kind 启动集群，这里需要将端口映射到本地才能进行访问
   ```
   $ sudo docker run -itd --name prow-dev-deck -p 8080:80 --link prow-dev-control-plane --net kind zhangsean/nginx-port-proxy prow-dev-control-plane:31112
   $ sudo docker run -itd --name prow-dev-hook -p 8088:80 --link prow-dev-control-plane --net kind zhangsean/nginx-port-proxy prow-dev-control-plane:30699
   ```
   **注意** 这里 kind-control-plane:<port> 为 NodePort 端口，而 <port>:80 为要映射到的端口。
4. 访问 `http://127.0.0.1:8080/` 可以看到 Prow 界面，部署完成。

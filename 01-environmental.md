## 环境搭建

需要的前置环境如下：

- go
- docker
- k8s 组件
- kind

### Go

Go 语言环境安装只需要按官方文档即可，这里的步骤是我个人为了方便管理而使用的。

1. 下载 Go 语言的预编译二进制文件：`https://studygolang.com/dl/golang/go1.15.7.linux-amd64.tar.gz`
2. 解压 `tar -C $HOME/.go -xzf go1.15.7.linux-amd64.tar.gz` ，此操作将 go 相关文件解压到 `$HOME/.go` 路径下
3. 在 `$HOME/.go` 文件夹下新建 `env` 文件，内容如下：

   ```shell
   #!/bin/sh
   # affix colons on either side of $PATH to simplify matching
   case ":${PATH}:" in
        *:"$HOME/.go/bin":*)
            ;;
        *)
            # Prepending path in case a system-installed go needs to be overridden
            export PATH="$HOME/.go/bin:$PATH"
            export PATH="$(go env GOPATH)/bin:$PATH"
            ;;
   esac
   ```
4. 打开 `$HOME/.bashrc` 文件，写入 `source "$HOME/.go/env"`（请了解您计算机启动时 dotfile 的加载顺序，你可能会更倾向于写入 .profile 文件）
5. 验证，打开终端，输入 `go version`，输出为 `go version go1.15.7 linux/amd64`，安装成功。
6. (可选) goproxy 加速 `go env -w GO111MODULE=on && go env -w GOPROXY=https://goproxy.cn,direct`

### Docker

很高兴看到 docker 添加了实验性的 cgroup v2 支持，fedora 用户少了很多麻烦。 

1. 添加仓库 `sudo dnf config-manager --add-repo https://download.docker.com/linux/fedora/docker-ce.repo`
2. (可选) 配置从清华源下载 `sudo sed -i 's+download.docker.com+mirrors.tuna.tsinghua.edu.cn/docker-ce+' /etc/yum.repos.d/docker-ce.repo`
3. 安装社区版 `sudo dnf install docker-ce`
4. 启动 docker `systemctl enable --now docker`
5. 查看 docker 状态，`systemctl status docker` 看到 Active: active (running) 则说明启动成功
6. (可选) 配置加速服务
    ```shell
    $ sudo tee /etc/docker/daemon.json <<-'EOF'
    {
      "registry-mirrors": [
          "https://ksrmtc13.mirror.aliyuncs.com",
          "https://docker.mirrors.ustc.edu.cn",
          "http://f1361db2.m.daocloud.io",
          "https://registry.docker-cn.com",
          "https://mirror.baidubce.com"]
    }
    EOF
    $ sudo systemctl daemon-reload
    $ sudo systemctl restart docker
    ```
7. 测试 docker 容器 
    - 使用 alpine 镜像测试 `sudo docker pull alpine`
    - 运行镜像 `sudo docker run -it --rm alpine /bin/sh`
    - 随便选几条命令测试 `/ # apk update`
      ```text
      fetch https://dl-cdn.alpinelinux.org/alpine/v3.13/main/x86_64/APKINDEX.tar.gz
      fetch https://dl-cdn.alpinelinux.org/alpine/v3.13/community/x86_64/APKINDEX.tar.gz
      v3.13.0-40-ga7b5f79547 [https://dl-cdn.alpinelinux.org/alpine/v3.13/main]
      v3.13.0-42-g90aa2099f0 [https://dl-cdn.alpinelinux.org/alpine/v3.13/community]
      OK: 13870 distinct packages available
      ```
    - 退出 `/ # exit`
    
### k8s 组件

1. 添加仓库（国内）

   ```
   $ sudo tee /etc/yum.repos.d/kubernetes.repo <<-'EOF'
   [kubernetes]
   name=Kubernetes
   baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
   enabled=1
   gpgcheck=1
   repo_gpgcheck=1
   gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
   EOF
   ```
2. 安装组件 `dnf install -y kubelet kubeadm kubectl`
3. 启动时需要使用 `systemctl enable kubelet && systemctl start kubelet`
4. 执行 3 之后，`sudo systemctl status kubectl` 显示 Unit kubectl.service could not be found. 不需要理会，因为你还没有集群。
     
### kind

根据官方文档，由于前面已经安装了 go 语言环境并设置 `GO111MODULE=on` 。只需要 `go get sigs.k8s.io/kind@v0.9.0` 即可。

**注意** 必须将 GOPATH 添加到 PATH 。否则会遇到 `kind: command not found` ，这一步已经在安装 GO 时候第三步中，通过 `export PATH="$(go env GOPATH)/bin:$PATH"` 解决。

由于并非全局安装，sudo 模式执行 kind 命令也会报 `command not found` ，可以通过：

- 执行命令时使用 `sudo $(go env GOPATH)/bin/kind` 替换 `sudo kind` ，或者：
- 为 sudo 设置别名 `alias sudo='sudo env PATH=$PATH'`（写入 `$HOME/.bashrc` ）


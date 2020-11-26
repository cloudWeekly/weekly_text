## Docker 镜像格式与本地存储规则

### 1. Open Container Initiative（OCI，开放容器标准）

一项新技术的推进必然少不了标准化的过程，标准化可以使技术的推进与开发能够基于通用标准之上，开发者可以分别聚焦于不同的细节，并通过标准化的接入方式来为新技术做出贡献，提升技术的成熟度与社区的活跃度，推动技术快速发展。

OCI，Open Container Initiative，开放容器标准就是在容器技术发展过程中出现的容器标准。OCI  在 Linux 基金会的支持下于成立，致力于围绕容器格式与运行时指定开放的行业标准。OCI 由 Docker 公司（下文中“Docker 公司”表示开发 Docker 容器引擎的 Docker 公司，独立出现的 “Docker” 则表示 Docker 容器引擎技术）、CoreOS 于 2015 年 6 月份启动。其提出了两个规范：

- **Image Format Specification：容器格式标准**。规定应该以何种格式存储、分发镜像。
- **Runtime Specification：容器运行时标准**。规定如何下载、解压缩、运行 [Filesystem Bundle](https://github.com/opencontainers/runtime-spec/blob/7c4c8f63a63693f75cfa0f3f397151fb8d9732ad/bundle.md)。

简单来说，OCI 规范了镜像如何存储与运行。任何符合 OCI 格式的镜像都可以在符合 OCI 标准的运行时之上工作。这里不得不提到 Docker 公司，Docker 公司将 Docker v2 镜像格式捐赠给 OCI 作为镜像规范的基础，同时 Docker 公司将 [libcontainer](https://github.com/docker-archive/libcontainer) 移动到 [runC](https://github.com/opencontainers/runc) 中贡献给了 OCI 作为容器运行时的参考实现。Docker v2 镜像格式和 runC 成为了 OCI 的指定参考和实现规范。

因此，我们后文中要探讨的 Docker 镜像格式都是以 OCI 镜像格式规范为基础的，**Docker 镜像是对 OCI 镜像格式的一种实现**。

> **容器运行时的演进流程**
>
> 容器技术出现于 Docker 容器引擎之前，LXC（LinuX Container）也是一种广义上的容器技术。但是受限于 LXC 的功能和难度，直到 Docker 容器引擎出现之前，容器技术一直不太火热。Docker 容器引擎提供了更高级的隔离性和易用性，所以说 Docker 诞生之后，容器技术才得到了真正的发展。
>
> Docker 公司与 2013 年发布了 Docker 项目，随后 Docker 越做越大，同时出于市场和社区的考虑，Docker 公司将 Docker 容器引擎逐渐拆解，分别拆解出了多个开源项目，包括：
>
> - [libcontainer](https://github.com/docker-archive/libcontainer)：Docker 容器运行时，后被移动到 runC 项目中，被贡献给 OCI。
> - [libnetwork](https://github.com/moby/libnetwork)：Docker 网络库。提出了 CNM 框架，同时提供 Docker 网络插件定义。*后来容器编排领域的事实标准 kubernetes 并[没有采用 libnetwork](https://kubernetes.io/zh/blog/2016/01/14/why-kubernetes-doesnt-use-libnetwork/) 而是使用了 CNI。*
> - [notary]()：[TUF](https://github.com/theupdateframework/specification) 框架的 golang 实现，负责容器安全。
> - [hyperkit](https://github.com/moby/hyperkit)：macOS 上运行的轻量级虚拟化工具包，基于 macOS 的 Hypervisor 框架，利用硬件虚拟化运行虚拟机。我们的 MacBook 能够运行 Docker 其实靠的就是一台轻量级 hyperkit 虚拟机。
>
> 后来，Docker 公司开源了工业级通用容器运行时 [containerd](https://github.com/containerd/containerd)，并将其贡献给了 CNCF。containerd 从更高的层面覆盖了容器运行时所需要做的一切，执行、分发、网络、存储、监控等，同时支持多种 OCI 容器运行时，其中就包括 runC。containerd 内部使用 containerd-shim 管理容器，每一个启动的容器都会对应一个 containerd-shim 进程，该进程的启动参数就包括 OCI 的 bundle 目录、OCI 容器运行时二进制程序。
>
> 所以一个 docker 容器引擎启动容器要经历以下步骤：
>
>  <img src="https://i.loli.net/2020/11/24/whY968M1Gtc7WkV.png" alt="image-20201124194659063" style="zoom:50%;" />
>
> 再到后来，Google 开源了 Kubernetes。Kubernetes 作为一个通用的容器编排系统，自然不能只支持 Docker（或者说后来的 containerd）作为唯一的容器运行时，所以 Kubernetes 提出了 CRI（容器运行时接口）。CRI 是一套通过 protocol buffers 定义的 API，只要实现了 CRI 接口，就能够接入 Kubernetes 作为容器运行时。在 Kubernetes 1.5 时，Kubernetes 自己实现了 [docker CRI shim](https://github.com/kubernetes/kubernetes/tree/release-1.5/pkg/kubelet/dockershim)。那么此时 Docker in Kubernetes 的启动流程就如下：
>
> <img src="https://i.loli.net/2020/11/24/4ew9HNg5pmyxSV6.png" alt="img" style="zoom: 50%;" />
>
> 后来 containerd 自己实现了新的 daemon，叫做 [cri-containerd](https://github.com/containerd/cri)，实现了 CRI 接口，使得 kubelet 可以直接绕过 dockerd 直接和 containerd 通信。
>
> <img src="https://i.loli.net/2020/11/24/moXQd6Kjv1wS7eV.png" alt="img" style="zoom:50%;" />
>
> 再到后来，containerd 终于实现了内建的 CRI 插件，kubelet 也终于能直接和 containerd 进程通信了。
>
> <img src="https://i.loli.net/2020/11/24/jWXEqs2MuUAScnB.png" alt="img" style="zoom:50%;" />
>
> 但是这种状态还是很奇怪，kubelet 需要对接的是 OCI 容器运行时，而不是 containerd，容器的网络、存储等管理也可以由 Kubernetes 或者更具体的 kubelet 完成。为了减少不必要的中间开销，社区孵化了 [CRI-O](https://github.com/cri-o/cri-o) 并加入了 CNCF，CRI-O 的目标是直接让 kubelet 和 OCI 容器运行时对接，镜像管理，存储管理和网络均使用标准化的实现。
>
> 下面这张图可以展示出容器编排领域的事实标准 Kubernetes 的容器管理代理 kubelet 和单机容器运行时之间的关系：
>
> <img src="https://i.loli.net/2020/11/24/cH18zJYB5Elrhba.png" alt="img"  />



### 2. OCI 镜像格式标准

[OCI 镜像格式标准](https://github.com/opencontainers/image-spec) 规定了符合 OCI 标准的镜像格式必须、建议、禁止的格式和处理方式。主要分为以下几个部分：

- layer（changeset）：描述镜像层级的文件系统。每个 layer 保存了和上层之间的差异部分，包括新增、修改、删除文件的表示。
- config 文件：镜像的完整描述信息，包括构建环境、构建历史、环境变量、启动命令等所有配置。
- manifest 文件：镜像 config 和 layer 的描述文件，可以完成描述镜像的配置为文件内容。
- index 文件：manifest 列表，保存同一镜像在不同平台的 manifest。

Docker 也相应地实现了上述几个部分。



### 3. Docker 镜像格式

#### 3.1 格式概览

关于 Docker 镜像的基本格式，相关资料已经足够多，这里只简单概述一下。Docker 镜像和容器本质上都是由多个文件镜像层（后文称 layer）从底向上依次迭加（最顶层为读写层，其余均为只读层）形成的文件视图，下层的文件会被上层的同名文件覆盖（删除也是一种覆盖）。在这个文件视图内部用户可以进行随意的修改，这些修改不会反应到底层只读层，而是读写层。我们常说的镜像，就是由别人已经构建好的多个只读层组成的 layer 集合，而容器，就是在原有 layer 之上新增了一个可读写的 layer，然后在本地运用联合挂载的方式，将 layer 组合。所以，镜像存储的本质就是 layer 配置与数据的存储。

![preview](https://i.loli.net/2020/11/25/LEXSbZMsezTj2Yr.png)

#### 3.2 manifest

我们先从一个命令开始。

`docker manifest` 子命令表示与 manifest 相关的操作。Manifest 为镜像的描述信息，包括镜像的大小、摘要等信息。

这里只用到 `docker manifest inspect` 这个子命令（需要首先开启 server 和 client 的实验模式，`docker version` 命令可以返回当前实验模式是否开启），用于查看镜像的 manifest，返回的结果可能为 2 种格式，`application/vnd.docker.distribution.manifest.v2+json` 或 `application/vnd.docker.distribution.manifest.list.v2+json`，前者是一个单纯的 manifest，后者是 manifest 列表，列表内通常是同一个镜像的不同操作系统和 CPU 架构下的不同版本。

> Docker 规定了一些自定义的媒体类型，用来描述镜像。包括：
>
> - `application/vnd.docker.distribution.manifest.v1+json`：v1 版本的 manifest
> - `application/vnd.docker.distribution.manifest.v2+json`：v2 版本的 manifest
> - `application/vnd.docker.distribution.manifest.list.v2+json`：manifest 列表，对应 OCI 标准中的 image index
> - `application/vnd.docker.container.image.v1+json`：镜像 config，对应 OCI 标准中的 image config
> - `application/vnd.docker.image.rootfs.diff.tar.gzip`：压缩格式的 layer 文件
> - `application/vnd.docker.image.rootfs.foreign.diff.tar.gzip`：压缩格式的 layer 文件，但是不能被推送到镜像仓库
> - `application/vnd.docker.plugin.v1+json`：插件配置文件
>
> 这些媒体类型可以在向镜像仓库发送请求时加入到头部的 Accept 字段，用来表示请求的媒体类型。

比如，假如分别执行 `docker manifest inspect nginx:1.10` 和 `docker manifest inspect ubuntu:bionic`，我们会发现这两条命令返回的数据格式完全不同：

```shell
$ docker manifest inspect nginx:1.10
{
	"schemaVersion": 2,
	"mediaType": "application/vnd.docker.distribution.manifest.v2+json",
	"config": {
		"mediaType": "application/vnd.docker.container.image.v1+json",
		"size": 3694,
		"digest": "sha256:0346349a1a640da9535acfc0f68be9d9b81e85957725ecb76f3b522f4e2f0455"
	},
	"layers": [
		{
			"mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
			"size": 51438476,
			"digest": "sha256:6d827a3ef358f4fa21ef8251f95492e667da826653fd43641cef5a877dc03a70"
		},
		{
			"mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
			"size": 19959345,
			"digest": "sha256:1e3e18a64ea9924fd9688d125c2844c4df144e41b1d2880a06423bca925b778c"
		},
		{
			"mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
			"size": 194,
			"digest": "sha256:556c62bb43ac9073f4dfc95383e83f8048633a041cb9e7eb2c1f346ba39a5183"
		}
	]
}

$ docker manifest inspect ubuntu:bionic
{
   "schemaVersion": 2,
   "mediaType": "application/vnd.docker.distribution.manifest.list.v2+json",
   "manifests": [
      {
         "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
         "size": 943,
         "digest": "sha256:45c6f8f1b2fe15adaa72305616d69a6cd641169bc8b16886756919e7c01fa48b",
         "platform": {
            "architecture": "amd64",
            "os": "linux"
         }
      },
      {
         "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
         "size": 943,
         "digest": "sha256:e80b8affb2361dc632c1fa8fcbf6b6514f750eb6ef99b7e7f825a55f849bfd89",
         "platform": {
            "architecture": "arm",
            "os": "linux",
            "variant": "v7"
         }
      },
      ...
   ]
}
```

实际上这是 docker client 在展示返回的数据时，进行了简化，可以在运行 `docker manifest inspect` 命令时加上 `-v` 参数来展示返回的详细信息。

```shell
$ docker manifest inspect nginx:1.10 -v
{
	"Ref": "docker.io/library/nginx:1.10",
	"Descriptor": {
		"mediaType": "application/vnd.docker.distribution.manifest.v2+json",
		"digest": "sha256:6202beb06ea61f44179e02ca965e8e13b961d12640101fca213efbfd145d7575",
		"size": 948,
		"platform": {
			"architecture": "amd64",
			"os": "linux"
		}
	},
	"SchemaV2Manifest": {
		"schemaVersion": 2,
		"mediaType": "application/vnd.docker.distribution.manifest.v2+json",
		"config": {
			"mediaType": "application/vnd.docker.container.image.v1+json",
			"size": 3694,
			"digest": "sha256:0346349a1a640da9535acfc0f68be9d9b81e85957725ecb76f3b522f4e2f0455"
		},
		"layers": [
			{
				"mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
				"size": 51438476,
				"digest": "sha256:6d827a3ef358f4fa21ef8251f95492e667da826653fd43641cef5a877dc03a70"
			},
			{
				"mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
				"size": 19959345,
				"digest": "sha256:1e3e18a64ea9924fd9688d125c2844c4df144e41b1d2880a06423bca925b778c"
			},
			{
				"mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
				"size": 194,
				"digest": "sha256:556c62bb43ac9073f4dfc95383e83f8048633a041cb9e7eb2c1f346ba39a5183"
			}
		]
	}
}

$ docker manifest inspect ubuntu:bionic -v
[
	{
		"Ref": "docker.io/library/ubuntu:bionic@sha256:45c6f8f1b2fe15adaa72305616d69a6cd641169bc8b16886756919e7c01fa48b",
		"Descriptor": {
			"mediaType": "application/vnd.docker.distribution.manifest.v2+json",
			"digest": "sha256:45c6f8f1b2fe15adaa72305616d69a6cd641169bc8b16886756919e7c01fa48b",
			"size": 943,
			"platform": {
				"architecture": "amd64",
				"os": "linux"
			}
		},
		"SchemaV2Manifest": {
			"schemaVersion": 2,
			"mediaType": "application/vnd.docker.distribution.manifest.v2+json",
			"config": {
				"mediaType": "application/vnd.docker.container.image.v1+json",
				"size": 3352,
				"digest": "sha256:56def654ec22f857f480cdcc640c474e2f84d4be2e549a9d16eaba3f397596e9"
			},
			"layers": [
				{
					"mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
					"size": 26701612,
					"digest": "sha256:171857c49d0f5e2ebf623e6cb36a8bcad585ed0c2aa99c87a055df034c1e5848"
				},
				{
					"mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
					"size": 852,
					"digest": "sha256:419640447d267f068d2f84a093cb13a56ce77e130877f5b8bdb4294f4a90a84f"
				},
				{
					"mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
					"size": 162,
					"digest": "sha256:61e52f862619ab016d3bcfbd78e5c7aaaa1989b4c295e6dbcacddd2d7b93e1f5"
				}
			]
		}
	},
	{
		"Ref": "docker.io/library/ubuntu:bionic@sha256:e80b8affb2361dc632c1fa8fcbf6b6514f750eb6ef99b7e7f825a55f849bfd89",
		"Descriptor": {
			"mediaType": "application/vnd.docker.distribution.manifest.v2+json",
			"digest": "sha256:e80b8affb2361dc632c1fa8fcbf6b6514f750eb6ef99b7e7f825a55f849bfd89",
			"size": 943,
			"platform": {
				"architecture": "arm",
				"os": "linux",
				"variant": "v7"
			}
		},
		"SchemaV2Manifest": {
			"schemaVersion": 2,
			"mediaType": "application/vnd.docker.distribution.manifest.v2+json",
			"config": {
				"mediaType": "application/vnd.docker.container.image.v1+json",
				"size": 3313,
				"digest": "sha256:32114f98a2fcee7d1d3b0c117139fab1760535f54c1ad8397903441834fc4358"
			},
			"layers": [
				{
					"mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
					"size": 22279517,
					"digest": "sha256:20e126218ac644f56ef7147c3363108a0814d921e6016af54a1b4c964159f1a9"
				},
				{
					"mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
					"size": 852,
					"digest": "sha256:3d156c2b31482935ec0363b6e3f1cb6fc56da57e61fc80078914918fe53f8fa5"
				},
				{
					"mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
					"size": 186,
					"digest": "sha256:93c1a0dbe2162972438aa89d4f90dca5db0e4cee58819ba354ea1c0031101b7a"
				}
			]
		}
	},
	...
]
```

可以看出，两条命令返回的数据格式基本类似，只不过前者返回的是一个 `application/vnd.docker.distribution.manifest.v2+json` 对象，后者返回的是 `application/vnd.docker.distribution.manifest.v2+json` 对象列表，也就是 `application/vnd.docker.distribution.manifest.list.v2+json`。

下面观察 `application/vnd.docker.distribution.manifest.v2+json` 对象：

```json
{
	"Ref": "docker.io/library/ubuntu:bionic@sha256:45c6f8f1b2fe15adaa72305616d69a6cd641169bc8b16886756919e7c01fa48b",
	"Descriptor": {
		"mediaType": "application/vnd.docker.distribution.manifest.v2+json",
		"digest": "sha256:45c6f8f1b2fe15adaa72305616d69a6cd641169bc8b16886756919e7c01fa48b",
		"size": 943,
		"platform": {
			"architecture": "amd64",
			"os": "linux"
		}
	},
	"SchemaV2Manifest": {
		"schemaVersion": 2,
		"mediaType": "application/vnd.docker.distribution.manifest.v2+json",
		"config": {
			"mediaType": "application/vnd.docker.container.image.v1+json",
			"size": 3352,
			"digest": "sha256:56def654ec22f857f480cdcc640c474e2f84d4be2e549a9d16eaba3f397596e9"
		},
		"layers": [
			{
				"mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
				"size": 26701612,
				"digest": "sha256:171857c49d0f5e2ebf623e6cb36a8bcad585ed0c2aa99c87a055df034c1e5848"
			},
			{
				"mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
				"size": 852,
				"digest": "sha256:419640447d267f068d2f84a093cb13a56ce77e130877f5b8bdb4294f4a90a84f"
			},
			{
				"mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
				"size": 162,
				"digest": "sha256:61e52f862619ab016d3bcfbd78e5c7aaaa1989b4c295e6dbcacddd2d7b93e1f5"
			}
		]
	}
}
```

其中：

- **`Ref`**：镜像引用信息，为了区分同一镜像在不同操作系统和 CPU 架构，这里的镜像 tag 已经变成了摘要的形式（具体摘要是如何计算出来的后面会讲到）；

- **`Descriptor`**：描述指向的对象的信息。实质上是 [OCI 标准中的 descriptor](https://github.com/opencontainers/image-spec/blob/b6e51fa50549ee0bd5188494912a7f4c382cb0d4/descriptor.md) 在 Docker 中的实现。在上面这个例子中，Descriptor 表示指向的对象是一个 `application/vnd.docker.distribution.manifest.v2+json` 对象，也就是镜像的原始 manifest，该 manifest 文件的摘要是 `sha256:45c6f8f1b2fe15adaa72305616d69a6cd641169bc8b16886756919e7c01fa48b`。在本地执行 `docker image --digests | grep 45c6f8f1b2fe15adaa72305616d69a6cd641169bc8b16886756919e7c01fa48b` 就可以看到该摘要对应的镜像。所以在 amd64 架构、Linux 内核下的 `ubuntu:bionic` 镜像也可以表示成 `ubuntu:bionic@sha256:45c6f8f1b2fe15adaa72305616d69a6cd641169bc8b16886756919e7c01fa48b`。这就是上面提到的 `Ref` 中的后半部分内容；

- **`SchemaV2Manifest`**：表示 **manifest 的真正内容**，这段内容的摘要就是 `Descriptor` 中描述的 `sha256:45c6f8f1b2...`

  > 下面验证一下关于 SchemaV2Manifest 的描述是否正确
  >
  > ```shell
  > $ docker tag ubuntu:bionic 127.0.0.1:5000/ubuntu:bionic
  > $ docker push 127.0.0.1:5000/ubuntu:bionic
  > 
  > # 查看远程仓库返回的 manifest 是否与 docker inspect 返回的列表中的值一致
  > $ curl -H "Accept:application/vnd.docker.distribution.manifest.v2+json" http://127.0.0.1:5000/v2/ubuntu/manifests/bionic
  > 
  > # 可以看到 manifest 的摘要值与 Descriptor 中描述的一致
  > $ curl -H "Accept:application/vnd.docker.distribution.manifest.v2+json" http://127.0.0.1:5000/v2/ubuntu/manifests/bionic | sha256sum
  > 45c6f8f1b2fe15adaa72305616d69a6cd641169bc8b16886756919e7c01fa48b  -
  > ```

  - `schemaVersion`：[OCI 规定的 schema 版本号](https://github.com/opencontainers/image-spec/blob/b6e51fa50549ee0bd5188494912a7f4c382cb0d4/manifest.md#image-manifest-property-descriptions)；

  - `mediaType`：对象类型，Docker 实现的对象类型在[官方文档](application/vnd.docker.distribution.manifest.v2+json)中有说明，一共 7 种；

  - `config`：表示镜像 config（OCI 中的 [config](https://github.com/opencontainers/image-spec/blob/b6e51fa50549ee0bd5188494912a7f4c382cb0d4/config.md) 在 Docker 中的实现）的元信息；

    - `mediaType`：这里的对象类型固定为 `application/vnd.docker.container.image.v1+json`，表示镜像 config；

    - `size`：镜像 config 的大小，单位为字节；

    - `digest`：镜像 config 的摘要。**这个值就是我们通常所说的镜像 ID（docker images 输出的第一列）**。还是以上面的输出为例，我们可以尝试执行 `docker images | grep 56def `，我们会发现这个摘要对应的就是 `ubuntu:bionic` 这个镜像。 

      > *这就意味着，docker client 其实是以镜像 config 来区分不同的镜像的。*
      >
      > *而且，每一个本地保存的镜像，都会有一个对应的镜像 config，保存在 `<Docker根目录>/image/<Docker存储驱动，默认为overlay2>/imagedb/content/sha256/<digest>`中，我们可以自行计算一下本地镜像 config 的摘要，观察其是否与文件名一致。*

  - `layers`：指向 `application/vnd.docker.image.rootfs.diff.tar.gzip` 对象的 `Descriptor` 列表。每个 `application/vnd.docker.image.rootfs.diff.tar.gzip` 都对应一个压缩归档的 layer，layer 按照从底向上的顺序依次保存，即第一个 layer 为最底层的 layer。其实这里的 digest 就是执行 `docker pull` 时输出的层级信息，且顺序一致，也是从底向上。

    > ```shell
    > $ docker pull ubuntu:bionic
    > bionic: Pulling from library/ubuntu
    > 171857c49d0f: Pull complete # SchemaV2Manifest.layers[0].digest
    > 419640447d26: Pull complete # SchemaV2Manifest.layers[1].digest
    > 61e52f862619: Pull complete # SchemaV2Manifest.layers[2].digest
    > Digest: sha256:646942475da61b4ce9cc5b3fadb42642ea90e5d0de46111458e100ff2c7031e6
    > Status: Downloaded newer image for ubuntu:bionic
    > docker.io/library/ubuntu:bionic
    > ```

    > *我们可以尝试着下载 ubuntu:bionic 这个镜像的最底层 layer，验证一下上面提到的内容。*
    >
    > ```shell
    > # 下载 tgz 格式的 layer
    > $ curl -H "Accept:application/vnd.docker.image.rootfs.diff.tar.gzip" http://127.0.0.1:5000/v2/ubuntu/blobs/sha256:171857c49d0f5e2ebf623e6cb36a8bcad585ed0c2aa99c87a055df034c1e5848 -o ubuntu.1.tgz
    > 
    > # 验证摘要信息，可以见到和第一个 Descriptor 的 digest 字段完全一致
    > $ sha256sum ubuntu.1.tgz
    > 171857c49d0f5e2ebf623e6cb36a8bcad585ed0c2aa99c87a055df034c1e5848  ubuntu.1.tgz
    > ```

以上就是关于 manifest 文件的内容。总的来说，manifest 文件描述了一个镜像的 config 信息（config 字段）、layer 信息（layers 字段），其中 config 信息描述了镜像的元数据，包括作者、配置、构建历史等，layer 信息保存了镜像需要的数据，通过这两部分我们可以完整地复现出镜像构建时的状态，从而保证了容器内部的一致性。

而且我们可以看出 Docker 镜像（其实不仅仅是 Docker，而是所有符合 OCI 规范的镜像格式）通过层层的摘要来保证镜像数据的完整性，防止被篡改。具体过程就是**分别对 config 和各 layer 取摘要，保存到 manifest 中。然后对 manifest 取摘要，保存到返回数据中的 Descriptor 中。最终通过在镜像仓库与客户端中传输 Descriptor 列表**。那假如 Descriptor 被篡改了呢？这就不是应用层需要负责的问题了，因为应用层（也就是 Docker）传输的载体是 Descriptor，Descriptor 的完整性需要有传输层来保证。

> 这里需要再次提一点，在实验途中，我们为 `ubuntu:bionic` 这个镜像重新打上标签 `127.0.0.1:5000/ubuntu`，如果此时观察这 2 个相同镜像的摘要，会出现以下现象：
>
> ```shell
> $ docker images --digests | grep bionic
> 127.0.0.1:5000/ubuntu                           bionic              sha256:45c6f8f1b2fe15adaa72305616d69a6cd641169bc8b16886756919e7c01fa48b   56def654ec22        8 weeks ago         63.2MB
> ubuntu                                          bionic              sha256:646942475da61b4ce9cc5b3fadb42642ea90e5d0de46111458e100ff2c7031e6   56def654ec22        8 weeks ago         63.2MB
> ```
>
> 镜像 ID 相同，但是摘要值不同。为什么会出现这样的变化呢？目前我没有办法成功复现出 `6469` 这个摘要值的计算过程。
>
> 但是通过观察 `docker manifest inspect ubuntu:bionic -v` 的输出我们可以发现，`45c6` 这个摘要值其实是 `6469` 这个摘要的一个子集，仅表示在 amd64 的 linux 下的镜像的 manifest 摘要值。
>
> ![image-20201124170116237](https://i.loli.net/2020/11/24/DmtWUAzXjMT97s8.png)
>
> 所以我们可以合理推测，在执行 `docker tag` 时，docker client 只会为符合本宿主机的镜像重新打上标签，甚至推送到镜像仓库的时候，也只会把标签对应的镜像推送到仓库。这就能够解释为什么从镜像仓库仓库下载的 manifest 摘要值也是 `45c6`。
>
> 所以可以推测，原始摘要值 `6469` 是镜像的 manifest 列表的摘要值，包含了镜像的所有平台下的分支版本，而新摘要值 `45c6` 只是镜像的 amd64 架构的 Linux 内核版本的 manifest 摘要值，这个值是前者的一个子集。

通过上面的讨论，我们就能够读懂 `docker pull/push` 和 `docker manifest ` 中的所有摘要的指向。如下图所示：

![image-20201124204241392](https://i.loli.net/2020/11/25/Y7A8Ka5ihSrlMUt.png)

![image-20201124204422695](https://i.loli.net/2020/11/25/1qILKahpEDA37iX.png)



### 4. Docker 镜像存储

#### 4.1 本地存储

第 3 节介绍的 manifest 是 Docker 镜像的描述文件，用于在镜像仓库与本地传输时的镜像描述。

当镜像被存储在本地时，Docker daemon 会按照不同的 ID 以[基于内容的寻址方式](https://en.wikipedia.org/wiki/Content-addressable_storage)存储。

这里需要详细介绍 Docker 中不同的 ID：

- **Manifest ID**：**镜像 manifest 的摘要 ID**。其实在 OCI 和 Docker 都没有 manifest ID 这个概念，这里只是随便起了个名字。`docker manifest inspect -v ` 或 `docker images --digests` 或者对从镜像仓库中下载的 manifest 取摘要值都会看到这个 id。这个 ID 只会在 manifest list 和本地出现。

  ![image-20201124225800683](https://i.loli.net/2020/11/25/EVeh27PqOr4tSlR.png)

- **Image ID**：**镜像 ID**。也就是 `docker images` 输出的 ID，这个 ID 是根据镜像 config 计算出的，只要镜像不变，镜像 ID 也不会变化。这个 ID 会在镜像 manifest、本地出现。

  ![image-20201124221042889](https://i.loli.net/2020/11/25/jtNb8Tlk9LmrKD1.png)

- **Layer Digest**：manifest 中出现的 **layer 归档压缩后的摘要**。这个 ID 会在 manifest 和本地出现。

  ![image-20201125100926017](https://i.loli.net/2020/11/25/rwbe34Kguj1vBmQ.png)

- **Diff ID**：用于描述**每一层 layer 的未压缩状态的摘要值**。Diff ID 可以通过执行 `docker image inspect` 或者本地镜像 config 文件来查看。**注意区分 diffID 和 manifest 中 layers 中的 digest 值，前者是 layer 未压缩目录（即 tar 文件）的摘要，且只会出现在镜像 config 中，后者是 layer 归档压缩后（即 tgz 文件）的摘要，会出现在镜像 manifest 中**。这个 ID 会在镜像 config、本地出现。

  ![image-20201124223034655](https://i.loli.net/2020/11/24/MYbWkVhq4Npfj73.png)

- **Chain ID**：用于描述**当前 layer 及其下所有 layer 的摘要值**。这个值只会在本地出现。Chain ID 可以由当前 layer 及其底层 layer 的 Diff ID 计算得出，计算公式如下：

  ```
  // 镜像从底向上依次为 L₀、L₁...Lₙ
  ChainID(L₀) =  DiffID(L₀)
  ChainID(L₀|...|Lₙ₋₁|Lₙ) = Digest(ChainID(L₀|...|Lₙ₋₁) + " " + DiffID(Lₙ))
  ```

  ![image-20201124223956218](https://i.loli.net/2020/11/24/heuEJHV2as7WNPx.png)

- **Cache ID（仅在本地唯一）**：layer 在本地的存储目录 ID，同一 layer，不同主机，cache id 都不一样。

下图描述了使用 overlay2 存储驱动的 Docker 镜像本地存储目录结构： 

![image-20201126161815719](https://i.loli.net/2020/11/26/cqNQ4Yk6yzjWtCD.png)

其中：

- `containers`：本地已创建的 Docker 容器的配置信息，包括主机名、DNS 配置等。
- `image/<存储驱动>`：保存镜像 layer 元数据，不保存 layer 真实数据，layer 真实数据保存在 `<存储驱动>` 这个目录下。
  - `distribution`：本目录下有 2 个子目录，分别为 `diffid-by-digest` 和 `v2metadata-by-diffid`。从二者的名字就可以看出这两个目录保存的分别是从 layer 的 digest 到其 diffID 的映射和从 layer 的 diffID 到其 metadata 的映射。
  - `repositories.json`：保存本地所有镜像及其镜像 ID。
- `overlay2(<存储驱动>)`：不同的 Docker 存储驱动的目录结构都互不相同，以 overlay2 为例，其下有两种类型的目录：
  - `l目录`：该目录下有众多长 26 位的软链，指向了 `本地缓存目录` 下的 diff 目录。这种表示方式和挂载有关，后面我们会提到，在使用本地 overlay 挂载的时候，会使用本目录下的软链而不是直接引用 `本地缓存目录` 下的 diff 目录，这是因为挂载声明的长度有限制，直接使用后者可能导致长度超过限制值。
  - `本地缓存目录`：layer 的实际存储位置。`image/overlay2/layerdb/sha256` 下的所有 layer 都有一个 `cache-id` 文件，这个 ID 指向的就是本地缓存目录下的 ID。所以可以说 `layerdb` 只存储了 layer 的元数据，而真正的数据存储在存储驱动下的目录中。
    - `diff目录`：保存 layer 实际内容，真正挂载的时候也只会用到不同 layer 的 diff 目录；
    - `link`：保存当前缓存层的短 ID。和上面的 `l目录` 为互相引用的关系；
    - `lower`：保存当前 layer 之下的所有 layer 的短 ID 集合。挂载的时候可以直接读取这个文件，而不用再解析镜像 config；
    - `work目录`：overlay 的 COW 目录。

至此，使用 overlay2 存储驱动的 Docker 引擎的镜像本地存储方式已经大致明了。概述如下：**Docker 镜像的元数据都保存在 `image/overlay2/imagedb` 中，而 Docker 镜像的真正数据通过该路径下的 `cache-id` 文本文件指向 `overlay2` 目录下的缓存目录中，该缓存目录的 `diff` 子目录保存了 layer 的所有文件，正真需要在本地挂载时，挂载对象也是这些 `diff` 目录**。

> 下面验证一下上述内容。我们尝试手动单独下载 `ubuntu:bionic` 这个镜像的第 2 层，来和本地缓存目录中和第 2 层对应的目录进行比较。
>
> ```shell
> # 首先查看 ubuntu:bionic 第 2 层的 digest
> $ docker images --digests | grep bionic
> 127.0.0.1:5000/ubuntu                           bionic              sha256:45c6f8f1b2fe15adaa72305616d69a6cd641169bc8b16886756919e7c01fa48b   56def654ec22        2 months ago        63.2MB
> ubuntu                                          bionic              sha256:646942475da61b4ce9cc5b3fadb42642ea90e5d0de46111458e100ff2c7031e6   56def654ec22        2 months ago        63.2MB
> $ curl -H "Accept:application/vnd.docker.distribution.manifest.v2+json" http://127.0.0.1:5000/v2/ubuntu/manifests/bionic | sha256sum
> 45c6f8f1b2fe15adaa72305616d69a6cd641169bc8b16886756919e7c01fa48b  -
> $ curl -H "Accept:application/vnd.docker.distribution.manifest.v2+json" http://127.0.0.1:5000/v2/ubuntu/manifests/bionic
> {
>    ...
>    "layers": [
>      ...
>       {
>          "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
>          "size": 852,
>          "digest": "sha256:419640447d267f068d2f84a093cb13a56ce77e130877f5b8bdb4294f4a90a84f" # 第 2 层的 digest
>       },
>       ...
>    ]
> }
> 
> # 手动下载
> $ curl -H "Accept:application/vnd.docker.image.rootfs.diff.tar.gzip" http://127.0.0.1:5000/v2/ubuntu/blobs/sha256:419640447d267f068d2f84a093cb13a56ce77e130877f5b8bdb4294f4a90a84f -o /tmp/ubuntu.2.tgz
> $ cd /tmp
> $ gzip -d ubuntu.2.tgz
> 
> # 验证第 2 层的 diffID 是否与镜像 config 中的一致
> $ sha256sum ubuntu.2.tar
> 3fd9df55318470e88a15f423a7d2b532856eb2b481236504bf08669013875de1  ubuntu.2.tar
> $ cat /var/lib/docker/image/overlay2/imagedb/content/sha256/56def654ec22f857f480cdcc640c474e2f84d4be2e549a9d16eaba3f397596e9 | jq 
> ...
>   "rootfs": {
>     "type": "layers",
>     "diff_ids": [
>       "sha256:80580270666742c625aecc56607a806ba343a66a8f5a7fd708e6c4e4c07a3e9b",
>       "sha256:3fd9df55318470e88a15f423a7d2b532856eb2b481236504bf08669013875de1", # 与前一步计算出的 diffID 一致
>       "sha256:7a694df0ad6cc5789a937ccd727ac1cda528a1993387bf7cd4f3c375994c54b6"
>     ]
>   }
> ...
> 
> # 验证文件内容
> $ tar tvf ubuntu.2.tar # 这是手动下载下来的 layer 的数据
> drwxr-xr-x 0/0               0 2020-09-26 06:33 etc/
> drwxr-xr-x 0/0               0 2020-09-22 01:17 etc/apt/
> drwxr-xr-x 0/0               0 2020-09-26 06:33 etc/apt/apt.conf.d/
> -rw-r--r-- 0/0              44 2020-09-26 06:33 etc/apt/apt.conf.d/docker-autoremove-suggests
> -rw-r--r-- 0/0             318 2020-09-26 06:33 etc/apt/apt.conf.d/docker-clean
> -rw-r--r-- 0/0              70 2020-09-26 06:33 etc/apt/apt.conf.d/docker-gzip-indexes
> -rw-r--r-- 0/0              27 2020-09-26 06:33 etc/apt/apt.conf.d/docker-no-languages
> drwxr-xr-x 0/0               0 2020-09-22 01:16 etc/dpkg/
> drwxr-xr-x 0/0               0 2020-09-26 06:33 etc/dpkg/dpkg.cfg.d/
> -rw-r--r-- 0/0              16 2020-09-26 06:33 etc/dpkg/dpkg.cfg.d/docker-apt-speedup
> drwxr-xr-x 0/0               0 2020-09-26 06:33 sbin/
> -rwxr-xr-x 0/0              17 2020-09-26 06:33 sbin/initctl
> drwxr-xr-x 0/0               0 2020-09-22 01:14 usr/
> drwxr-xr-x 0/0               0 2020-09-26 06:33 usr/sbin/
> -rwxr-xr-x 0/0              19 2020-09-26 06:33 usr/sbin/policy-rc.d
> drwxr-xr-x 0/0               0 2020-09-22 01:17 var/
> drwxr-xr-x 0/0               0 2018-04-05 20:27 var/lib/
> drwxr-xr-x 0/0               0 2020-09-26 06:33 var/lib/dpkg/
> -rw-r--r-- 0/0             136 2020-09-26 06:33 var/lib/dpkg/diversions
> -rw-r--r-- 0/0              98 2020-09-22 01:17 var/lib/dpkg/diversions-old
> # 然后我们需要计算出第 2 层的 chainID，然后去 layerdb 下的 chainID 目录中找 cache-id，cache-id 就是本地实际存储的目录
> $ echo -n "sha256:80580270666742c625aecc56607a806ba343a66a8f5a7fd708e6c4e4c07a3e9b sha256:3fd9df55318470e88a15f423a7d2b532856eb2b481236504bf08669013875de1" | sha256sum
> dd44b56f7a8f4d7c34f8fe346f507e46defea98f198bccd13ef227a80a512f18  -
> $ cat /var/lib/docker/image/overlay2/layerdb/sha256/dd44b56f7a8f4d7c34f8fe346f507e46defea98f198bccd13ef227a80a512f18/cache-id
> c8e8fc76e3d1ddec3725f82a924c4ae2f0ade76ae47fef73faa657a8a7a0a9d7
> # 输出本地 layer 的文件内容，对比发现和我们下载的内容完全一致
> $ tree /var/lib/docker/overlay2/c8e8fc76e3d1ddec3725f82a924c4ae2f0ade76ae47fef73faa657a8a7a0a9d7/diff/
> /var/lib/docker/overlay2/c8e8fc76e3d1ddec3725f82a924c4ae2f0ade76ae47fef73faa657a8a7a0a9d7/diff/
> ├── etc
> │   ├── apt
> │   │   └── apt.conf.d
> │   │       ├── docker-autoremove-suggests
> │   │       ├── docker-clean
> │   │       ├── docker-gzip-indexes
> │   │       └── docker-no-languages
> │   └── dpkg
> │       └── dpkg.cfg.d
> │           └── docker-apt-speedup
> ├── sbin
> │   └── initctl
> ├── usr
> │   └── sbin
> │       └── policy-rc.d
> └── var
>     └── lib
>         └── dpkg
>             ├── diversions
>             └── diversions-old
> 
> 11 directories, 9 files
> ```

结合第 3、4 章，我们会发现 OCI 和 Docker 中使用了非常多的摘要，这些摘要也相当容易混淆，这么做的具体原因是什么呢？我们可以总结出下图的结构。可以看出通过逐层计算摘要，我们可以保证最终存储在本地的镜像 config 不被篡改，也能保证 cache 和原始 layer 的 tar 包保持一致。

![image-20201125113831693](https://i.loli.net/2020/11/25/ZPspgabxcfXhToO.png)

#### 4.2 联合挂载

Docker 用到了联合文件系统（UnionFS）来组合多层 layer 形成一个统一的 rootfs。当使用镜像启动容器时，Docker存储驱动会将存储于本地的镜像只读 layer 和一个新创建的读写 layer（另外包含一个 init 只读 layer，存储容器的 DNS 等配置）联合挂载为联合文件系统。如下图所示。

![镜像和容器 roofs 示意图](https://i.loli.net/2020/11/25/lNbYTxvitsREFdh.png)

Docker 支持的存储驱动包括 overlay、aufs、device mapper 等。这里简单介绍 overlay 的挂载方式。Docker 所用的 overlay 存储驱动将各 layer 联合挂载为 OverlayFS，挂载的结果可以通过执行 `mount | grep overlay` 来查看。比如下面这条语句就是我本地的一个容器的挂载声明。

```shell
overlay on /var/lib/docker/overlay2/d9b9992ddd1eee0f71995eeab4e02a2873f5c8eb7049c9235c3a88b44e7109b8/merged type overlay (rw,relatime,seclabel,lowerdir=/var/lib/docker/overlay2/l/MJUZFEGDIVNV7BF6OEN4NNIA2L:/var/lib/docker/overlay2/l/G5R4F3JYNU3F27FLR2MI2TCCR4:/var/lib/docker/overlay2/l/JB4LF7RXORGZGDHRVF2KEDHEER:/var/lib/docker/overlay2/l/P7L7BSMEQS27CURZPU7AUFPQ6K:/var/lib/docker/overlay2/l/GQ4UJCDPAZPU6K63LJ3XXS5VB6:/var/lib/docker/overlay2/l/VAB3S6XBK52PO4KYVWHTWAHXXQ,upperdir=/var/lib/docker/overlay2/d9b9992ddd1eee0f71995eeab4e02a2873f5c8eb7049c9235c3a88b44e7109b8/diff,workdir=/var/lib/docker/overlay2/d9b9992ddd1eee0f71995eeab4e02a2873f5c8eb7049c9235c3a88b44e7109b8/work)
```

括号内表示了挂载时声明的一些属性，其中 `upperdir`、`lowerdir`、`workdir`分别表示顶层读写层、底层只读层（可以声明多个只读层，以`:`分隔）、以及用于存放临时文件的工作基础目录。

具体到上述例子，挂载点为 `/var/lib/docker/overlay2/d9b9.../merged` ，这个目录就是多层迭加形成的容器内的文件系统的根目录。`upperdir` 也为 `d9b9...` 这个 layer 的 diff 目录，表示本层所保存的内容，`lowerdir` 为 `/var/lib/docker/overlay2/d9b9.../lower` 里的内容，`workdir` 也是 `d9b9...` 这个 layer 的 work 目录。`d9b9...` 这个 ID 会被保存到 `/var/lib/docker/image/overlay2/layerdb/mounts/<容器ID>/mount-id` 这个文本文件里。



### 5. 总结

Docker 镜像作为 Docker 容器引擎的基础，Docker 镜像通过分层与联合的理念，将虚拟环境的文件系统以层级的方式分隔，创建、传输与删除都是以不可变的层级为单位，这即提高了传输效率，也使得容器的构建更灵活。OCI 开始制定后，Docker 将其 v2 版本的镜像格式贡献给 OCI，等到 OCI 容器镜像标准 v1 发布以后，Docker 镜像的设计理念也随着 OCI 的传播而辐射到整个容器领域。

本文深入地分析了 Docker 镜像的格式以及本地存储规则，希望读完本文后，我们对于它们的认识能够加深一层，为以后可能的二次开发打下一个良好的基础。



### 参考链接

[1] [开放容器标准(OCI) 内部分享](https://xuanwo.io/2019/08/06/oci-intro/)

[2] [Docker Leads OCI Release of v1.0 Runtime and Image Format Specifications](https://www.docker.com/blog/oci-release-of-v1-0-runtime-and-image-format-specifications/)

[3] [容器开放接口规范概述](https://www.cnblogs.com/xiaochina/p/12782651.html)

[4] [Image Manifest V 2, Schema 2](https://docs.docker.com/registry/spec/manifest-v2-2/)

[5] [镜像格式二十年：从 Knoppix 到 OCI-Image-v2](https://www.sofastack.tech/blog/twenty-years-of-image-format-from-knoppix-to-oci-image-v2/)


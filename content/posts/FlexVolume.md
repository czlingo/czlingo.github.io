---
title: "FlexVolume"

categories: [
    "kubernetes"
]
---

在 Kubernetes 中，存储插件的开发有两种方式： FlexVolume 与 CSI。

本篇文章主要讲述 FlexVolume， 通过使用 NFS 实现 FlexVolume 插件，了解 FlexVolume 的原理和使用方法。

对于一个 FlexVolume 类型的 PV 来说， 它的 YAML 文件如下所示：

```json
apiVersion: v1
kind: PersistentVolume
metadata: 
    name: pv-flex-nfs
spec:
    capacity:
        storage: 10Gi
    accessMode:
      - ReadWrtieMany
    flexVolume:
        driver: "k8s/nfs"
        fsType: "nfs"
        options:
            server: "172.17.0.2"
```

这个 PV 定义的 Volume 类型是 flexVolume。并且，指定了这个 Volume 的 driver 叫作 k8s/nfs。

而 Volume 的 options 字段，则是一个自定义字段。它的类型其实是 map[string]string。所以可以在 options 中加入想要自定义的参数。

当这个 PV 被创建与某个 PVC 绑定时，就会进入到 Volume 处理流程，这个流程中有两阶段处理，即“Attach阶段” 和 “Mount阶段”。它们的作用是在 Pod 所绑定的宿主机上，完成这个 Volume 目录的持久化过程，比如为虚拟机挂载磁盘(Attach)，或者挂在一个 NFS 的共享目录(Mount)。

在具体的控制循环中，这两个操作实际上调用的正是 Kubernetes 的 pkg/volume 目录下的存储插件(Volume Plugin)。这里用到的就是 flexvolume。

以“Mount阶段”为例，而这里面主要是封装出一行命令(NewDriverCall)，由 kubelet 在 “Mount阶段” 去执行。 这个例子中， Kubelete 要通过插件在宿主机上执行的命令为

```
/usr/libexec/kubernetes/kubelet-plugins/volume/exec/k8s~nfs/nfs mount <mount dir> <json param>
```

其中， /usr/libexec/kubernetes/kubelet-plugins/volume/exec/k8s~nfs/nfs 就是插件的可执行文件的路径。这个名叫 nfs 的文件，正是需要编写的插件的实现。这个路径中的 k8s~nfs 部分，正是插件在 kubernetes 里的名字。即 driver="k8s/nfs" 字段。

“mount” 参数，定义的就是当前的操作。在 FlexVolume 里，这些操作参数的名字是固定的，比如 init、mount、unmount、attach 以及 dettach 等等，分别对应不同的 Volume 处理操作。

mount 参数后面的两个字段： <mount dir> 和 <json params>，则是 FlexVolume 必须提供给这条命令的两个执行参数。

其中第一个执行参数 <mount dir>，正是 kubelet 调用 SetUpAt() 方法传递的 dir 的值。它代表的是当前正在处理的 Volume 在宿主机上的目录。这个例子中，这个路径如下：

```json
/var/lib/kubelet/pods/<Pod ID>/volumes/k8s~nfs/test
```

test 正是前面定义的 PV 的名字；而 k8s~nfs，则是插件的名字。

<json params> 则是一个 JSON Map 格式的参数列表。前面所定义的 options 字段的值，都会被追加在这个参数里。从 /pkg/volume/flexvolume/mounter.go SetUpAt() 方法里可以看到，这个参数列表里还包括了 Pod 的名字、Namepace 等元数据。

以 shell 脚本作为插件的实现：

```shell
#!/bin/bash

# Copyright 2015 The Kubernetes Authors.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

# ============================ Example ================================
# apiVersion: v1
# kind: Pod
# metadata:
#   name: busybox
#   namespace: default
# spec:
#   containers:
#   - name: busybox
#     image: busybox
#     command:
#       - sleep
#       - "3600"
#     imagePullPolicy: IfNotPresent
#     volumeMounts:
#     - name: test
#       mountPath: /data
#   volumes:
#   - name: test
#     flexVolume:
#       driver: "k8s/nfs"
#       fsType: "nfs"
#       options:
#         server: nfs.example.k8s.io
#         share: "/share/test1"
# =====================================================================


# Notes:
#  - Please install "jq" package before using this driver.
usage() {
	err "Invalid usage. Usage: "
	err "\t$0 init"
	err "\t$0 mount <mount dir> <json params>"
	err "\t$0 unmount <mount dir>"
	exit 1
}

err() {
	echo -ne $* 1>&2
}

log() {
	echo -ne $* >&1
}

ismounted() {
	MOUNT=`findmnt -n ${MNTPATH} 2>/dev/null | cut -d' ' -f1`
	if [ "${MOUNT}" == "${MNTPATH}" ]; then
		echo "1"
	else
		echo "0"
	fi
}

domount() {
	MNTPATH=$1

	local NFS_SERVER=$(echo $2 | jq -r '.server')
	local SHARE=$(echo $2 | jq -r '.share')
	local PROTOCOL=$(echo $2 | jq -r '.protocol')
	local ATIME=$(echo $2 | jq -r '.atime')
	local READONLY=$(echo $2 | jq -r '.readonly')

	if [ -n "${PROTOCOL}" ]; then
		PROTOCOL="tcp"
	fi

	if [ -n "${ATIME}" ]; then
		ATIME="0"
	fi

	if [ -n "${READONLY}" ]; then
		READONLY="0"
	fi

	if [ "${PROTOCOL}" != "tcp" ] && [ "${PROTOCOL}" != "udp" ] ; then
		err "{ \"status\": \"Failure\", \"message\": \"Invalid protocol ${PROTOCOL}\"}"
		exit 1
	fi

	if [ $(ismounted) -eq 1 ] ; then
		log '{"status": "Success"}'
		exit 0
	fi

	mkdir -p ${MNTPATH} &> /dev/null

	local NFSOPTS="${PROTOCOL},_netdev,soft,timeo=10,intr"
	if [ "${ATIME}" == "0" ]; then
		NFSOPTS="${NFSOPTS},noatime"
	fi

	if [ "${READONLY}" != "0" ]; then
		NFSOPTS="${NFSOPTS},ro"
	fi

	mount -t nfs -o${NFSOPTS} ${NFS_SERVER}:/${SHARE} ${MNTPATH} &> /dev/null
	if [ $? -ne 0 ]; then
		err "{ \"status\": \"Failure\", \"message\": \"Failed to mount ${NFS_SERVER}:${SHARE} at ${MNTPATH}\"}"
		exit 1
	fi

    # 需要注意的是，当这个 mount -t nfs 操作完成后， 必须把一个 JSON 格式的字符串，
    # '{"status": "Success"}' 返回给调用者，也就是 kubelet。 这是 kubelet 判断这次调用是否成功的唯一依据。
	log '{"status": "Success"}'
	exit 0
}

unmount() {
	MNTPATH=$1
	if [ $(ismounted) -eq 0 ] ; then
		log '{"status": "Success"}'
		exit 0
	fi

	umount ${MNTPATH} &> /dev/null
	if [ $? -ne 0 ]; then
		err "{ \"status\": \"Failed\", \"message\": \"Failed to unmount volume at ${MNTPATH}\"}"
		exit 1
	fi

	log '{"status": "Success"}'
	exit 0
}

op=$1

if ! command -v jq >/dev/null 2>&1; then
	err "{ \"status\": \"Failure\", \"message\": \"'jq' binary not found. Please install jq package before using this driver\"}"
	exit 1
fi

if [ "$op" = "init" ]; then
	log '{"status": "Success", "capabilities": {"attach": false}}'
	exit 0
fi

if [ $# -lt 2 ]; then
	usage
fi

shift

case "$op" in
	mount)
		domount $*
		;;
	unmount)
		unmount $*
		;;
	*)
		log '{"status": "Not supported"}'
		exit 0
esac

exit 1
```

当 kubelet 在宿主机上执行 “nfs mount <mount dir> <json params>” 时，这个名叫 nfs 的脚本， 就可以直接从 <mount dir> 参数里拿到 Volume 在宿主机上的目录，即 MNTPATH 

在 “Mount 阶段”， kubelet 的 VolumeManagerReconcile 控制循环里的一次 reconcile 操作的执行流程，如下：

```
kubelet --> pkg/volume/flexvolume.SetUpAt() --> /usr/libexec/kubernetes/kubelet-plugins/volume/exec/k8s~nfs/nfs mount <mount dir> <json param>
```

PS：像 NFS 这样的文件系统存储，不需要在宿主机上挂在磁盘或者块设备。所以不要执行 attach 和 dettach 操作。

### flexVolume的局限性

flexVolume 实现方式简单但局限性很大。比如不能支持 Dynamic Provisioning。或者在 mount 操作时，可能会产生一些挂在信息。这些信息，在后面执行 unmount 操作的时候会被用到。但在 flexVolume 的实现里，没办法把这些信息保存在一个变量里，因为每一次对插件的调用都是一次独立的操作。只能将这些信息写到宿主机的临时文件里，等到需要的时候再去读取。
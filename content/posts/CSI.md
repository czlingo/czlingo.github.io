---
title: "CSI"
catagories: [
    "kubernetes"
]
---

通过 FlexVolume 已经能够实现存储插件的扩展，那为什么还需要 CSI 这种方式呢？在 FlexVolume 文章中可以主要到，每一次对 FlexVolume 插件的调用都是一次完全独立的操作。而如果需要保存一些重要的信息，就只能将这些信息写入到宿主机的临时文件里，等到 unmount 的阶段的时候再去读取。

这也是为什么需要 Container Storage Interface(CSI) 这样更完善、更编程友好的插件方式。

![存储插件管理原理](/png/storage.png)

在上述体系下，无论是FlexVolume，还是 Kubernetes 内置的其他存储插件，实际上单人的角色，仅仅是 Volume 管理中的 “Attach阶段” 和 “Mount阶段” 的具体执行者。而像 Dynamic Provisioning 这样的功能，就不是存储插件的责任，而是 Kubernetes 本身存储管理功能的一部分。

CSI 插件体系的设计思想，就是把这个 Provision 阶段，以及 Kubernetes 里的部分存储管理功能，从主干代码里剥离出来，做成几个单独的组件。

![csi](/png/csi.png)

这套存储插件体系多了三个独立的外部组件(External Componenets)，包括Driver Registrar、External Provisioner 及 External Attacher。而最右侧的部分则是需要我们编写代码来实现的 CSI 插件。通过 gRPC 的方式对外提供三个服务，分别是：CSI Identity、 CSI Controller 和 CSI Node。

对于 External Components，其中 Driver Register 组件，负责将插件注册到 kubelet 里(可以类比为，将可执行文件放在插件目录下)。 而在具体实现上， Driver Registrar 需要请求 CSI 插件的 Identity 服务来获取插件信息。

External Provisioner 组件，负责的正是 Provision 阶段。 在具体实现上，External Provisioner 监听(Watch) 了 API Server 里的 PVC 对象。当一个 PVC 被创建时，它就会调用 CSI Controller 的 CreateVolume 方法，为你创建对应的 PV。此外，如果你使用的存储是公有云提供的磁盘(或者块设备)的话，这一步就需要调用公有云(或者是块设备服务)的 API 来创建这个 PV 所描述的磁盘(或者块设备)。由于 CSI 插件是独立于 Kubernetes 之外的，所以在 CSI 的 API 里不会直接使用 Kubernetes 定义的 PV 类型，而是会自已定义一个单独的 Volume 类型。

最后一个 External Attacher 组件，负责的正是 “Attach 阶段”。在具体实现上，它监听了 API Server 里 VolumeAttachment 对象的变化。 VolumeAttachment 对象是 Kubernetes 确认一个 Volume 可以进入 “Attach 阶段” 的重要标志。一旦出现了 VolumeAttachment 对象，External Attacher 就会调用 CSI Controller 服务的 ControllerPublish 方法，完成它所对应的 Volume 的 Attach 阶段。

Volume 的“Mount 阶段”，并不属于 External Components 的职责。当 kubelet 的 VolumeManagerReconciler 控制循环检查到它需要执行 Mount 操作的时候，会通过 pkg/volume/csi 包，直接调用 CSI Node 服务完成 Volume 的“Mount 阶段”。

在实际使用 CSI 插件的时候，一般会将这三个 External Components 作为 sidecar 容器和 CSI 插件放置在同一个 Pod 中。主要是考虑到 External Components 对 CSI 插件的调用非常频繁，所以这种 sidecar 的部署方式非常高效。

接下来，则是 CSI 插件里的三个服务：Identity、Controller 和 Node。

其中，CSI 插件的 Identity 服务，负责对外暴露这个插件本身的信息：

```gRPC
service Identity {
    // return the version and name of the plugin
    rpc GetPluginInfo(GetPluginInfoRequest)
        returns (GetPluginInfoResponse) {}
    // reports whether the plugin has the ability of serving the Controller interface
    rpc GetPluginCapabilities(GetPluginCapabilitiesRequest)
        returns (GetPluginCapabilitiesResponse) {}
    // called by the CO just to check whether the plugin is running or not
    rpc Probe (ProbeRequest)
        returns (ProbeResponse) {}
}
```

CSI Controller 服务，定义的则是对 CSI Volume (对应 Kubernetes 里的 PV) 的管理接口，比如：创建和删除 CSI Volume、对 CSI Volume 进行 Attach/Dettach (在 CSI 里，这个操作被叫做 Publish/Unpublish)， 以及对 CSI Volume 进行 Snapshot 等：

```gRPC
service Controller {
    // provisions a volume
    rpc CreateVolume (CreateVolumeRequest)
        returns (CreateVolumeResponse) {}
    
    // deletes a previously provisioned volume
    rpc DeleteVolume (DeleteVolumeRequest)
        returns (DeleteVolumeResponse) {}

    // make a volume available on some required node
    rpc ControllerPublishVolume (ControllerPublishVolumeRequest)
        returns (ControllerPublishVolumeResponse) {}
    
    // make a volume un-available on some required node
    rpc ControllerUnpublish Volume (ControllerUnpublishVolumeRequest)
        returns (ControllerUnpublishVolumeResponse) {}
    
    ...

    // make a snapshot
    rpc CreateSnapshot (CreateSnapshotRequest)
        returns (CreateSnapshotResponse) {}

    // Delete a given snapshot
    rpc DeleteSnapshot (DeleteSnapshotRequest)
        returns (DeleteSnapshotResponse) {}

    ...
}
```

CSI Controller 服务里定义的这些操作有个共同特点，那就是它们无需在宿主机上进行，而是属于 Kubernetes 里 Volume Controller 的逻辑，也就是属于 Master 节点的一部分。

需要注意的是，CSI Controller 服务的实际调用者，并不是 Kubernetes (即： 通过pkg/volume/csi 发起 CSI 请求)， 而是 External Provisioner 和 External Attacher。这两个 External Components，分别通过监听 PVC 和 VolumeAttachment 对象，来跟 Kubernetes 进行协作。

CSI Volume 需要在宿主机上执行的操作，都定义在 CSI Node 服务里：

```protobuf
service Node {
    // temporarily mount the volume to a staging path
    rpc NodeStageVolume (NodeStageVolumeRequest)
        returns (NodeStageVolumeResponse) {}
    
    // unmount the volume from staging path
    rpc NodeUnstageVolume (NodeUnstageVolumeRequest)
        returns (NodeUnStageVolumeResponse) {}
    
    // unmount the volume from staging to target path
    rpc NodePublicVolume (NodePublishVolumeRequest)
        returns (NodePublishVolumeResponse) {}

    // umount the volume from staging path
    rpc NodeUnpublishVolume (NodeUnpublishVolumeRequest)
        returns (NodeUnpublishVolumeResponse) {}

    // stats for the volume
    rpc NodeGetVolumeStats (NodeGetVolumeStatsRequest)
        returns (NodeGetVolumeStatsResponse) {}
    
    ...

    // similar to NodeGetId
    rpc NodeGetInfo (NodeGetInfoRequest)
        returns (NodeGetInfoResponse) {}
}
```

需要注意的是， “Mount 阶段”在CSI Node里的接口，是由 NodeStageVolume 和 NodePublishVolume 两个接口共同实现的。

- 当 AttachDetachController 需要进行 “Attach” 操作时，它实际上会执行到 pkg/volume/csi 目录中， 创建一个 VolumeAttachment 对象，从而触发 External Attacher 调用 CSI Controller 服务的 ControllerPublishVolume 方法。
- 当 VolumeManagerReconciler 需要进行 “Mount” 操作时，它实际上也会执行到 pkg/volume/csi 目录中，直接向 CSI Node 服务发起调用 NodePublishVolume 方法的请求。

---

1. Register 过程： CSI 插件应作为 daemonSet 部署到每个节点。然后插件 Container 挂载 hostpath 文件夹，把插件可执行文件放在其中， 并启动 rpc 服务 (identity, controller, node)。 External component Driver Registrar 利用 kubelet plugin watcher 特性 watch 指定底文件夹路径来自动检测到这个存储插件。然后通过调用 identity rpc 服务，获取 driver 的信息，并完成注册。

2. Provision 过程： 部署 External Provisioner。Provisioner 将会 watch API Server 中 PVC 资源的创建， 并且 PVC 所指定的 storageClass 的 provisioner 是上面启动的插件。那么 External Provisioner 将会调用插件的 controller.createVolume() 接口。主要工作是创建网络磁盘，并根据磁盘的信息创建相应的 PV。

3. Attach 过程：部署 External Attacher。 Attacher 将会监听 API Server 中 VolumeAttachment 对象的变化。一旦出现新的 VolumeAttachment， Attacher 会调用插件的 controllerPublish() 接口。把相应的磁盘 attach 到声明使用此 PVC/PV 的 pod 所调度到的 node 上。 挂载的目录: /var/lib/kubelet/pods/<Pod ID>/volumes/xxx~netdisk/<name>

4. Mount 过程：mount 不可能在远程的 container 里完成，所以这个工作需要 kubelet 来做。kubelet 的 VolumeManagerReconciler 控制循环，检测到需要执行 Mount 操作的时候，通过调用 pkg/volume/csi 包，调用 CSI Node 服务，完成 volume 的 Mount 阶段。具体工作是调用 CRI 启动带有 volume 参数的 container，把上阶段准备好的磁盘 mount 到 container 指定的目录。
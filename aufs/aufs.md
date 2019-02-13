# AUFS #

-> Docker在启动容器的时候，需要创建文件系统，为rootfs提供挂载点。典型的Linux文件系统有bootfs和rootfs 2部分主城，bootfs主要包含bootloader和kernel，bootloader主要是引导加载kernel，当kernel被加载到内存中后bootfs就被umount了。rootfs包含的就失典型Linux系统中的/dev、/proc、/bin、/etc等标准目录和文件。

-> 在Docker的体系里把union mount的这些read-only的rootfs叫做Docker的镜像。但是，此时的每一层rootfs都是read-only的，我们此时还不能对其进行操作。当我们创建一个容器，也就是将Docker镜像进行实例化，系统会在一层或是多层read-only的rootfs之上分配一层空的read-write的rootfs

-> AUFS是一种Union File 	System，所谓UnionFS就是把不同物理位置的目录合并到mount到同一个目录中。UnionFS的一个最重要的应用就是 ，把一张CD/DVD和一个硬盘目录给联合mount在一起，然后，你就可以对这个只读的CD/DVD上的文件进行修改（当然修改的文件存在硬盘上的目录）

-> 在mount aufs命令中，目录默认权限是：命令行第一个（最左边）的目录是可新建可读可写的，后面的全部是只读的。如果有重复的文件名，在mount命令行上，越往前的优先级就越高。


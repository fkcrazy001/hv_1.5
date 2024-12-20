# x86 type1.5启动流程分析

## jailhouse boot type 1.5

流程入口是在jailhouse, 加载改动过的jailhouse内核模块，然后enable。
```bash
JH_DIR=~/jailhouse-arceos
JH=$JH_DIR/tools/jailhouse

sudo $JH disable
sudo rmmod jailhouse
sudo insmod $JH_DIR/driver/jailhouse.ko
sudo chown $(whoami) /dev/jailhouse
echo "enable jailhouse"
sudo $JH enable $JH_DIR/configs/x86/qemu-arceos.cell
```

### jailhouse enable arceos_type1.5

- tools侧，通过ioctl命令操作jailhouse device, 使能 type1.5
  - config parse
  - enable
    ```c
	config = read_file(argv[2], NULL); // read config file
    fd = open(JAILHOUSE_DEVICE, O_RDWR); // open JAILHOUSE_DEVICE
	err = ioctl(fd, JAILHOUSE_ENABLE, config);
    ```
- kmod：入口：jailhouse_cmd_enable
  - 通过config初始化各种内存 @todo
  - enter hv : on_each_cpu(enter_hypervisor, header, 0);
  - enter_hypervisor: 进入 arceos.bin 所定义的入口，就是 x86下面的_start

### type1.5 arceos 启动流程

- linux ctx save
  - linux sp 保存，存在函数调用规范的寄存器以及 gs(干嘛用？)
  - main core 初始化设备以及内存  primary_init_early

- cpu enable hv
  - activate_hv_pt： 初始化 hv 的pt： @todo
  - axhal::platform_init(); 初始化别的设备

- vcpu, vm 初始化， 返回 guest linux
  - new_vcpu， bind_vcpu， vcpu与物理cpu一一对应
  - run_type15_vcpu：hypercraft 中代码，处理vm exit 信息


### type1.5 arceos 响应linux vmmcall (start new kernel 等)

- vmcall 0x101： create config
- vmcall 0x102:  load image
- vmcall 0x103:  boot os
- nmi
  - 通过nmi进行消息的传递，传递的是上面的 vmcall 信息。
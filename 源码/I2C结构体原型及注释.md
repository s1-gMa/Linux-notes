---
tags:
  - I2C
---
# **i2c_adapter**
```C
/* 表示一条 I2C 总线 / 控制器实例（/dev/i2c-X 背后对应的“总线对象”） */
struct i2c_adapter {
    struct module *owner;             // 拥有此 adapter 的内核模块，一般为 THIS_MODULE，防止模块被卸载时仍在使用

    unsigned int class;               // 支持探测的设备“类别”掩码，用于自动探测设备时过滤

    const struct i2c_algorithm *algo; // 该总线使用的 I2C 算法，里面是 master_xfer/smbus_xfer 等回调
    void *algo_data;                  // 传给 algo 的私有数据，适配器驱动自用

    /* data fields that are valid for all devices   */
    const struct i2c_lock_operations *lock_ops; // 自定义锁操作（可选），覆盖默认 bus_lock 行为
    struct rt_mutex bus_lock;         // I2C 总线锁，保证同一条总线访问的互斥
    struct rt_mutex mux_lock;         // I2C 复用器锁，处理挂在 mux 后面的多路总线并发

    int timeout;                      // 传输超时时间（jiffies），避免某次传输无限阻塞
    int retries;                      // 传输失败时的重试次数

    struct device dev;                // 内嵌的通用 device，用于 sysfs、电源管理、驱动模型等
    unsigned long locked_flags;       // I2C core 内部使用的状态标志位

#define I2C_ALF_IS_SUSPENDED        0 // locked_flags 的 bit：当前适配器是否处于 suspend 状态
#define I2C_ALF_SUSPEND_REPORTED    1 // locked_flags 的 bit：挂起状态是否已经向上层报告

    int nr;                           // 适配器编号，如 0 → /dev/i2c-0
    char name[48];                    // 适配器名字，用于日志打印、/sys 里显示

    struct completion dev_released;   // 用来等待/通知 dev 被完全释放的同步原语

    struct mutex userspace_clients_lock; // 保护 userspace_clients 链表的互斥锁
    struct list_head userspace_clients;  // 由用户空间创建的“虚拟 client”(例如通过 i2c-dev) 列表

    struct i2c_bus_recovery_info *bus_recovery_info; // 总线恢复相关信息和回调（总线挂死时怎么 recovery）
    const struct i2c_adapter_quirks *quirks;         // 该适配器的一些特殊限制/quirks（如最大报文长度等）
    struct irq_domain *host_notify_domain;           // SMBus Host Notify 使用的中断域
    struct regulator *bus_regulator;                 // 为 I2C 总线/设备供电的 regulator 句柄
    struct dentry *debugfs;                          // adapter 在 debugfs 中的目录节点

    /* 7bit address space */
    DECLARE_BITMAP(addrs_in_instantiation, 1 << 7);  // 记录已经用于实例化 client 的 7bit 地址，防止重复创建
};
```

# **i2c_algorithm**
```C
/* 描述适配器如何在底层真正“发 I2C / SMBus 报文”的算法回调集合 */
struct i2c_algorithm {
    /*
     * 如果适配器算法不能做 I2C 级别访问，可以让 xfer 为 NULL；
     * 如果适配器算法能做 SMBus 访问，则实现 smbus_xfer；
     * 若 smbus_xfer 为 NULL，则用普通 I2C 报文模拟 SMBus 协议。
     */
    union {
        int (*xfer)(struct i2c_adapter *adap,
                    struct i2c_msg *msgs,
                    int num);
        int (*master_xfer)(struct i2c_adapter *adap,
                           struct i2c_msg *msgs,
                           int num);
    };
    // xfer/master_xfer：主模式下执行一次 I2C 传输，msgs 是报文数组，num 是报文数量

    union {
        int (*xfer_atomic)(struct i2c_adapter *adap,
                           struct i2c_msg *msgs, int num);
        int (*master_xfer_atomic)(struct i2c_adapter *adap,
                                  struct i2c_msg *msgs, int num);
    };
    // xfer_atomic/master_xfer_atomic：不能睡眠的上下文（原子上下文）里使用的传输版本

    int (*smbus_xfer)(struct i2c_adapter *adap, u16 addr,
                      unsigned short flags, char read_write,
                      u8 command, int size, union i2c_smbus_data *data);
    // smbus_xfer：按 SMBus 事务格式执行一次传输（包含命令码、事务大小类型等）

    int (*smbus_xfer_atomic)(struct i2c_adapter *adap, u16 addr,
                             unsigned short flags, char read_write,
                             u8 command, int size, union i2c_smbus_data *data);
    // smbus_xfer_atomic：不能睡眠环境下的 SMBus 版本

    /* To determine what the adapter supports */
    u32 (*functionality)(struct i2c_adapter *adap);
    // 返回该适配器支持哪些功能（I2C_FUNC_* 位图），上层据此决定可用 API

#if IS_ENABLED(CONFIG_I2C_SLAVE)
    union {
        int (*reg_target)(struct i2c_client *client);
        int (*reg_slave)(struct i2c_client *client);
    };
    // reg_target/reg_slave：注册一个作为“从机/target”角色的 client 到这个适配器上

    union {
        int (*unreg_target)(struct i2c_client *client);
        int (*unreg_slave)(struct i2c_client *client);
    };
    // unreg_target/unreg_slave：从适配器中注销这个从机/target client
#endif
};
```

# **i2c_client**
```C
/* 表示挂在某条 I2C 总线上的“一颗从设备”实例 */
struct i2c_client {

    unsigned short flags;        /* client 特性标志位 */

#define I2C_CLIENT_PEC      0x04   /* 启用 SMBus PEC（Packet Error Checking） */
#define I2C_CLIENT_TEN      0x10   /* 使用 10bit 地址，需配合 I2C_M_TEN */
                                   /* Must equal I2C_M_TEN below */
#define I2C_CLIENT_SLAVE    0x20   /* 本 client 工作在从机模式 */
#define I2C_CLIENT_HOST_NOTIFY  0x40 /* 启用 I2C/SMBus Host Notify 功能 */
#define I2C_CLIENT_WAKE     0x80   /* 能唤醒系统（wakeup capable），用于 board_info */
#define I2C_CLIENT_SCCB     0x9000 /* 使用 Omnivision SCCB 协议 */
                                   /* Must match I2C_M_STOP|IGNORE_NAK */

    unsigned short addr;         /* 设备 I2C 地址：7bit 地址存于低 7 位 */

    char name[I2C_NAME_SIZE];    /* 设备名（一般是芯片型号），用于匹配驱动、日志等 */

    struct i2c_adapter *adapter; /* 该 client 所在的适配器（哪一条 /dev/i2c-X） */

    struct device dev;           /* 通用 device 结构体，挂在设备模型上，用于 sysfs/PM 等 */

    int init_irq;                /* 初始化时获取到的 IRQ 号（来自 board_info/DT） */
    int irq;                     /* 实际使用的 IRQ 号（可能被重映射或修改） */

    struct list_head detected;   /* 用于“探测到的设备”链表（legacy 设备扫描场景） */

#if IS_ENABLED(CONFIG_I2C_SLAVE)
    i2c_slave_cb_t slave_cb;     /* 作为 I2C 从机时，接收主机访问的回调函数 */
#endif

    void *devres_group_id;       /* probe 期间 devres 分组 ID，便于出错时自动释放资源 */
    struct dentry *debugfs;      /* 为该 client 创建的 debugfs 目录，用于调试信息输出 */
};

```

* **addr 存放从机地址**
* **adapter指针指向从机对应的是哪条总线**

# **i2c_msg**
```C
/* 表示一次 I2C 传输里的一条“消息/报文段” */
struct i2c_msg {

    __u16 addr;                  // 目标从设备地址（7/10bit，取决于 flags）

    __u16 flags;                 // 控制此消息行为的标志位

#define I2C_M_RD         0x0001  /* 本消息是读操作（如果未置位则为写）——固定为 0x0001 */
#define I2C_M_TEN        0x0010  /* 使用 10bit 地址，需要适配器支持 I2C_FUNC_10BIT_ADDR */
#define I2C_M_DMA_SAFE   0x0200  /* 缓冲区可用于 DMA（仅在内核空间使用） */
#define I2C_M_RECV_LEN   0x0400  /* SMBus block read：第一字节为长度，会自动调整 len */
#define I2C_M_NO_RD_ACK  0x0800  /* 忽略读操作中的 ACK，用于协议“篡改”，需 FUNC_PROTOCOL_MANGLING */
#define I2C_M_IGNORE_NAK 0x1000  /* 忽略 NAK，仍继续传输，同样属于“协议篡改” */
#define I2C_M_REV_DIR_ADDR 0x2000 /* 反转 R/W 位方向（特殊协议需求） */
#define I2C_M_NOSTART    0x4000  /* 本消息不发送 START，和前一个消息黏在一起，需支持 NOSTART */
#define I2C_M_STOP       0x8000  /* 在本消息后强制发送 STOP，属于协议篡改 */

    __u16 len;                   // 本消息需要发送/接收的数据长度（字节数）
    __u8 *buf;                   // 数据缓冲区指针：写时为待发数据，读时为接收缓冲区
};

```
- **flags用于表示传输方向(读|写)**

# **i2c_driver**
```C
/* I2C 设备驱动（client driver）本体的描述结构 */
struct i2c_driver {
    unsigned int class;          // 支持的设备类别（与 i2c_adapter->class 配合使用，旧式自动探测）

    /* Standard driver model interfaces */
    int (*probe)(struct i2c_client *client);
    // 当有 i2c_client 与本驱动匹配时，由 I2C core 调用，完成设备初始化

    void (*remove)(struct i2c_client *client);
    // 对应 client 被移除（或驱动卸载）时调用，做资源释放、注销子设备等

    /* driver model interfaces that don't relate to enumeration  */
    void (*shutdown)(struct i2c_client *client);
    // 系统关机/重启前调用的钩子，用于把设备置于安全状态

    /*
     * Alert 回调，例如 SMBus Alert 协议：
     * 具体 data 的格式由协议决定：
     * - SMBus Alert：低位 bit 通常表示“事件标志”
     * - SMBus Host Notify：data 对应从设备报告的 16bit 负载数据
     */
    void (*alert)(struct i2c_client *client,
                  enum i2c_alert_protocol protocol,
                  unsigned int data);

    /*
     * 类似 ioctl 的回调，可实现对设备的专用命令操作
     * 调用方通常是内核内部的其他子系统或通过中间层转发
     */
    int (*command)(struct i2c_client *client,
                   unsigned int cmd, void *arg);

    struct device_driver driver;
    // 内嵌的通用 device_driver 结构（包含 .name、.owner、OF/ACPI 匹配表等）

    const struct i2c_device_id *id_table;
    // 传统 ID 匹配表：用于非 DT/ACPI 场景下根据 name 匹配 driver 和 client

    /* Device detection callback for automatic device creation */
    int (*detect)(struct i2c_client *client,
                  struct i2c_board_info *info);
    // 旧式自动探测回调：扫描某些地址时，由驱动检查设备是否存在并填写 board_info

    const unsigned short *address_list;
    // 旧式自动探测时要扫描的地址列表，以 0 结束

    struct list_head clients;
    // 挂在此驱动下的所有 i2c_client 链表，用于管理已经绑定的设备

    u32 flags;
    // 驱动的一些特性标志（例如支持的特殊行为），由内核或驱动自己定义使用
};

```

# **i2c_board_info**
```C
/* 板级信息：用来描述“板子上有哪些 I2C 设备”并用于创建 i2c_client */
struct i2c_board_info {
    char        type[I2C_NAME_SIZE];
    // 设备类型字符串，要与 i2c_driver 的 id_table / of_match 中 name 对应

    unsigned short flags;
    // client flags 的初始值（如 I2C_CLIENT_TEN、I2C_CLIENT_PEC 等）

    unsigned short addr;
    // I2C 地址（通常为 7bit，放在低 7 位）

    const char  *dev_name;
    // 可选的 dev_name，用于生成 /dev 节点等场景（若驱动需要）

    void        *platform_data;
    // 传统板级私有数据指针，传给驱动使用（非 DT/ACPI 时代常用）

    struct fwnode_handle *fwnode;
    // 固件节点句柄：指向 Device Tree node / ACPI node 等抽象固件描述节点

    const struct software_node *swnode;
    // software node：软件虚拟出来的“固件节点”，用来模拟 DT/ACPI 的属性

    const struct resource *resources;
    // 资源数组（寄存器地址范围、中断等），部分驱动可能用到

    unsigned int    num_resources;
    // 上面 resources 的数量

    int        irq;
    // 设备的中断号（如果有），也可以从 resources 中获取，这是一个快捷字段
};

```
``




``








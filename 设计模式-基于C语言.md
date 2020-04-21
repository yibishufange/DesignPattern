#### 设计模式分类：
* 创建型：创建对象实例的方法；
* 结构型：处理对象之间关系的方法；
* 行为型：处理对象之间互动的方法；

----
#### 创建型：工厂方法模式（Factory Pattern）
* OO描述：定义一个用于创建对象的接口，让子类决定实例化哪一个类；工厂方法使一个类的实例化延迟到其子类。
* C实践：把动态内存分配的工作交个一个“工厂”来做，我们只要告诉工厂需要什么规格的产品就可以；
* 用途：对客户隐藏内存分配的细节和复杂的创建过程，是内部算法的剧烈变动不会影响到调用者的代码；
>案例1：
> 命令行操作中要为待输出的字符串申请内存，常见代码是这样写的：
```c
/* 申请待输出字符串所使用的内存 */
CHAR *szOutString;
szOutString = VOS_Malloc(PID_FOO, FOO_MAX_MLINFO_SIZE * sizeof(CHAR));
if (NULL_PTR == szOutString)
{
    /* szx-pending */
    VOS_RECORD_ERROR(ERR_FOO_NULLPTR);
    return ERR_FOO_NULLPTR;
}
VOS_MemSet(szOutString, 0, FOO_MAX_MLINFO_SIZE * sizeof(CHAR));
```
> 这段代码有几个问题，
> 1）重复度很高，调用者需要记住FOO_MAX_MLINFO_SIZE这样的细节，如果在不同的任务中使用，或者要修改申请的大小，还得去查任务ID和宏的名称，容易出错；
> 2）VOS_Malloc/VOS_MemSet为同一个目标服务，不应该直接暴露给用户；
> 为此，我们可以建立一个专门生产这类产品的工厂：
```c
/*
  * 在模块内定义分配字符串的接口。细节被封装起来，这样今后不但
  * 使用方便，也不容易出错了。显然，记忆FOO_String_Max_Alloc()这
  * 样的名字比记一大堆的宏容易多了。
  */
CHAR * FOO_String_Max_Alloc()
{
    CHAR * TempStr = VOS_Malloc(PID_FOO, FOO_MAX_MLINFO_SIZE *    
                                sizeof(CHAR));
    if (NULL_PTR == TempStr)
    {
        /* szx-pending */
        VOS_RECORD_ERROR(ERR_FOO_NULLPTR);
        return NULL_PTR;
    }
    VOS_MemSet(TempStr, 0, FOO_MAX_MLINFO_SIZE  * sizeof(CHAR));
    return TempStr;
}
/*
  * 与之对应的是，同时应该定义释放字符串对象
  * 的接口。
  */
VOID FOO_String_Free(CHAR *String)
{
    VOS_Free(String);
}
```
> 有了这2个接口，我们就可以从这个工厂里简单的得到一个字符串或者销毁字符串。
> 我们的系统越来越复杂，不仅FOO模块需要输出字符串了，ABC模块也来凑热闹，于是，系统中出现了ABC_String_Max_Alloc()。如此多的小作坊重复着类似的工作，是一种浪费，我们可以将小工厂合并成一个大工厂了。
```c
/*
  * 我们在系统中建造了一间大工厂，可以按需生成
  * 各种各样的String。
  */
CHAR * String_Alloc_Max_for(enum Module mod)
{
    CHAR * TempStr;
    switch (mod) {
        case MOD_FOO:
                TempStr = FOO_String_Alloc();
                break;
        case MOD_ABC:
                TempStr = ABC_String_Alloc();
                break;
        default:
                TempStr = NULL;
    }
    return TempStr;
}
```
* 小节：工厂方法就是在代码中建立这样一个方法集：它能为我们按需分配内存，用户无需为得到这些内存记忆具体的细节，工厂生产方法的变化对上层用户透明。
#### 创建型：原型模式（Prototype Pattern）
* OO描述：用原型实例指定创建对象的种类，并且通过拷贝这些原型创建新的对象。
* 工厂模式是一种从无到有创建产品的方法。但有的时候，依照已经存在的产品实体来创建更加方便，这就是原型模式。
> 案例1：对于macvlan协议，经常要向vlan内的端口广播一个报文，重新创建报文开销太大，因此linux kernel的网络子系统是这样处理的。
```c
static void macvlan_broadcast(struct sk_buff *skb, 
                              const struct macvlan_port *port,
                              struct net_device *src,
                              enum macvlan_mode mode)
{
    const struct ethhdr *eth = eth_hdr(skb);
    const struct macvlan_dev *vlan;
    struct hlist_node *n;
    struct sk_buff *nskb;
    unsigned int i;
    int err;

    if (skb->protocol == htons(ETH_P_PAUSE))
        return;

    for (i = 0; i < MACVLAN_HASH_SIZE; i++) {
        hlist_for_each_entry_rcu(vlan, n, &port->vlan_hash[i], hlist) {
            if (vlan->dev == src || !(vlan->mode & mode))
                continue;

            nskb = skb_clone(skb, GFP_ATOMIC);  /* 复制一份报文 */
            err = macvlan_broadcast_one(nskb, vlan, eth,
                                        mode == MACVLAN_MODE_BRIDGE);
            macvlan_count_rx(vlan, skb->len + ETH_HLEN,
                             err == NET_RX_SUCCESS, 1);
        }
    }
}
```
> 案例2：在VOIP_SendSctpData()中，调用VOS_CopySocket()复制socket，避免了再次创建。
```c
{
    /* 拷贝Socket */
    vsSctpSocket = VSP_CopySocket(g_ulSctpTaskID, 
                                  ulCurrentTaskID,                          
                                  g_vsSctpRawSocket,        
                                  VOS_NULL);
    if ((VOS_NULL_LONG == vsSctpSocket) || 
            (VOS_NULL == vsSctpSocket))
    {
        /* 使用VPRODUCT的打印函数，与接收打印保持一致 */
        VPRODUCT_DotPrintf(IAS_VPRODUCT_MAJ_PLT_BUTT, 
                           VPRODUCT_SOCKET_INFO,
                           "VOIP_SendSctpData::拷贝Socket失败!");
        VOS_RECORD_ERROR(vsSctpSocket);
        return VOS_ERR;
    }
    ucIfCloseSock = TRUE;
}
```
* 小节：原型模式看起来也是一种工厂，但它是仿造而不是原创；当克隆一个对象比重新创建更方便的时候，使用原型模式。

----
#### 结构型：桥接模式（Bridge）
* OO描述：将抽象部分与它的实现部分分离，使它们都可以独立地变化。
> 案例：以linux kernel的网络设备模型作为考察对象。
> 如果我们把网络设备视为对象的话，很自然地，设备的操作方法应该是设备对象方法的一部分。但不同类型的网络设备的操作方法集不同，因此它是代码中剧烈变动的部分。同理，相同类型的网络设备对象的操作方法集相同，应该让它被共享。

```c
struct net_device {
        ...;
        /* Management operations */
        const struct net_device_ops  *netdev_ops;
        const struct ethtool_ops  *ethtool_ops;
        ...;
}
/* 这是firewire网络设备的操作方法实例 */
static const struct net_device_ops fwnet_netdev_ops = {
	.ndo_open        = fwnet_open,
	.ndo_stop        = fwnet_stop,
	.ndo_start_xmit  = fwnet_tx,
	.ndo_change_mtu  = fwnet_change_mtu,
};

/* 这是HDLC网络设备的操作方法实例 */
static const struct net_device_ops hdlcdev_ops = {
	.ndo_open       = hdlcdev_open,
	.ndo_stop       = hdlcdev_close,
	.ndo_change_mtu = hdlc_change_mtu,
	.ndo_start_xmit = hdlc_start_xmit,
	.ndo_do_ioctl   = hdlcdev_ioctl,
	.ndo_tx_timeout = hdlcdev_tx_timeout,
};
/*
  * 使用了Bridge模式之后，对象的操作和对象的抽象分离，
  * 避免了耦合，隔离了不同设备的动作差异，也提高了重用度。
  * 打开和关闭设备的代码可以这样写，并且不受具体打开和
  * 关闭动作细节的影响。
  */
if (netif_running(netdev))
        netdev->netdev_ops->ndo_open(netdev);

if (netif_running(netdev))
        netdev->netdev_ops->ndo_stop(netdev);
```
#### 结构型：外观模式（Facade Pattern）
* OO描述：为子系统中的一组接口提供一个一致的界面。Facade模式定义了一个高层接口，这个接口使得子系统更加容易使用。
* 只有当一个复杂的子系统有多个剧烈变化的接口时，才考虑用Facade模式来重构。
> 案例：
> 报文需要分发给不同的任务处理，如果每个任务都有自己的收包接口，那么驱动代码就会和这些任务耦合在一起。利用Façade模式，我们可以对外只提供一个总的收包接口，具体的处理细节被封装起来。

#### 结构型：代理模式（Proxy Pattern）
* OO描述：为其他对象提供一种代理以控制对这个对象的访问。
* 代理模式就是在用户和对象之间增加访问控制接口，形成“访问层”，方便控制对原始对象的访问。基于接口，而不是基于实现。这样做有几个好处：
    - 用户代码和对象代码通过中间层解耦；
    - 访问方式标准化，减少出错的机会；
    - 便于控制不同用户的访问能力；
> 案例：我们在系统中大量使用的hook其实就是Proxy模式的一种表现形式。通过hook，平台和产品代码解耦。

----
#### 行为型：状态模式（State Pattern）
* OO描述：允许一个对象在其内部状态改变时改变它的行为。对象看起来似乎修改了它的类。
> 案例：ip协议栈在解析完报文之后，需要决定报文的去向。可能传递到L4，也可能转发，或者干脆丢弃。
> 一种实现方法是用分支来控制流程：
```c
if (Packet) {
    switch (Packet->dst) {
        case PKT_DST_LOCAL:
             return local_rcv(Packet);
        case PKT_DST_OTHER:
             return forward_packet(Packet);
        /* 缺省也是丢弃 */
        case PKT_DST_BLACKHOLL:
             return packet_blackholl(Packet);
        default:
             return RES_DROP;
    }
}
```
> 这样的实现将导致调用者感知到报文去向的细节，不利于解耦。因此，可以用State模式在查路由表的时候就动态改变Packet的“投递”行为。
```c
void fib_lookup(struct sk_buff *Packet)
{
        if (__fib_lookup(LOCAL_TABLE, Packet) == RES_SUCC) {
            Packet->dst = PKT_DST_LOCAL;
            Packet->deliver = &skb_deliver_hook;
            return;
        }
        if (__fib_lookup(MAIN_TABLE, Packet) == RES_SUCC) {
            Packet->dst = PKT_DST_OTHER;
            Packet->deliver = &skb_deliver_hook;
            return;
        }
        if (blackholl_need(Packet)) {
            Packet->dst = PKT_DST_BLACKHOLL;
            Packet->deliver = &blackholl_consume_hook;
            return;
        }
}
/* 转发流程看起来就简单多了 ：不同的Packet有不同的行为 */
fib_lookup(Packet);
return Packet->deliver(Packet);  /* 把对象自己传入，相当于C++的this指针 */
```

#### 行为型：责任链模式（Chain of Responsibility Pattern）
* OO描述：使多个对象都有机会处理请求，从而避免请求的发送者和接收者之间的耦合关系。将这些对象连成一条链，并沿着这条链传递该请求，直到有一个对象处理它为止。
* 这个模式和Observer模式的区别在于它有多个潜在的处理者，但最终只有一个处理者起作用。
> 案例：考虑iptables过滤机制。我们定义了两条过滤规则。加上默认规则，一共三条：
-P INPUT ACCEPT
-A INPUT -s 192.168.2.0/24 -j DROP 
-A INPUT -s 192.168.2.100/32 -j DROP
来自192.168.2.100的报文在第二条规则点就被处理（DROP）了，不会传递到第三条规则点。这就是此模式的一个应用。

#### 行为型：模板模式（Template Pattern）
* OO描述：定义一个操作中的算法的骨架，而将一些步骤延迟到子类中。Te m p l a t e  M e t h o d使得子类可以不改变一个算法的结构即可重定义该算法的某些特定步骤。
> 案例：某模块为其它应用模块定义了各个阶段下发的配置操作，而各个业务模块通过注册具体的钩子函数来实现不同阶段的操作。

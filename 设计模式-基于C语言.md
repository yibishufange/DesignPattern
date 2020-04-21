#### 设计模式分类：
* 创建型：创建对象实例的方法；
* 结构型：处理对象之间关系的方法；
* 行为型：处理对象之间互动的方法；

#### 创建型：工厂方法模式
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
#### 创建型：原型模式
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

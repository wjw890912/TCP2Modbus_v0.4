


QQ:925295580
e-mail:925295580@qq.com
Time:201512
author:王均伟



       SW

   beta---(V0.1)


1、TCP/IP基础协议栈
.支持UDP
.支持TCP
.支持ICMP

2、超轻量级efs文件系统
.支持unix标准文件API
.支持SDHC标准。兼容V1.0和V2.0大容量SDK卡-16G卡无压力。（驱动部分参考开源 :-)）
.超低内存占用，仅用一个512字节的高速缓存来对文集遍历。

3、支持1-write DS18B20 	温度传感器
.支持单总线严格时序
.支持ROM搜索，遍历树叶子，允许一条总线挂接多个温度传感器
.数据自动转换为HTML文件

4、TCP/IP应用
.支持TFTP服务端，可以完成文件的下载任务。（此部分来自GITHUB，增加部分TIMEOUT 事件）tftp -i
.支持NETBIOS服务。
.支持一个TCP服务器，本地端口8080.	测试服务
.支持一个TCP客户端，本地端口40096	远端端口2301 远端IP192.168.0.163	用来做数据交互
.支持一个UDP广播，  本地端口02	    广播地址255.255.255.255	  用来把温度采集的数据发广播
.支持一个HTTP服务器 本地端口80  	http:ip  访问之	  关联2只18B20温度传感器显示在简单的SD卡的网页上

5、系统编译后
Program Size: Code=51264 RO-data=28056 RW-data=1712 ZI-data=55048  

6、网络配置

ipaddr：192, 168, 0, 253-209
netmask：255, 255, 0, 0
gw：192, 168, 0, 20
macaddress：xxxxxxxxxx


tks for GitHUB

	    HW

 stm32f107+DP83848CVV+ds18B20*2+SDHC card (4GB)

7、修改了一个SDcrad的BUG，有一个判断语句写错了，fix后可以支持2GB和4GB以上的两种卡片了。20160122
8、增加了一个宏在main.h中
#define MYSELFBOARD
如果定义了，那么表示使用李伟给我的开发板
如果不定义就选择我自己画的那块板子。
没有什么本质区别，框架一样，只是IO口稍有改动。20160122


 20160123
9、增加一个SMTP应用，可以通过定义USED_SMTP来使能 。
10、完成smtp.c的移植和测试。可以向我的邮箱发送邮件。邮箱需要设置关闭SSL。并且需要修改一下源码 的几个宏定义。
11、把采集的温度数据20分钟发一封邮件到我自己的邮箱中。完成。  必须用通过认真的IP否则过不去防火墙

12、调整了一下发邮箱的时间为5分钟一封邮件，@163邮箱的限制大约是300封邮件


20160402

13、测试中发现有内存泄露的情况,通过增加内存的信息通过TCP输出的 2301端口debug后发现
  异常的如下
 总内存数（字节）：6144
 已用内存（字节）：5816
 剩余内存数（字节）：328
 使用标示：1
 正常的如下
 总内存数（字节）：6144
 已用内存（字节）：20
 剩余内存数（字节）：6124
 使用标示：1
 显然memalloc使内存溢出查找代码因为除了SMTP应用程序使用malloc外其他不具有使用的情况。
 所以肯定是SMTP出问题
 进一步分析代码为SMTP的smtp_send_mail（）中
 smtp_send_mail_alloced(struct smtp_session *s)
 函数使用的
 s = (struct smtp_session *)SMTP_STATE_MALLOC((mem_size_t)mem_len);
 分配了一块内存没有事正常的释放。
 这样反复
 几次最终导致这块应用代码不能正常返回一块完整的 mem_le大小的内存块而一直保留了328字节的剩余内存。
 这最终导致了所有依赖mem的应用程序全部获取不到足够的内存块。而出现的内存溢出。
 继续分析 释放的内存句柄  (struct smtp_session *) s
 发现几处问题

 1）非正常中止	“风险”
   if (smtp_verify(s->to, s->to_len, 0) != ERR_OK) {
    return ERR_ARG;
  }
  if (smtp_verify(s->from, s->from_len, 0) != ERR_OK) {
    return ERR_ARG;
  }
  if (smtp_verify(s->subject, s->subject_len, 0) != ERR_OK) {
    return ERR_ARG;
  }
  由于没有对  smtp_send_mail_alloced 函数进行判断所以如果此处返回会造成函数不能正常中止
  也就会导致 (struct smtp_session *) s	没有机会释放（因为在不正常中止时是在后面处理的）
  但是考虑到源数据是固定的从片上flash中取得的，这种几率几乎没有。但是存在风险。所以统一改为
  if (smtp_verify(s->to, s->to_len, 0) != ERR_OK) {
    	 err = ERR_ARG;
     goto leave;
  }
  if (smtp_verify(s->from, s->from_len, 0) != ERR_OK) {
    	 err = ERR_ARG;
     goto leave;
  }
  if (smtp_verify(s->subject, s->subject_len, 0) != ERR_OK) {
    	 err = ERR_ARG;
     goto leave;
  }

  2）、非正常TCP连接，主要原因。
  原来的函数为：
  if(tcp_bind(pcb, IP_ADDR_ANY, SMTP_PORT)!=ERR_OK)
	{
	return	ERR_USEl;
  	   
	}
  显然还是同样的会造成malloc 分配了但是没有被调用，修改为
  if(tcp_bind(pcb, IP_ADDR_ANY,SMTP_PORT)!=ERR_OK)
  {
	err = ERR_USE;
    goto leave;		   
  }

   这样	  leave中就会自动处理释放掉这个非正常中止的而造成的内存的溢出问题。
   leave:
  smtp_free_struct(s);
  return err;

 归根结底是一个问题。那就是必须保证malloc 和free 成对出现。



 14、NETBIOS 名字服务增加在lwipopts.h中增加
 #define NETBIOS_LWIP_NAME "WJW-BOARD"
 正确的名称
 这样可以使用如下格式找到板子的IP地址
 ping 	wjw-board 
 而不用指定IP地址
 20160410



 /*测试中发现长时间运行后SMTP还有停止不发的情况，内存的问题上面已经解决，下面尝试修改进行解决，并继续测试-见17条*/
  20160427

 15、修改SNMTP的timout超时时间统一为2分钟，因为我的邮件重发时间为3分钟。默认的10分钟太长。先修改之。不是他影响的。fix

 /** TCP poll timeout while sending message body, reset after every
 * successful write. 3 minutes def:(3 * 60 * SMTP_POLL_INTERVAL / 2)*/
#define SMTP_TIMEOUT_DATABLOCK  ( 1 * 60 * SMTP_POLL_INTERVAL / 2)
/** TCP poll timeout while waiting for confirmation after sending the body.
 * 10 minutes def:(10 * 60 * SMTP_POLL_INTERVAL / 2)*/
#define SMTP_TIMEOUT_DATATERM   (1 * 60 * SMTP_POLL_INTERVAL / 2)
/** TCP poll timeout while not sending the body.
 * This is somewhat lower than the RFC states (5 minutes for initial, MAIL
 * and RCPT) but still OK for us here.
 * 2 minutes def:( 2 * 60 * SMTP_POLL_INTERVAL / 2)*/
#define SMTP_TIMEOUT            ( 1 * 60 * SMTP_POLL_INTERVAL / 2)
 20160427
 
  16、增加监控SMTP TCP 部分的变量数组
   smtp_Tcp_count[0]//tcp new count 
    smtp_Tcp_count[1]//bind count
	 smtp_Tcp_count[2]connect count 
	 smtp_Tcp_count[3]bind fail save the all pcb list index number
	 that all use debug long time running on smtp .
 20160427

17、发现不是SMTP的问题似乎邮箱出问题了，重新修改以上15条参数全部为2分钟 ，30*4*0.5S=1MIN *2=2min 

/** TCP poll interval. Unit is 0.5 sec. */
#define SMTP_POLL_INTERVAL      4
/** TCP poll timeout while sending message body, reset after every
 * successful write. 3 minutes def:(3 * 60 * SMTP_POLL_INTERVAL / 2)*/
#define SMTP_TIMEOUT_DATABLOCK  30*2
/** TCP poll timeout while waiting for confirmation after sending the body.
 * 10 minutes def:(10 * 60 * SMTP_POLL_INTERVAL / 2)*/
#define SMTP_TIMEOUT_DATATERM   30*2
/** TCP poll timeout while not sending the body.
 * This is somewhat lower than the RFC states (5 minutes for initial, MAIL
 * and RCPT) but still OK for us here.
 * 2 minutes def:( 2 * 60 * SMTP_POLL_INTERVAL / 2)*/
#define SMTP_TIMEOUT
20160429            30*2
18、加长了KEEPALIVBE时间为
	pcb->so_options |= SOF_KEEPALIVE;
   pcb->keep_idle = 1500+150;// ms
    pcb->keep_intvl = 1500+150;// ms
   pcb->keep_cnt = 2;// report error after 2 KA without response
 20160429


19、增加几个SMTP结果的变量	smtp_Tcp_count[10]	upsize 10 dword

20、增加监控SMTP TCP 部分的变量数组
   smtp_Tcp_count[0]//tcp new count 
    smtp_Tcp_count[1]//bind count
	 smtp_Tcp_count[2]connect count 
	 smtp_Tcp_count[3]bind fail save the all pcb list index number
add smtp send result 
              smtp_Tcp_count[4]|= (smtp_result);  
			  smtp_Tcp_count[4]|= (srv_err<<8);
			  smtp_Tcp_count[4]|= (err<<24);

			  if(err==ERR_OK){smtp_Tcp_count[5]++;}	//smtp成功次数统计

			   if(arg!=(void*)0)
			   {
				smtp_Tcp_count[6]=0xAAAAAAAA ;	 //有参数
			   }
			   else
			   {
			   
			   smtp_Tcp_count[6]=0x55555555 ;	//没有参数
			   }
20160430
21、

 以上测试中发现运行到9天左右就会不再执行SMTP代码返回数据如下： 低字节在前--高字节在后
 【Receive from 192.168.0.253 : 40096】：
5D 11 00 00     5D 11 00 00    58 11 00 00    00 00 00 00     04 00 00 F6  48 11 00 00  55 55 55 55 

上面的数据可知：
tcp new count=0x115d
bind count=0x115d
connect count=0x1158
bind fail  number=0
smtp_result=  4（SMTP_RESULT_ERR_CLOSED）
srv_err=00
tcp err=0xf6是负数需要NOT +1=(-10) 错误代码为  ERR_ABRT	  Connection aborted

以上数据定格，不在变化，说明这个和TCP baind 没有关系，是TCP new和之前的调用问题，所以继续锁定这个问题找。

20160513

22、把速度调快，10秒钟一次SMTP 连接。修改SMTP 应用程序的超时时间为8秒钟同时增加
smtp_Tcp_count[8]++;来计数总的调用SMTP的次数
smtp_Tcp_count[7]++;来计数SMTP 用的TCP new之前的次数。排除一下TCP new的问题！
如果这个变量一直变化而后面的没有变化这证明TCP —new出错。反之再向前推，直到调用它的地方一点点排除。

继续追踪这个停止TCP 连接的问题。
20160513


 /********为了接口陈新的485而做*******************/

23、增加modbus RTU 主机部分底层TXRX的代码，打算使用RTU 和TCP 做成透传485.这边不处理，只转发。
定义了一个宏

//定义了则使用MODBUS RTU TX/RX底层收发 (注意应用层没有使用。因为应用层打算交给服务器做，这边仅仅做RTU透传)
#define USED_MODBUS

18:52调试TX通过。更换了TIM3和USART2的Remap

20160613

24、更新STM32的固件库，使用2011版本的，原因是原来的2009版本的CL系列的串口驱动有问题。波特率不正常。换为2011的正常了，
    MODBUS RTU的流程做了修改。发送屏蔽掉CRC校验的产生，。直接透传。
	注意是MODBUS 这个串口从软件上看也是半双工的	。
20160614

25、上面的24条问题最终结果是晶振问题导致的，和固件库没有关系。


#if !defined  HSE_VALUE
 #ifdef STM32F10X_CL   
  #define HSE_VALUE    ((uint32_t)8000000) /*!< Value of the External oscillator in Hz */
 #else 
  #define HSE_VALUE    ((uint32_t)8000000) /*!< Value of the External oscillator in Hz */
 #endif /* STM32F10X_CL */
#endif /* HSE_VALUE */


  这里	 #define HSE_VALUE    ((uint32_t)8000000) /*!< Value of the External oscillator in Hz */
  要定义为你自己的外部晶振值。
20160615

26、增加了服务器接口和陈新，这一版要准备用在大鹏数据采集上做网管，所以定了简答的协议，这边直观转发。

 IP4_ADDR(&ip_addr,114,215,155,179 );//陈新服务器

 /*服务器发下的协议类型 
协议帧格式
帧头+类型+数据域+\r\n
帧头：
一帧网关和服务器交互的开始同时还担负判读数据上传还是下发的任务。
【XFF+0X55】：表示数据上传到服务器
【0XFF+0XAA】: 表示是数据下发到网关
类型：
0x01:表示土壤温湿度传感器
0x02表示光照传感器
0x03 表示PH 值传感器
0x04 表示氧气含量传感器
数据域
不同的类型的传感器数据域，数据域就是厂家提供的MODBUS-RTU的协议直接搭载上。

 服务器发送：FF AA + 类型 +【modbus RTU 数据域 】+\r\n
 网关回复  ：FF 55 + 类型 +【modbus RTU 数据域 】+\r\n

*/

 27、根据服务器要求修改网关的上报数据

 getid 命令改为

 AB +00 +[三字节ID]+CD +'$'+'\r'+'\n'   9个字节

 修改上报数据位

 FF 55 + 类型（1字节）+ID（3字节）+[modbus-RTU数据域] + '$'+'\r'+'\n'

20180710

28、开启DHCP 
20160710

29、修改了服务器接口和主要是修改了IP地址其他不变。
 IP4_ADDR(&ip_addr,60,211,192,14);//济宁大鹏客户的IP服务器地址。
20160920


30、在stm32f10x.h中增加UID的地址映射，以便直接获取ID号

 typedef struct
 {
 	uint32_t UID0_31;
 	uint32_t UID32_63;
	uint32_t UID64_96;
 }ID_TypeDef;
#define UID_BASE              ((uint32_t)0x1FFFF7E8) /*uid 96 by wang jun wei */
#define UID    ((ID_TypeDef*) UID_BASE)
20160920
31、修改ID从CPU的UID中运算获得
  uint32_t id;

  UID_STM32[0]=0xAB;//mark

  UID_STM32[1]=0x00;//备用

  id=(UID->UID0_31+UID->UID32_63+UID->UID64_95);//sum ID
  
  UID_STM32[2]=id;
  UID_STM32[3]=id>>8;//
  UID_STM32[4]=id>>16; // build ID 	lost The Hight 8bit data save the low 3 byte data as it ID

  UID_STM32[5]=0xCD; //mark

  UID_STM32[6]=0x24;
  UID_STM32[7]=0x0d;  //efl
  UID_STM32[8]=0x0a;
 20160920
32、增加MAC地址从芯片ID获取。
 void GetMacAddr(void)
{

uint32_t cid;
cid=(UID->UID0_31+UID->UID32_63+UID->UID64_95);//sum ID
macaddress[0]=0x00;
macaddress[1]=0x26;
macaddress[2]=cid;
macaddress[3]=cid>>8;
macaddress[4]=cid>>16;
macaddress[5]=cid>>24;

}
20160921
33、修复MODBUS上报的错误ID

	 Mb2TcpBuff[0]=0xFF;
	 Mb2TcpBuff[1]=0x55;
	 Mb2TcpBuff[2]=Statues_MB;
	 Mb2TcpBuff[3]=UID_STM32[2];
	 Mb2TcpBuff[4]=UID_STM32[3];
	 Mb2TcpBuff[5]=UID_STM32[4];
	 memcpy(&Mb2TcpBuff[6],ucMBFrame,Mb2TcpLenth);//数据复制进TCP发送buf中
	 Mb2TcpBuff[Mb2TcpLenth+6]=0x24;
	 Mb2TcpBuff[Mb2TcpLenth+6+1]=0x0d;
	 Mb2TcpBuff[Mb2TcpLenth+6+2]=0x0a;	
20160921 


###########################################################################################

     新协议

##########################################################################################

34、在陈新新平台上修改通讯协议为

帧头/通信方向/应用类/网关ID/设备分类/设备类型/设备id/数据/帧尾/间隔符

帧头：2字节，FFAA
通信方向：1字节，01网关到服务器、02服务器到网关
子系统Id：业务子系统的ID号
网关ID：3个字节，设备配置产生
设备分类：1个字节 01网关、02设备、03直连 写死
设备类型：2个字节、设备配置产生
设备id：3个字节、设备配置产生，直连设备id用三个字节
数据：MODBUS-RTU、自定义
帧尾：2字节 AAFF
间隔符：3字节 240D0A



  0-1+FFAA (帧头)
  2+通信方向(01:网关到服务器02:服务器到网关)
  3+子系统Id：业务子系统的ID号
  456+网关ID(3个字节，设备配置产生)
  7+设备分类（1个字节 01网关、02设备、03直连 写死）
  89+设备类型（2个字节、设备配置产生）
  101112+设备id（3个字节、设备配置产生，直连设备id用三个字节）
  13-n+数据：MODBUS-RTU、自定义
  n+1+2+帧尾：2字节 AAFF
  n+3+4+5间隔符：3字节 240D0A

  网关ID为010203的服务器到网关协议为
  FF AA + 02 + 01 +  01  02 03 + 01 + 00 00 +  00   00   00 + MODBUS-RTU	 +	AA    FF +   24     0D     0A
  [0][1]  [2]  [3]	[4] [5] [6]	 [7]  [8][9]  [10] [11]	[12]  [13]~[n]			[n+1] [n+2]	 [n+3] [n+3] [n+4]







  网关ID为010203的网关到服务器协议为
  FF AA +   01 + 01  +   01  02 03    +   01    + 00 00     +  00   00   00     + MODBUS-RTU	 +	 AA    FF +   24     0D     0A
  [0][1]   [2]  [3]	    [4] [5] [6]	      [7]     [8][9]       [10] [11][12]      [13]~[n]			[n+1] [n+2]	 [n+3] [n+4] [n+5]

   FFAA       [0][1] 
   01	      [2]
   01 	      [3]
   01 02 03	  [4][5][6]       
   01         [7]
   00 00      [8][9]
   000000     [10][11][12] 
   MODBUS-RTU [13]~[n]
   AAFF 	  [n+1][n+2]
   240D0A	  [n+3][n+4][n+5]







 Mb2TcpLenth=usLength;//更新接受到的数据长度，准备启动TCP发送
 Mb2TcpBuff[0]=0xFF;
 Mb2TcpBuff[1]=0xAA;          //帧头

 Mb2TcpBuff[2]=0x01;         //上传数据到服务器标示

 Mb2TcpBuff[3]=0x01;         //子系统Id：业务子系统的ID号 ,z暂时未用，值默认为01即可


 Mb2TcpBuff[4]=UID_STM32[2];
 Mb2TcpBuff[5]=UID_STM32[3];
 Mb2TcpBuff[6]=UID_STM32[4]; //网关ID(3个字节，设备配置产生)

 Mb2TcpBuff[7]=0x01;         //设备分类（1个字节 01网关、02设备、03直连 写死）


 Mb2TcpBuff[8]=0x01;					  
 Mb2TcpBuff[9]=0x01;         //设备类型（2个字节、设备配置产生）


 Mb2TcpBuff[10]=UID_STM32[2];
 Mb2TcpBuff[11]=UID_STM32[3];
 Mb2TcpBuff[12]=UID_STM32[4]; //设备id(3个字节，设备配置产生),

memcpy(&Mb2TcpBuff[13],ucMBFrame,Mb2TcpLenth);//MODBUS-RTU数据复制进TCP发送buf中


 Mb2TcpBuff[Mb2TcpLenth+13+0]=0xAA;
 Mb2TcpBuff[Mb2TcpLenth+13+1]=0xFF;	 //帧尾

 Mb2TcpBuff[Mb2TcpLenth+13+2]=0x24;	
 Mb2TcpBuff[Mb2TcpLenth+13+3]=0x0D;
 Mb2TcpBuff[Mb2TcpLenth+13+4]=0x0A;	//间隔符	
  
 Mb2TcpLenth+=18;//计算帧总长度，最终作为发送标示发送到TCP buf


     	     	                        			 	 





  
帧头/通信方向/应用类/网关ID/设备分类/设备类型/设备id/数据/帧尾/间隔符

帧头：2字节，FFAA
通信方向：1字节，01网关到服务器、02服务器到网关
子系统Id：业务子系统的ID号
网关ID：3个字节，设备配置产生
设备分类：1个字节 01网关、02设备、03直连 写死
设备类型：2个字节、设备配置产生
设备id：3个字节、设备配置产生，直连设备id用三个字节
数据：MODBUS-RTU、自定义
帧尾：2字节 AAFF
间隔符：3字节 240D0A

直连设备协议中，网关id部分是000000

网关id 
FFAA(帧头2字节) 01(通信方向1字节) 01(子系统Id1字节) 000001(网关ID3字节) 01(设备分类1字节) 0001(设备类型2字节) 000000(设备id 3字节) 00(数据N字节) AAFF(帧尾2字节) 240D0A(间隔符3字节) 

FFAA 01 01 000002 01 0001 000000 00 AAFF 240D0A 

网关上报设备信息

FFAA 01 01 000001 02 0001 000001 FE030605DC09EC00C876F8 AAFF 240D0A
FFAA 01 01 000001 02 0001 000001 FE030605DC09EC00C876F8 AAFF 240D0A
FFAA 01 01 000001 02 0001 000002 FF030605DC09EC00C876F8 AAFF 240D0A
FFAA 01 01 000001 02 0001 000003 FE030605DC09EC00C876F8 AAFF 240D0A

直连
FFAA 01 01 000000 03 0001 000001 FE030605DC09EC00C876F8 AAFF 240D0A
FFAA 01 01 000000 03 0001 000002 FE030605DC09EC00C876F8 AAFF 240D0A
FFAA 01 01 000000 03 0001 000003 FE030605DC09EC00C876F8 AAFF 240D0A


getid
FF AA 01 01 9D A1 1A 01 00 01 9D A1 1A 00 AA FF 24 0D 0A


000000-Tx:01 03 00 00 00 0E C4 0E
000002-Tx:03 03 00 00 00 0E C5 EC


 FFAA 01 01 000001 02 0001 000003 01030000000EC40E AAFF 240D0A

网关上报数据：

FF AA 01 01 00 00 00 01 01 01 00 00 00 1C FC 00 00 00 00 E0 00 00 00 E0 00 AA FF 24 0D 0A



35、修改keepalive时间为1000MS 500ms间隔太小，掉线太频繁。	  20170914

36、修改上层统一下发数据报文截取长度为前个字节，上报不变，getid 不变

上层统一下发： FFAA 02 011000000001020000A650 AAFF 240D0A

解析这个。修改代码为：

	char *ptr=(char*)p->payload;
	uint16_t len;
	ptr+=3;	//jump FF AA and other there the ptr is point to MODBUS-RTU   
	if(p->tot_len<1500) //asseter the buffer lenth 
	{
	
	TcpRecvLenth=p->tot_len-5-3;//upadta the recive data lenth	 AA FF  24 0D 0A =5 BYTES
							   
	memcpy(TcpRecvBuf,ptr,TcpRecvLenth); // data copy to appliction data buffer 



 例子：
 TCP SEVER :FFAA 02 01030000000EC40E AAFF 240D0A
 REPO: FF AA 00 00 00 00 00 02 00 00 00 00 00 01 03 1C 00 01 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 01 00 01 00 01 00 01 1F 27 AA FF 24 0D 0A  

  20170915


 37、更新协议
 通用协议解析
  
-------------------网关、直连设备协议-------------------
帧头/通信方向/应用类/网关ID/设备分类/设备类型/设备id/数据/帧尾/间隔符
 
帧头：2字节，FFAA
通信方向：1字节，01网关到服务器、02服务器到网关
子系统Id：1字节，业务子系统的ID号
网关ID：3个字节，设备配置产生
设备分类：1个字节 01网关(网关上报id时用01)、02设备(网关上报设备数据用02)、03直连 写死
设备类型：2个字节、设备配置产生
设备id：3个字节、设备配置产生，直连设备id用三个字节
数据：MODBUS-RTU、自定义
帧尾：2字节 AAFF
间隔符：3字节 240D0A

直连设备协议中，网关id部分是000000
网关id 
FFAA 01 01 000001 01 0001 000000(设备id用六个零) 00（数据用00代替） AAFF 240D0A 
网关上报设备信息
FFAA 01 01 000001 02 0001 000001 FE030605DC09EC00C876F8 AAFF 240D0A
直连
FFAA 01 01 000000 03 0001 000001 FE030605DC09EC00C876F8 AAFF 240D0A

 
 -------------------客户端ID协议-------------------
帧头：2字节，FFAA
通信方向：1字节，01网关到服务器、02服务器到网关
子系统Id：1字节 业务子系统的ID号
设备类型：1字节 网关还是直连设备00,01
app是否初次连接服务：00初次连接app需要将app对应的用户账号对应的所有网关id一次发送过来，01发送信息
Client Id：3个字节,网页是五个字节
Client 类型：01 android、02 ios、03 web
网关or直连设备的：id，初次建立连接没有，发送消息时有
数据：MODBUS-RTU、自定义，初次建立连接为00
帧尾：2字节 AAFF
间隔符：3字节 240D0A
App Client ID
FFAA 02 01 00 00 000001 01 AAFF 240D0A 240D0A 
FFAA 02 01 00 00 000001 01 000001000002000003 AAFF 240D0A
FFAA 02 01 00 00 0000000001 03 000001 AAFF 网页初次连接不带分隔符
 
 -------------------客户端发软网关数据协议-------------------
帧头：2字节，FFAA
通信方向：1字节，01网关到服务器、02服务器到网关
子系统Id：1字节 业务子系统的ID号
设备类型：1字节 网关还是直连设备00,01
app是否初次连接服务：00初次连接app需要将app对应的用户账号对应的所有网关id一次发送过来，01发送信息
Client 类型：01 android、02 ios、03 web
Client Id：3个字节,网页是五个字节
网关or直连设备的：id，初次建立连接没有，发送消息时有
数据：MODBUS-RTU、自定义，初次建立连接为00
帧尾：2字节 AAFF
间隔符：3字节 240D0A
App Client Data
FFAA 02 01 00 01 01 000001  000001 FE030605DC09EC00C876F8 AAFF 240D0A
FFAA 02 01 00 01 03 0000000001  000001 FE030605DC09EC00C876F8 AAFF 240D0A网页初次连接不带分隔符
 
 -------------------客户端发网关数据协议-------------------
FFAA 02 011000000001020000A650 AAFF 240D0A

20170915


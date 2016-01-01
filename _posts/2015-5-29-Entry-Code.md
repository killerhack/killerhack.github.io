# 逆向--各类程序入口代码以及特征总结  
有点闲暇时间，总结一下，我个人认为逆向的识别壳以及脱壳的技术，都是靠看出来的，用一句数学用语来说就叫解无定法观察得之。下面就来简单说一下。  
##  各类无壳程序的特征  
我们知道不同的编译器将代码连接起来的特征是不一样的，各类语言的编译原理无非是将符号化的高层语言，比如C#，java,Delphi按照其语法定义连接成可以执行的EXE文件，但这中间就会丢失很多高层语言本身的符号，最终在EXE中只保留了必要的指令，而这些指令则就是汇编语言中最简单的操作。所以我们可以先不用管具体它是怎么实现的，让我们先看看各类编译器下入口代码的特征。  
###  1.VC6的结果  
VC6通常用来编译C或者C++的程序，用OD载入一个VC6程序后其入口代码如下。可以看到第三行压栈之后四五行有两个PUSH的值。
	
	00401700 >/$  55push :-(  
	00401701  |.  8BEC  mov :-(,:-)  
	00401703  |.  6A FF push -0x1    
	00401705  |.  68 00254000   push 吾爱破解.00402500  
	0040170A  |.  68 86184000   push <jmp.&MSVCRT._except_handler3>  ;  入口地址; SE handler installation
	0040170F  |.  64:A1 0000000>mov :-),dword ptr fs:[0]  
	00401715  |.  50push :-) ;  kernel32.BaseThreadInitThunk  
	00401716  |.  64:8925 00000>mov dword ptr fs:[0],:-)  
	0040171D  |.  83EC 68   sub :-),0x68  
	00401720  |.  53push :-(  
	00401721  |.  56push :-)  
	00401722  |.  57push :-(   
往下四五行以后就可以看到一段明显的api，这个就VC6的特征了。

	0040172B  |.  6A 02         push 0x2
	0040172D  |.  FF15 90214000 call dword ptr ds:[<&MSVCRT.__set_app_ty>;  msvcrt.__set_app_type
	00401733  |.  59            pop :-(                                  ;  kernel32.76E0337A
	00401734  |.  830D 4C314000>or dword ptr ds:[0x40314C],-0x1
	0040173B  |.  830D 50314000>or dword ptr ds:[0x403150],-0x1
	00401742  |.  FF15 8C214000 call dword ptr ds:[<&MSVCRT.__p__fmode>] ;  msvcrt.__p__fmode
	00401748  |.  8B0D 40314000 mov :-(,dword ptr ds:[0x403140]
	0040174E  |.  8908          mov dword ptr ds:[eax],:-(
	00401750  |.  FF15 88214000 call dword ptr ds:[<&MSVCRT.__p__commode>;  msvcrt.__p__commode
	00401756  |.  8B0D 3C314000 mov :-(,dword ptr ds:[0x40313C]
	0040175C  |.  8908          mov dword ptr ds:[eax],:-(
	0040175E  |.  A1 84214000   mov :-),dword ptr ds:[<&MSVCRT._adjust_f>
	00401763  |.  8B00          mov :-),dword ptr ds:[eax]
	00401765  |.  A3 48314000   mov dword ptr ds:[0x403148],:-)          ;  kernel32.BaseThreadInitThunk

这个set_app_type这些就是。当然也许有人会说这个不是直接拿PEID或者其他查壳工具一看就知道了么，没错，但是你的眼睛也得熟悉这些东西，否则你不知道原本的程序长什么样子的时候，遇到一些PEID等工具识别不出来的壳的时候你就抓瞎了，所以这个阶段就是要你熟悉这些代码的结构，就当是大脑神经网络模式训练吧哈哈。
**区段信息是固定的四个.text .rdata.data .rsrc**

### 2.VS2008&VS2013
都是微软家的编译器，就放在一起对比一下，基本上从代码入口的处理角度上来讲，2013和2008差别也不大，所以可以放在一起看，这里仅限的是2008或者2013的C/C++程序编译的结果，如果用的是C#或者VB.net的就又是另外一种样子了。

VS的入口特征很短      

	
	0025DDAC > $  E8 EF4E0000   call 吾爱破解.00262CA0
	0025DDB1   .^ E9 79FEFFFF   jmp 吾爱破解.0025DC2F

然后跟进第一个CALL之后就会看到一串系统调用

	00262CD6  |.  50            push :-)                                 ; /pFileTime = kernel32.BaseThreadInitThunk
	00262CD7  |.  FF15 0C212700 call dword ptr ds:[<&KERNEL32.GetSystemT>; \GetSystemTimeAsFileTime
	00262CDD  |.  8B75 FC       mov :-),dword ptr ss:[ebp-0x4]
	00262CE0  |.  3375 F8       xor :-),dword ptr ss:[ebp-0x8]           ;  kernel32.76E0337A
	00262CE3  |.  FF15 38222700 call dword ptr ds:[<&KERNEL32.GetCurrent>; [GetCurrentProcessId
	00262CE9  |.  33F0          xor :-),:-)                              ;  kernel32.BaseThreadInitThunk
	00262CEB  |.  FF15 68222700 call dword ptr ds:[<&KERNEL32.GetCurrent>; [GetCurrentThreadId
	00262CF1  |.  33F0          xor :-),:-)                              ;  kernel32.BaseThreadInitThunk
	00262CF3  |.  FF15 88212700 call dword ptr ds:[<&KERNEL32.GetTickCou>; [GetTickCount
	00262CF9  |.  33F0          xor :-),:-)                              ;  kernel32.BaseThreadInitThunk
	00262CFB  |.  8D45 F0       lea :-),dword ptr ss:[ebp-0x10]
	00262CFE  |.  50            push :-)                                 ; /pPerformanceCount = kernel32.BaseThreadInitThunk
	00262CFF  |.  FF15 08212700 call dword ptr ds:[<&KERNEL32.QueryPerfo>; \QueryPerformanceCounter

**在区段信息方面相对于VC6多了一个.reloc来保存更多的信息。**

###3.易语言
易语言虽然在正儿八经的软件工程里是很少见的，但是基于其可以少背单词的优点，也算是中国特色的一门编程语言了，很多Script-Kiddie非常偏爱，很多外挂以及小工具还真都是用它写的。易语言分为独立编译和非独立编译两种模式，**说白了就是把基础的运行库（后缀fne fnr）放在哪里的差别而已**。
>而且易语言基本上就是C++typedef来的，所以包括编译在内很多东西都和VC6很像，致使很多查壳软件也识别出来的是VC。区段也是一样的。  

1. 非独立编译模式  
非独立编译模式下，可以看到在代码入口的第一件事请就是加载这两个运行库，所以特色很鲜明。  

    
	    00401000 >/$  E8 89000000   call 吾爱破解.0040108E
	    00401005  |.  50push eax ; /ExitCode = 0x76E03368
	    00401006  \.  E8 B5010000   call <jmp.&KERNEL32.ExitProcess> ; \ExitProcess
	    0040100B   .  47 65 74 4E 6>ascii "GetNewSock",0
	    00401016   .  45 72 72 6F 7>ascii "Error",0
	    0040101C   .  6B 72 6E 6C 6>ascii "krnln.fne",0
	    00401026   .  4E 6F 74 20 6>ascii "Not found the ke"
	    00401036   .  72 6E 65 6C 2>ascii "rnel library or "
	    00401046   .  74 68 65 20 6>ascii "the kernel libra"
	    00401056   .  72 79 20 69 7>ascii "ry is invalid!",0
	    00401065   .  6B 72 6E 6C 6>ascii "krnln.fnr",0
	    0040106F   .  50 61 74 68 0>ascii "Path",0
	    00401074   .  53 6F 66 74 7>ascii "Software\FlySky\"
	    00401084   .  45 5C 49 6E 7>ascii "E\Install",0  
	    

或者在运行之后看OD模块窗口，也能看到运行库文件。  
2. 独立编译模式
独立编译模式可以通过**右键菜单中文搜索字符串**或者还是肉眼调，入口代码稍微往下走走就可以看到VC的初始化过程GetCommandLineA GetStartupInfo GetModuleHandleA，在GetMoudouleHandleA下第一个call就是main函数，当然了，C语言中可能是“主函数”哈哈.  
来看代码

	0046C68D  |.  FF15 5CC34800 call dword ptr ds:[<&KERNEL32.GetCommand>; [GetCommandLineA
	0046C693  |.  A3 242A4C00   mov dword ptr ds:[0x4C2A24],eax          ;  kernel32.BaseThreadInitThunk
	0046C698  |.  E8 88460000   call 吾爱破解.00470D25
	0046C69D  |.  A3 B8124C00   mov dword ptr ds:[0x4C12B8],eax          ;  kernel32.BaseThreadInitThunk
	0046C6A2  |.  E8 31440000   call 吾爱破解.00470AD8
	0046C6A7  |.  E8 73430000   call 吾爱破解.00470A1F
	0046C6AC  |.  E8 2A360000   call 吾爱破解.0046FCDB
	0046C6B1  |.  8975 D0       mov [local.12],esi
	0046C6B4  |.  8D45 A4       lea eax,[local.23]
	0046C6B7  |.  50            push eax                                 ; /pStartupinfo = kernel32.BaseThreadInitThunk
	0046C6B8  |.  FF15 E4C24800 call dword ptr ds:[<&KERNEL32.GetStartup>; \GetStartupInfoA
	0046C6BE  |.  E8 04430000   call 吾爱破解.004709C7
	0046C6C3  |.  8945 9C       mov [local.25],eax                       ;  kernel32.BaseThreadInitThunk
	0046C6C6  |.  F645 D0 01    test byte ptr ss:[ebp-0x30],0x1
	0046C6CA  |.  74 06         je short 吾爱破解.0046C6D2
	0046C6CC  |.  0FB745 D4     movzx eax,word ptr ss:[ebp-0x2C]
	0046C6D0  |.  EB 03         jmp short 吾爱破解.0046C6D5
	0046C6D2  |>  6A 0A         push 0xA
	0046C6D4  |.  58            pop eax                                  ;  kernel32.76E0337A
	0046C6D5  |>  50            push eax                                 ;  kernel32.BaseThreadInitThunk
	0046C6D6  |.  FF75 9C       push [local.25]
	0046C6D9  |.  56            push esi
	0046C6DA  |.  56            push esi                                 ; /pModule = NULL
	0046C6DB  |.  FF15 50C34800 call dword ptr ds:[<&KERNEL32.GetModuleH>; \GetModuleHandleA
	0046C6E1  |.  50            push eax                                 ;  kernel32.BaseThreadInitThunk
	0046C6E2  |.  E8 F7DD0000   call 吾爱破解.0047A4DE
	

然后跟进去这个```0046C6E2  |.  E8 F7DD0000   call 吾爱破解.0047A4DE```之后再跟进里层的第一个call，找到```00482B5B  |.  FF50 50       call dword ptr ds:[eax+0x50]```然后再此处F4执行到这一步，跟进去，然后再运行至pushad前一个call，跟进去就可以看到易语言的特征  

	0040D5D9  |.  FC            cld
	0040D5DA  |.  DBE3          finit
	0040D5DC  |.  E8 D8FFFFFF   call 吾爱破解.0040D5B9
	0040D5E1  |.  68 C4D54000   push 吾爱破解.0040D5C4
	0040D5E6  |.  B8 03000000   mov eax,0x3
	0040D5EB  |.  E8 33000000   call 吾爱破解.0040D623
	0040D5F0  |.  83C4 04       add esp,0x4
	0040D5F3  |.  E8 A53AFFFF   call 吾爱破解.0040109D
	0040D5F8  |.  E8 833AFFFF   call 吾爱破解.00401080
	0040D5FD  |.  E8 C73AFFFF   call 吾爱破解.004010C9
	0040D602  |.  68 01000152   push 0x52010001
	0040D607  |.  E8 11000000   call 吾爱破解.0040D61D
	0040D60C  |.  83C4 04       add esp,0x4
	0040D60F  |.  E8 03000000   call 吾爱破解.0040D617
	0040D614  |.  33C0          xor eax,eax                              ;  吾爱破解.00401002
	
任何一个call都会调用核心库的一些地方形如下面的调用。  

	00410828  - FF25 8F3A4000   jmp dword ptr ds:[0x403A8F]              ; krnln.1002D70F
	0041082E  - FF25 933A4000   jmp dword ptr ds:[0x403A93]              ; krnln.1002D672
	00410834  - FF25 973A4000   jmp dword ptr ds:[0x403A97]              ; krnln.1002D6A5
	0041083A  - FF25 9B3A4000   jmp dword ptr ds:[0x403A9B]              ; krnln.1002CE0A
	00410840  - FF25 8B3A4000   jmp dword ptr ds:[0x403A8B]              ; krnln.1002D80A
	00410846  - FF25 833A4000   jmp dword ptr ds:[0x403A83]              ; krnln.1002D72C
	0041084C  - FF25 6B3A4000   jmp dword ptr ds:[0x403A6B]              ; krnln.1002D60E
	00410852  - FF25 773A4000   jmp dword ptr ds:[0x403A77]              ; krnln.1002CE86
	00410858  - FF25 6F3A4000   jmp dword ptr ds:[0x403A6F]              ; krnln.1002CE24
	0041085E  - FF25 873A4000   jmp dword ptr ds:[0x403A87]              ; krnln.1002D75F


	

或者可以看看代码结构，有一定模式，都有些相同。  其中非独立模式编译还可以在.data下一个f2中断，等到然后执行后查看对战，就可以看到调用了运行库。点击第一条跳转过去，用f7跟进去果然还是上面的那堆调用。
 
	   
	地址       堆栈       函数过程 / 参数                       调用来自                      结构
	0018FE30   1002CD43   krnln.1005ED30                        krnln.1002CD3E                0018FE2C
	0018FE58   1002D84F   ? krnln.1002CCFF                      krnln.1002D84A                0018FE54   



当运行到这个地方的时候再去搜字符串就能搜到了。   

  
### 4.Delphi
第一个call跟进去一般都有这个GetMouldeHandle，此外区段中包含CODE，DATA，BSS这类的东西。

	00405BD2  |.  6A 00         push 0x0                                 ; /pModule = NULL
	00405BD4  |.  E8 2BFFFFFF   call <jmp.&kernel32.GetModuleHandleA>    ; \GetModuleHandleA
	00405BD9  |.  A3 64164500   mov dword ptr ds:[0x451664],eax          ;  kernel32.BaseThreadInitThunk
	00405BDE  |.  A1 64164500   mov eax,dword ptr ds:[0x451664]
	00405BE3  |.  A3 A8F04400   mov dword ptr ds:[0x44F0A8],eax          ;  kernel32.BaseThreadInitThunk
	00405BE8  |.  33C0          xor eax,eax                              ;  kernel32.BaseThreadInitThunk
	00405BEA  |.  A3 ACF04400   mov dword ptr ds:[0x44F0AC],eax          ;  kernel32.BaseThreadInitThunk
	00405BEF  |.  33C0          xor eax,eax                              ;  kernel32.BaseThreadInitThunk
	00405BF1  |.  A3 B0F04400   mov dword ptr ds:[0x44F0B0],eax          ;  kernel32.BaseThreadInitThunk

### 5.BC++
这个现在也很少见了随着borland公司把一手好牌打烂之后，Delphi和Borland C现在市场占有率都不大了，认识一下就可以了。
代码入口分析之后是一大段的JUMP

	004012FC > $ /EB 10         jmp short 吾爱破解.0040130E
	004012FE     |66            db 66                                    ;  CHAR 'f'
	004012FF     |62            db 62                                    ;  CHAR 'b'
	00401300     |3A            db 3A                                    ;  CHAR ':'
	00401301     |43            db 43                                    ;  CHAR 'C'
	00401302     |2B            db 2B                                    ;  CHAR '+'
	00401303     |2B            db 2B                                    ;  CHAR '+'
	00401304     |48            db 48                                    ;  CHAR 'H'
	00401305     |4F            db 4F                                    ;  CHAR 'O'
	00401306     |4F            db 4F                                    ;  CHAR 'O'
	00401307     |4B            db 4B                                    ;  CHAR 'K'
	00401308     |90            nop
	00401309     |E9            db E9

### 6. .Net
.Net是我目前最看好的Win平台下的语言，目前出场率也很高，不过对于它的逆向，因为机制中类似有一个虚拟机的东西，像JAVA一样，甚至可以逆向出源代码，不过我们还是从汇编的角度看看它怎么在OD中调试。
有时候拖进去可能会直接运行，这个有可能是中断在tls,可以使用TLLY插件去解决。从模块信息中就可以看到加载了很多.NET的库
### 7.AUTOIT
之前用这个做自动化非常棒，接触过不少，Autoit可以通过字符串直接发现，另外可以直接逆出源代码

### 8.其他类型etc ###
PB主要是加载模块中有libjcc.dll和pbvm90.dll，这个直接从动态链接库就能发现,Qt则是Qt5Core.dll Qt5Gui.dll Qt5Widgets.dll这类很明显的库就能看出来。

> 接下来就说一下各类壳的特征，知道没加壳的样子和入口特征之后，脱壳之后就可以看出来自己到底脱没脱干净。

## 加壳程序特征 ##

不要总是依赖于工具，多考虑手动的方式。自动化工具都是靠特征这类的方法去识别，但是像VMP这类的虚拟化壳，每次加完都不一样，所以就误报率很高，所以工具毕竟还是死的，要靠自己的经验来判断。

### 1.Aspack ###
入口点一上来是一个PUSHAD，就是所有寄存器入栈，保存下现场然后call,jmp，这个特征还可以看下区段。  

	00963001 >  60              pushad
	00963002    E8 03000000     call 吾爱破解.0096300A
	00963007  - E9 EB045D45     jmp 45F334F7
	0096300C    55              push ebp
	0096300D    C3              retn
	0096300E    E8 01000000     call 吾爱破解.00963014
	00963013    EB 5D           jmp short 吾爱破解.00963072
	00963015    BB EDFFFFFF     mov ebx,-0x13
	
区段可能为.aspack .adata多这两个区段，不过这两个名字是可变的，所以名字随便填的，不是特别可靠。
###  2.UPX  ###
UPX不同版本见是有点差距的，不过差别不大，区段信息特征非常明显，就是UPX0，UPX1,.RSRC一般都会压缩成三个区段，区段名称也可变，可能是假的。 
### 3.Themida ###
Themida也是主要看区段，目前入口加了一个随机的代码，执行之后解码之后才显示，一般都有五个区段，第一个名字是空的，最后两个是任意的，基本上就是Themida.版本不同稍微有区别。
### 4.Vmp ###
vmp的入口特征是特别乱的，基本啥也看不出来，基本上也是用区段看，一般都是程序原本的几个区段保留.reloc和.rsrc放在最下面，中间加三个区段，名字可变，如图。
![vmp区段](https://github.com/killerhack/killerhack.github.io/blob/master/_posts/vmp.png)  

	00D533CC    E8 F6EFFCFF     call 吾爱破解.00D223C7
	00D533D1    48              dec eax
	00D533D2    DE1F            ficomp word ptr ds:[edi]
	00D533D4    ff5e 43         call far fword ptr ds:[esi+0x43]
	00D533D7    29F0            sub eax,esi
	00D533D9    3950 3F         cmp dword ptr ds:[eax+0x3F],edx          ; ntdll_12.777C2080
	00D533DC    16              push ss
	00D533DD    17              pop ss
	00D533DE    F8              clc
	00D533DF    F7E8            imul eax
	00D533E1    D5 E6           aad 0xE6
	00D533E3    C9              leave
	00D533E4    B8 FD96900D     mov eax,0xD9096FD
	00D533E9    A5              movs dword ptr es:[edi],dword ptr ds:[es>
	00D533EA    BC CB37FE52     mov esp,0x52FE37CB
	00D533EF    66:27           daa
	00D533F1    9D              popfd

再加上这种乱序的结构，就是VMP。
#### 5.Shielden ####
区段信息中，默认是.SEDATA	但是会有两个相同的，如果分析一下，简单F7前两个JMP之后，就能看到其ASCII信息了。甚至版本也能出来。  

	0050A40D   > /E8 1C000000   call 吾爱破解.0050A42E                       ;  PUSH ASCII "Safengine Shielden v2.3.6.0"
	0050A412   . |53 61 66 65 6>ascii "Safengine Shield"
	0050A422   . |65 6E 20 76 3>ascii "en v2.3.6.0",0
	0050A42E   > |9C            pushfd
	0050A42F   . |E8 4CFFFFFF   call 吾爱破解.0050A380
	0050A434   > |66:8B1C24     mov bx,word ptr ss:[esp]
	0050A438   . |86DD          xchg ch,bl
	0050A43A   . |66:87F1       xchg cx,si
	0050A43D   . |0f9ec1        setle cl
	0050A440   . |8A0C24        mov cl,byte ptr ss:[esp]


小组成员: 邱容兴, 2019级网安2班, GitHub名称: rxQiu  
完成项目名称(仅个人,未组队):  


`1.python SM3实现  
2.SM3生日攻击  
3.The Rho method of SM3  
4.SM3长度拓展攻击`
  
### 1.SM3实现(参考国家标准文档)
tips:所使用的结构为str,能将消息哈希出结果  
reference:`http://www.sca.gov.cn/sca/xwdt/2010-12/17/1002389/files/302a3ada057c4a73830536d03e683110.pdf`  
SM3是我国采用的一种密码散列算法,在SHA-256基础上进行改进,消息分组长度为512bit,生成消息摘要长度为256bit.算法流程分为4个部分: 填充比特,消息拓展,输出结果.下面我们对每一部分进行解释.  
#### 1.1填充比特  
我们需要对消息m,转化为2进制后先填充1在填充0至规定长度,在填充消息长度的64位二进制.  
``` python
#函数传入消息16进制,输出填充后16进制串
def padding(m_hex):
    #数据长度
    m_lenth = len(m_hex)
    m_bin   =  bin(int(m_hex,16))[2:]
    #对于左侧为0的消息进行填充,这是由于对于16进制01而言,经过上述操作后会成为二进制1,缺少长度,所以zfill功能为在左侧补0到指定长度,即01即可成为0000 0001,而不是简单的1.
    bit = m_bin.zfill(4*m_lenth)
    #计算填充0的长度
    num = 448 - 1 - 4*m_lenth%512
    #数据长度二进制,长度为64位
    m_lenth_bin = bin(4*m_lenth)[2:].zfill(64)
    #进行填充,填充1和num长度的0和数据长度
    bit_padding = bit+ "1" + num*"0" + m_lenth_bin
    #16进制串长度
    words_len = len(bit_padding)//4
    #返回填充后的结果字
    return hex(int(bit_padding,2))[2:].zfill(words_len)
```

#### 1.2消息拓展  
由于需要用到循环移位,编写Rotate_left函数使用切片进行,先将16进制数据转为2进制串后,进行切片再转化为16进制.
``` python
def Rotate_left(X,n):
    x_bin = bin(int(X,16))[2:].zfill(32)
    #利用切片进行左移，返回结果字,比特长度为32,
    #每左移32位相当于未移动,所以使用切片n模32
    x_bin_rotate = x_bin[n%32:]+x_bin[:n%32]
    result = int(x_bin_rotate,2)
    #返回结果16进制
    return hex(result)[2:].zfill(8)
```

对于每一个512bit的消息分组,拓展生成132个字,使用在压缩函数处. W_0表示W,W_1表示W',先生成W前16个字,再根据算法,计算剩余的字.  
对于算法中的异或操作,python可以转化为int类型使用运算符号^.  
``` python
#每一个消息分组进行消息扩展
def Expansion(message_lump):
    W_0 = []
    W_1 = []
    for i in range(16):
        W_0.append(message_lump[8*i:8*i+8])
    for j in range(16,68):
        tmp1 = int(W_0[j-16],16)
        tmp2 = int(W_0[j-9],16)
        tmp3 = int(Rotate_left(W_0[j-3],15),16)
        tmp4 = int(Rotate_left(W_0[j-13],7),16)
        tmp5 = int(W_0[j-6],16)
        #计算异或
        tmp6 = hex(tmp1^tmp2^tmp3)[2:].zfill(8)
        #计算Wj数值
        tmp7 = int(P_1(tmp6),16) ^ tmp4 ^ tmp5
        #将Wj整数转化为字加入结果集合
        result = hex(tmp7)[2:].zfill(8)
        W_0.append(result)
    for k in range(64):
        num_0 = int( W_0[k],16)
        num_1 = int( W_0[k+4],16)
        tmp = hex(num_0 ^ num_1)[2:].zfill(8)
        W_1.append( tmp )
    return W_0,W_1
```

#### 1.3压缩函数  
压缩函数需要使用FF和GG两个函数,再此先进行介绍.FF函数和GG函数在前16轮和后48轮是不同的，所以我们传入参数进行控制.mode为0表示前16轮,mode为1表示后48轮.  
``` python
#X,Y,Z为字,mode为1时计算XYZ的异或
def FF(X,Y,Z,mode):
    x_num = int(X,16)
    y_num = int(Y,16)
    z_num = int(Z,16)
    if(mode == 0):
        result = x_num^y_num^z_num
    elif (mode==1):
        result = (x_num&y_num) | (x_num&z_num) | (y_num&z_num)
    #返回结果字
    word_result = hex(result)[2:].zfill(8)
    return word_result

def GG(X,Y,Z,mode):
    x_num = int(X,16)
    y_num = int(Y,16)
    z_num = int(Z,16)
    #X取反使用取符号~,为32位无符号整数，使用符号为有符号,可以加上2**32=4294967296,即与上0xFFFFFFFF
    x_reverse = ~x_num + 4294967296
    if(mode == 0):
        result = x_num^y_num^z_num
    elif (mode==1):
        result = (x_num&y_num) | (x_reverse&z_num)
    #返回结果字
    word_result = hex(result)[2:].zfill(8)
    return word_result
```

P置换存在两种,因此写了两个函数.初始量Tj存放于数组当中,前16轮使用Tj[0],后48轮使用TJ[1].初始IV在文档中已给.  


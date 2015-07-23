# udmabuf(User space mappable DMA Buffer)


## はじめに


### udmabufとは


udmabuf はLinux のカーネル空間に連続したメモリ領域をDMAバッファとして確保し、ユーザー空間からアクセス可能にするためのデバイスドライバです。主にUIO(User space I/O)を使ってユーザー空間でデバイスドライバを動かす場合のDMAバッファを提供します。

ユーザー空間でudmabufで確保したDMAバッファを利用する際は、デバイスファイル(/dev/udmabuf0など)をopen()して、mmap()でユーザー空間にマッピングするか、read()またはwrite()で行います。

openする際にO_SYNCフラグをセットすることによりCPUキャッシュを無効にすることが出来ます。

/sys/class/udmabuf/udmabuf0/phys_addr を読むことにより、DMAバッファの物理空間上のアドレスを知ることが出来ます。

udmabufのDMAバッファの大きさやデバイスのマイナー番号は、デバイスドライバのロード時(insmodによるロードなど)に指定できます。またプラットフォームによってはデバイスツリーに記述しておくこともできます。


### 構成



![図1 構成](./udmabuf1.jpg "図1 構成")

図1 構成

<br />




### 対応プラットフォーム


* OS : Linux Kernel Version 3.6 - 3.8 (私が動作を確認したのは 3.8です).

* CPU: ARM Cortex-A9 (ZYNQ)



## 使い方


### コンパイル


次のようなMakefileを用意しています。


```Makefile:Makefile
obj-m : udmabuf.o
all:
	make -C /usr/src/kernel M=$(PWD) modules
clean:
	make -C /usr/src/kernel M=$(PWD) clean
```





### インストール


insmod でudmabufのカーネルドライバをロードします。この際に引数を渡すことによりDMAバッファを確保してデバイスドライバを作成します。insmod の引数で作成できるDMAバッファはudmabuf0、udmabuf1、udmabuf2、udmabuf3の最大４つです。


```Shell
zynq$ insmod udmabuf.ko udmabuf0=1048576
udmabuf udmabuf0: driver installed
udmabuf udmabuf0: major number   = 248
udmabuf udmabuf0: minor number   = 0
udmabuf udmabuf0: phys address   = 0x1e900000
udmabuf udmabuf0: buffer size    = 1048576
zynq$ ls -la /dev/udmabuf0
crw------- 1 root root 248, 0 Dec  1 09:34 /dev/udmabuf0
```


アンインストールするには rmmod を使います。


```Shell
zynq$ rmmod udmabuf
udmabuf udmabuf0: driver uninstalled
```





### デバイスツリーによる設定


udmabufはinsmod の引数でDMAバッファを用意する以外に、Linuxのカーネルが起動時に読み込むdevicetreeファイルによってDMAバッファを用意する方法があります。devicetreeファイルに次のようなエントリを追加しておけば、insmod でロードする際に自動的にDMAバッファを確保してデバイスドライバを作成します。


```devicetree:devicetree.dts
		udmabuf0@devicetree {
			compatible = "ikwzm,udmabuf-0.10.a";
			minor-number = <0>;
			size = <0x00100000>;
		};

```




sizeでDMAバッファの容量をバイト数で指定します。

minor-number でudmabufのマイナー番号を指定します。マイナー番号は0から31までつけることができます。ただし、insmodの引数の方が優先され、マイナー番号がかち合うとdevicetreeで指定した方が失敗します。




```Shell
zynq$ insmod udmabuf.ko
udmabuf udmabuf0: driver installed
udmabuf udmabuf0: major number   = 248
udmabuf udmabuf0: minor number   = 0
udmabuf udmabuf0: phys address   = 0x1e900000
udmabuf udmabuf0: buffer size    = 1048576
zynq$ ls -la /dev/udmabuf0
crw------- 1 root root 248, 0 Dec  1 09:34 /dev/udmabuf0
```





### デバイスファイル


udmabufをinsmodでカーネルにロードすると、次のようなデバイスファイルが作成されます。

* /dev/udmabuf[0-31]

* /sys/class/udmabuf/udmabuf[0-31]/phys_addr

* /sys/class/udmabuf/udmabuf[0-31]/size

* /sys/class/udmabuf/udmabuf[0-31]/sync_mode



/dev/udmabuf[0-31]はmmap()を使って、ユーザー空間にマッピングするか、read()、write()を使ってバッファにアクセスする際に使用します。


```C:udmabuf_test.c
    if ((fd  = open("/dev/udmabuf0", O_RDWR)) != -1) {
        buf = mmap(NULL, buf_size, PROT_READ|PROT_WRITE, MAP_SHARED, fd, 0);
        /* ここでbufに読み書きする処理を行う */
        close(fd);
    }

```


また、ddコマンド等でにデバイスファイルを指定することにより、shellから直接リードライトすることも出来ます。


```Shell
zynq$ dd if=/dev/urandom of=/dev/udmabuf0 bs=4096 count=1024
1024+0 records in
1024+0 records out
4194304 bytes (4.2 MB) copied, 3.07516 s, 1.4 MB/s
```


```Shell
zynq$dd if=/dev/udmabuf4 of=random.bin
8192+0 records in
8192+0 records out
4194304 bytes (4.2 MB) copied, 0.173866 s, 24.1 MB/s
```




/sys/class/udmabuf/udmabuf[0-31]/phys_addr はDMAバッファの物理アドレスが読めます。


```C:udmabuf_test.c
    unsigned char  attr[1024];
    unsigned long  phys_addr;
    if ((fd  = open("/sys/class/udmabuf/udmabuf0/phys_addr", O_RDONLY)) != -1) {
        read(fd, attr, 1024);
        sscanf(attr, "%x", &phys_addr);
        close(fd);
    }

```


/sys/class/udmabuf/udmabuf[0-31]/size はDMAバッファのサイズが読めます。


```C:udmabuf_test.c
    unsigned char  attr[1024];
    unsigned int   buf_size;
    if ((fd  = open("/sys/class/udmabuf/udmabuf0/size", O_RDONLY)) != -1) {
        read(fd, attr, 1024);
        sscanf(attr, "%d", &buf_size);
        close(fd);
    }

```




/sys/class/udmabuf/udmabuf[0-31]/sync_mode はudmabufをopenする際にO_SYNCを指定した場合の動作を指定します。


```C:udmabuf_test.c
    unsigned char  attr[1024];
    unsigned long  sync_mode = 2;
    if ((fd  = open("/sys/class/udmabuf/udmabuf0/sync_mode", O_WRONLY)) != -1) {
        sprintf(attr, "%d", sync_mode);
        write(fd, attr, strlen(attr));
        close(fd);
    }
```


O_SYNCおよびキャッシュの設定に関しては次の節で説明します。




### DMAバッファとCPUキャッシュのコヒーレンシ


CPUは通常キャッシュを通じてメインメモリ上のDMAバッファにアクセスしますが、アクセラレータは直接メインメモリ上のDMAバッファにアクセスします。その際、問題になるのはCPUのキャッシュとメインメモリとのコヒーレンシ(内容の一貫性)です。

ハードウェアでコヒーレンシを保証できる場合、CPUキャッシュを有効にしても問題はありません。例えばZYNQにはACP(Accelerator Coherency Port)があり、アクセラレータ側がこのPortを通じてメインメモリにアクセスする場合は、ハードウェアによってCPUキャッシュとメインメモリとのコヒーレンシが保証できます。

ハードウェアでコヒーレンシを保証できない場合、別の方法でコヒーレンシを保証しなければなりません。udmabufでは単純にCPUがDMAバッファへのアクセスする際はCPUキャッシュを無効にすることでコヒーレンシを保証しています。CPUキャッシュを無効にする場合は、udmabufをopenする際にO_SYNCフラグを設定します。


```C:udmabuf_test.c
    /* CPUキャッシュを無効にする場合はO_SYNCをつけてopen する */
    if ((fd  = open("/dev/udmabuf0", O_RDWR | O_SYNC)) != -1) {
        buf = mmap(NULL, buf_size, PROT_READ|PROT_WRITE, MAP_SHARED, fd, 0);
        /* ここでbufに読み書きする処理を行う */
        close(fd);
    }

```


O_SYNCフラグを設定した場合のキャッシュの振る舞いはsync_modeで設定します。sync_modeには次の値が設定できます。

* sync_mode=0:  常にCPUキャッシュが有効。つまりO_SYNCフラグの有無にかかわらず常にCPUキャッシュは有効になります。

* sync_mode=1: O_SYNCフラグが設定された場合、CPUキャッシュを無効にします。O_SYNCフラグが設定されなかった場合、CPUキャッシュは有効です。

* sync_mode=2: O_SYNCフラグが設定された場合、CPUがDMAバッファに書き込む際、ライトコンバインします。ライトコンバインとは、基本的にはCPUキャッシュは無効ですが、複数の書き込みをまとめて行うことで若干性能が向上します。O_SYNCフラグが設定されなかった場合、CPUキャッシュは有効です。

* sync_mode=3: O_SYNCフラグが設定された場合、DMAコヒーレンシモードにします。といっても、DMAコヒーレンシモードに関してはまだよく分かっていません。O_SYNCフラグが設定されなかった場合、CPUキャッシュは有効です。

* sync_mode=4:  常にCPUキャッシュが有効。つまりO_SYNCフラグの有無にかかわらず常にCPUキャッシュは有効になります。

* sync_mode=5: O_SYNCフラグの有無にかかわらずCPUキャッシュを無効にします。

* sync_mode=6: O_SYNCフラグの有無にかかわらず、CPUがDMAバッファに書き込む際、ライトコンバインします。

* sync_mode=7: O_SYNCフラグの有無にかかわらず、DMAコヒーレンシモードにします。



参考までに、CPUキャッシュを有効/無効にした場合の次のようなプログラムを実行した際の処理時間を示します。


```C:udmabuf_test.c
int check_buf(unsigned char* buf, unsigned int size)
{
    int m = 256;
    int n = 10;
    int i, k;
    int error_count = 0;
    while(--n > 0) {
      for(i = 0; i < size; i = i + m) {
        m = (i+256 < size) ? 256 : (size-i);
        for(k = 0; k < m; k++) {
          buf[i+k] = (k & 0xFF);
        }
        for(k = 0; k < m; k++) {
          if (buf[i+k] != (k & 0xFF)) {
            error_count++;
          }
        }
      }
    }
    return error_count;
}
int clear_buf(unsigned char* buf, unsigned int size)
{
    int n = 100;
    int error_count = 0;
    while(--n > 0) {
      memset((void*)buf, 0, size);
    }
    return error_count;
}

```


表-1　checkbufの測定結果

<table border="2">
  <tr>
    <td align="center" rowspan="2">sync_mode</td>
    <td align="center" rowspan="2">O_SYNC</td>
    <td align="center" colspan="3">DMAバッファのサイズ</td>
  </tr>
  <tr>
    <td align="center">1MByte</td>
    <td align="center">5MByte</td>
    <td align="center">10MByte</td>
  </tr>
  <tr>
    <td rowspan="2">0</td>
    <td>無</td>
    <td align="right">0.437[sec]</td>
    <td align="right">2.171[sec]</td>
    <td align="right">4.340[sec]</td>
  </tr>
  <tr>
    <td>有</td>
    <td align="right">0.437[sec]</td>
    <td align="right">2.171[sec]</td>
    <td align="right">4.340[sec]</td>
  </tr>
  <tr>
    <td rowspan="2">1</td>
    <td>無</td>
    <td align="right">0.434[sec]</td>
    <td align="right">2.179[sec]</td>
    <td align="right">4.337[sec]</td>
  </tr>
  <tr>
    <td>有</td>
    <td align="right">2.283[sec]</td>
    <td align="right">11.414[sec]</td>
    <td align="right">22.830[sec]</td>
  </tr>
  <tr>
    <td rowspan="2">2</td>
    <td>無</td>
    <td align="right">0.434[sec]</td>
    <td align="right">2.169[sec]</td>
    <td align="right">4.337[sec]</td>
  </tr>
  <tr>
    <td>有</td>
    <td align="right">1.616[sec]</td>
    <td align="right">8.262[sec]</td>
    <td align="right">16.562[sec]</td>
  </tr>
  <tr>
    <td rowspan="2">3</td>
    <td>無</td>
    <td align="right">0.434[sec]</td>
    <td align="right">2.169[sec]</td>
    <td align="right">4.337[sec]</td>
  </tr>
  <tr>
    <td>有</td>
    <td align="right">1.600[sec]</td>
    <td align="right">8.391[sec]</td>
    <td align="right">16.587[sec]</td>
  </tr>
  <tr>
    <td rowspan="2">4</td>
    <td>無</td>
    <td align="right">0.437[sec]</td>
    <td align="right">2.171[sec]</td>
    <td align="right">4.337[sec]</td>
  </tr>
  <tr>
    <td>有</td>
    <td align="right">0.437[sec]</td>
    <td align="right">2.171[sec]</td>
    <td align="right">4.337[sec]</td>
  </tr>
  <tr>
    <td rowspan="2">5</td>
    <td>無</td>
    <td align="right">2.283[sec]</td>
    <td align="right">11.414[sec]</td>
    <td align="right">22.809[sec]</td>
  </tr>
  <tr>
    <td>有</td>
    <td align="right">2.283[sec]</td>
    <td align="right">11.414[sec]</td>
    <td align="right">22.840[sec]</td>
  </tr>
  <tr>
    <td rowspan="2">6</td>
    <td>無</td>
    <td align="right">1.655[sec]</td>
    <td align="right">8.391[sec]</td>
    <td align="right">16.587[sec]</td>
  </tr>
  <tr>
    <td>有</td>
    <td align="right">1.655[sec]</td>
    <td align="right">8.391[sec]</td>
    <td align="right">16.587[sec]</td>
  </tr>
  <tr>
    <td rowspan="2">7</td>
    <td>無</td>
    <td align="right">1.655[sec]</td>
    <td align="right">8.391[sec]</td>
    <td align="right">16.587[sec]</td>
  </tr>
  <tr>
    <td>有</td>
    <td align="right">1.655[sec]</td>
    <td align="right">8.391[sec]</td>
    <td align="right">16.587[sec]</td>
  </tr>
</table>

表-2　clearbufの測定結果

<table border="2">
  <tr>
    <td align="center" rowspan="2">sync_mode</td>
    <td align="center" rowspan="2">O_SYNC</td>
    <td align="center" colspan="3">DMAバッファのサイズ</td>
  </tr>
  <tr>
    <td align="center">1MByte</td>
    <td align="center">5MByte</td>
    <td align="center">10MByte</td>
  </tr>
  <tr>
    <td rowspan="2">0</td>
    <td>無</td>
    <td align="right">0.067[sec]</td>
    <td align="right">0.359[sec]</td>
    <td align="right">0.713[sec]</td>
  </tr>
  <tr>
    <td>有</td>
    <td align="right">0.067[sec]</td>
    <td align="right">0.362[sec]</td>
    <td align="right">0.716[sec]</td>
  </tr>
  <tr>
    <td rowspan="2">1</td>
    <td>無</td>
    <td align="right">0.067[sec]</td>
    <td align="right">0.362[sec]</td>
    <td align="right">0.718[sec]</td>
  </tr>
  <tr>
    <td>有</td>
    <td align="right">0.912[sec]</td>
    <td align="right">4.563[sec]</td>
    <td align="right">9.126[sec]</td>
  </tr>
  <tr>
    <td rowspan="2">2</td>
    <td>無</td>
    <td align="right">0.068[sec]</td>
    <td align="right">0.360[sec]</td>
    <td align="right">0.721[sec]</td>
  </tr>
  <tr>
    <td>有</td>
    <td align="right">0.063[sec]</td>
    <td align="right">0.310[sec]</td>
    <td align="right">0.620[sec]</td>
  </tr>
  <tr>
    <td rowspan="2">3</td>
    <td>無</td>
    <td align="right">0.068[sec]</td>
    <td align="right">0.361[sec]</td>
    <td align="right">0.715[sec]</td>
  </tr>
  <tr>
    <td>有</td>
    <td align="right">0.062[sec]</td>
    <td align="right">0.310[sec]</td>
    <td align="right">0.620[sec]</td>
  </tr>
  <tr>
    <td rowspan="2">4</td>
    <td>無</td>
    <td align="right">0.068[sec]</td>
    <td align="right">0.360[sec]</td>
    <td align="right">0.718[sec]</td>
  </tr>
  <tr>
    <td>有</td>
    <td align="right">0.067[sec]</td>
    <td align="right">0.360[sec]</td>
    <td align="right">0.710[sec]</td>
  </tr>
  <tr>
    <td rowspan="2">5</td>
    <td>無</td>
    <td align="right">0.913[sec]</td>
    <td align="right">4.562[sec]</td>
    <td align="right">9.126[sec]</td>
  </tr>
  <tr>
    <td>有</td>
    <td align="right">0.913[sec]</td>
    <td align="right">4.562[sec]</td>
    <td align="right">9.126[sec]</td>
  </tr>
  <tr>
    <td rowspan="2">6</td>
    <td>無</td>
    <td align="right">0.062[sec]</td>
    <td align="right">0.310[sec]</td>
    <td align="right">0.618[sec]</td>
  </tr>
  <tr>
    <td>有</td>
    <td align="right">0.062[sec]</td>
    <td align="right">0.310[sec]</td>
    <td align="right">0.619[sec]</td>
  </tr>
  <tr>
    <td rowspan="2">7</td>
    <td>無</td>
    <td align="right">0.062[sec]</td>
    <td align="right">0.310[sec]</td>
    <td align="right">0.620[sec]</td>
  </tr>
  <tr>
    <td>有</td>
    <td align="right">0.062[sec]</td>
    <td align="right">0.310[sec]</td>
    <td align="right">0.621[sec]</td>
  </tr>
</table>



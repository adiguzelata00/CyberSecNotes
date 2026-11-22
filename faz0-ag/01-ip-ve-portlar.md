# Ders 1: IP Adresleri ve Portlar
*Faz 0 · 21-22 Kasım 2026*

## Ana kavramlar

- **IP adresi:** Cihazın ağdaki adresi. IPv4: 4 oktet (octet), her biri 0–255 → `192.168.1.40`
- **Private / public:** Tek soru: *internette yönlendirilebilir (routable) mi?* "Kurumsal" ≠ public.
  | Private aralık | Not |
  |---|---|
  | `10.0.0.0 – 10.255.255.255` | Büyük kurumlar, kampüs (benim üniversite ağım: 10.x) |
  | `172.16.0.0 – 172.31.255.255` | Tuzak: 172.15 ve 172.32 **public** |
  | `192.168.0.0 – 192.168.255.255` | Ev ağları |
- **Özel adresler:** `127.0.0.1` loopback (cihazın kendisi) · `169.254.x.x` APIPA (DHCP'den IP alınamamış) · `100.64–100.127.x.x` CGNAT (Tailscale de bu aralığı kullanır) · `0.0.0.0` dinlemede "tüm adreslerim"
- **Port:** Cihazdaki numaralı kapı, arkasında bir servis dinler. 0–65535.
  - Well-known (0–1023) · Registered (1024–49151) · **Ephemeral (49152–65535):** istemciye verilen geçici, rastgele port
- **Socket:** `IP:port`. Bir bağlantının **iki ucu** vardır; `netstat` bağlantıyı değil, bu bilgisayardaki uçları listeler.
- **Port durumları:** open/listening (dinleyen var) · closed (kimse yok) · filtered (firewall engelliyor)
- **Dinleme adresi:** `127.0.0.1` → sadece bu bilgisayar · `0.0.0.0` → tüm ağlar (firewall izin verirse)
- **NAT:** Router, private IP:port ↔ public IP:port eşleşmesini **çeviri tablosuna** yazar. Tüm ev/kafe aynı public IP ile çıkar.
- **Port bir gelenektir, garanti değildir:** 443'ten SSH, C2 ya da sızdırılan veri de geçebilir.

### Ezber portlar
| Port | Servis | SOC notu |
|---|---|---|
| 20/21 | FTP | Şifresiz |
| 22 | SSH | Brute-force hedefi |
| 23 | Telnet | Şifresiz, açık görmek kırmızı bayrak |
| 25 | SMTP | E-posta gönderim, phishing analizi |
| 53 | DNS | DNS tunneling |
| 80 / 443 | HTTP / HTTPS | 443 = zararlıların gizlendiği yer |
| 110 / 143 | POP3 / IMAP | E-posta alma |
| 445 | SMB | WannaCry, yanal hareket (lateral movement) |
| 3389 | RDP | Ransomware giriş kapısı, internete asla açılmaz |

### İstemci / servis ayırt etme (3 kanıt)
1. Well-known port → servis
2. Ephemeral aralıktaki port → istemci
3. Process istemci uygulaması mı (ms-teams, tarayıcı)?
> "Büyük port = istemci" sadece sezgi kuralı: Samsung servisi 50911'de **dinliyordu**.

## Kullandığım komutlar

| Komut | Kabuk | Ne işe yarar |
|---|---|---|
| `ipconfig /all` | CMD/PS | IP, subnet mask, gateway, DNS, MAC |
| `curl ifconfig.me` | CMD/PS | Dışarıdan görünen public IP |
| `netstat -ano \| findstr LISTENING` | CMD/PS | Dinleyen portlar + PID |
| `netstat -ano \| findstr ESTABLISHED` | CMD/PS | Açık bağlantılar |
| `tasklist /fi "pid eq 1234"` | CMD/PS | PID → process adı |
| `tasklist /svc /fi "pid eq 1234"` | CMD/PS | svchost içinde hangi servis var |
| `Get-Process -Id 1234 \| Select-Object Id, ProcessName, Path` | **Sadece PowerShell** | Process'in diskteki yolu (kanıt) |
| `python -m http.server 8000 --bind 127.0.0.1` | CMD/PS | Kendi test sunucum, sadece localhost'a açık |
| `$c = New-Object System.Net.Sockets.TcpClient('127.0.0.1', 8000)` | PowerShell | Bağlantıyı açık tutup ESTABLISHED görmek · `$c.Close()` ile kapat |

**netstat sütunları:** `Proto | Local Address | Foreign Address | State | PID`
Local Address = **adres:port**. PID **son sütun**.
LISTENING satırında `0.0.0.0:0` (Foreign) = karşıda henüz kimse yok.
TIME_WAIT = kapanmış bağlantının kalıntısı, PID 0.

## Bulgularım (kendi bilgisayarım)
- CDPSvc (svchost, `C:\Windows\System32`) → `0.0.0.0:5040`
- Samsung `ss_conn_service` → `127.0.0.1:50911`
- tailscaled → `100.75.x:39039` (sadece Tailscale ağı)
- ms-teams → `52.123.x:443` (ipinfo: Microsoft) → normal (baseline)
- Kampüs ağı 10.x, Windows profili **Ortak**

## Sık yaptığım hatalar
- [ ] Komut adını yanlış yazmak (`tasklisting`, `/svc/fi`, `pif`) → Enter'dan önce komutu oku
- [ ] PowerShell cmdlet'ini CMD'de çalıştırmak → prompt `PS` ile mi başlıyor?
- [ ] Local Address'teki portu PID sanmak; LISTENING satırında Foreign'ı dinleme adresi sanmak
- [ ] "Kurumsal" adresi public saymak
- [ ] Ephemeral port grubu yerine "TCP" demek (protokol ≠ port grubu)
- [ ] "Açık" deyip **kime** açık olduğunu yazmamak
- [ ] Process adına güvenmek → isim tahmindir, **path kanıttır** (Masquerading, T1036)
- [ ] Gerekçeyi IP sınıfına dayandırmak; istemci/servis için **portlara** bak
- [ ] Tablonun tek sütununa bakıp saymak

## Alışkanlıklar
- Hata mesajının **ilk satırını** oku
- İsim → tahmin, path → kanıt
- IP sahibini doğrula (ipinfo.io)
- "Bu process'in bu adrese bağlanması normal mi?"
- Kamusal ağda Windows profili **Ortak** olmalı

## Üç alan bağlantısı
- **SOC:** `SRC:port → DST:port` satırından yön, servis ve normallik çıkarılır
- **Pentest:** Servisi alışılmadık porta taşıyarak port tabanlı tespiti atlatmak
- **DFIR:** Saldırganı bulma zinciri: site logu (public IP + port + saat) → NAT logu → DHCP logu → MAC → cihaz. Bellekte bağlantılar: Volatility `netscan` (PID ile)

## Tekrar soruları

1. `netstat` çıktısında `TCP 10.39.63.5:50239 52.123.137.3:443 ESTABLISHED 17180` satırını gördün. İstemci kim, servis ne, karşı taraf private mı public mi, bu bağlantıyı açan programı nasıl bulursun?
2. Bir sunucuda `0.0.0.0:3306` LISTENING görüyorsun. Bu port hangi servis olabilir, bu satır neden endişe verici, ve "gerçekten erişilebilir mi?" sorusunu cevaplamak için neye bakarsın?
3. Logda `SRC 45.12.x.x:58211 → DST 10.0.0.15:22`, 5 dakikada 300 kez. Yön ne, hangi servis, ne oluyor, kaynak portun büyük olmasının sebebi ne?

<details>
<summary>Cevaplar</summary>

1. İstemci benim (`10.39.63.5:50239`, ephemeral), servis HTTPS (443). Karşı taraf public (52, private aralıkta değil). `tasklist /fi "pid eq 17180"` → program; `Get-Process -Id 17180 | Select Path` → kanıt; ipinfo ile IP sahibi.
2. 3306 = MySQL (registered). `0.0.0.0` → tüm ağlarda dinliyor, veritabanı dışarıya açık olabilir. Firewall kurallarına ve (Nmap haftasında) dışarıdan taramaya bakılır; dinlemek ≠ erişilebilir.
3. Dışarıdan (public) içeriye (private), SSH'a brute-force. 58211 saldırganın ephemeral portu.
</details>

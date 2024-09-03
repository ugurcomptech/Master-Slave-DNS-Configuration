# Master Slave DNS Configuration

Bu proje, bir master ve bir slave DNS sunucusunun yapılandırmasını içermektedir. Master DNS sunucusu, ana veritabanı dosyalarını yönetir ve DNS kayıtlarını oluşturur, günceller. Slave DNS sunucusu ise master sunucusundan bu kayıtları senkronize ederek yedekleme ve yük dengeleme sağlar.


## BIND Servisini İndirmek


Aşağıdaki komutları yazarak BIND servisini indirelim

```
sudo apt-get update
sudo apt-get install build-essential libssl-dev libgeoip-dev
```



## Primary DNS Server Yapılandırması

**/etc/bind/named.conf** dosyasına yeni bir zonu tanımlayalım:

```
zone "example.com" {
    type master;
    file "/etc/bind/zones/example.com.db";
    allow-transfer { secondary-server-ip; };
};
```

### DNS Zone Dosyasını Oluşturma

İlk öncelikle **zones** klasörünü oluşturmamız gerekiyor.

```
mkdir -p zones
```


**/etc/bind/zones/example.com.db** dosyasını oluşturun ve aşağıdaki gibi yapılandıralım:

```
$TTL 604800
@   IN  SOA dns1.domain.com dns.domain.com (
  2024090111 ; Serial
        604800     ; Refresh
        86400      ; Retry
        2419200    ; Expire
        604800 )   ; Negative Cache TTL
;


@       IN  A       1.1.1.1
www     IN  A       1.1.1.1
mail    IN  A       1.1.1.1

@       IN  MX   0  mail.domain.com.


@       IN  NS      dns1.domain.com.
@       IN  NS      dns2.domain.com.

```

### BIND'i Yeniden Başlatma

```
sudo systemctl restart bind9
```

## Secondary DNS Server Yapılandırması

### Zone Dosyasını Tanımlama


**/etc/bind/named.conf** dosyasına aşağıdaki zon tanımını ekleyin:

```
zone "example.com" {
    type master;
    file "/etc/bind/zones/example.com.db";
    allow-transfer { primary-server-ip; };
};
```

### BIND'i Yeniden Başlatma

```
sudo systemctl restart bind9
```


## Senkronizasyon

BIND DNS sunucuları, Primary sunucuda yapılan değişiklikleri, belirli aralıklarla Secondary sunucuya otomatik olarak senkronize edecektir. Serial numarasını her güncellemede arttırmanız gerekir. Bu numara, Secondary sunucunun Primary sunucudaki değişiklikleri fark etmesi için gereklidir. Sizler  bir kayıt değiştiğiniz zaman senkronizasyon gerçekleştirilir.

Bu şekilde BIND-DNS ile senkron çalışan bir DNS yapısı kurabilirsiniz. Bu yapı, DNS hizmetlerinin sürekliliğini sağlamak ve yük dağılımı yapmak için idealdir.


## ÖNEMLİ NOTLAR

**NOT 1:** Dışarıdan eğer DNS servisinize erişemez iseniz yüksek ihtimalle UFW veya FİREWALLD üzerinden **53/udp** portuna veya **BIND9** servisine izin vermediğinizden dolayıdır.

```
ufw allow 53/udp
ufw allow bınd9
ufw reload
```

**NOT 2:** Eğer `zone` dosyalarınızı slave sunucunuzda yedeklemek istiyor iseniz aşağıdaki komutu çalıştırabilirsiniz. İsterseniz bunu crontaba bunu ekleyebilirsiniz.

```
0 * * * * /etc/bind/serial.sh
```

```
#!/bin/bash

# Variables
MASTER_ZONE_PATH="/etc/bind/zones/domain.com.db"
SLAVE_SERVER="ubuntu@1.1.1.1"
SLAVE_ZONE_PATH="/var/cache/bind/domain.com.db"
RSYNC_KEY_PATH="/root/private.key"

# Update the serial number in the zone file
echo "Güncellemeler yapılacak olan dosyayı bulup güncelliyoruz..."

# Copy the zone file to the slave server
echo "Zone dosyasını slave sunucuya kopyalıyoruz..."
rsync -avz -e "ssh -i $RSYNC_KEY_PATH" --delete $MASTER_ZONE_PATH $SLAVE_SERVER:$SLAVE_ZONE_PATH

# Restart the BIND service
echo "Bind servisini yeniden başlatıyoruz..."
sudo systemctl restart named.service

echo "Senkronizasyon ve güncellemeler tamamlandı."
```









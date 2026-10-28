# OKD 4.21 Kurulum Kontrol Listesi

**Hedef sürüm:** `4.21.0-okd-scos.11`  
**Cluster adı:** `okdtest`  
**Base domain:** `example.com`

## 1. Kurulum Öncesi

### 1.1. Sanal Makineler

- [ ] 3 master, 3 worker, 1 bootstrap ve 1 bastion (yönetim makinesi) oluşturuldu.
- [ ] Her makineye test ortamına uygun **8 vCPU, 12 GB RAM ve 150 GB disk** atandı.
- [ ] Her makinenin hedef kurulum diski belirlendi; `/dev/sda` olduğu varsayılmadan kontrol edildi.
- [ ] IP adresleri ve hostname’ler aşağıdaki tabloyla eşleştirildi.
- [ ] SCOS ISO, bootstrap, master ve worker makinelerine bağlandı.
- [ ] İlk kurulum için makinelerin ISO üzerinden açılması sağlandı.
- [ ] Diske kurulum sonrasında **kurulum diskinin boot sıralamasında ilk sıraya alınması** planlandı.

| Makine | IP adresi | Hostname |
|---|---|---|
| Master 1 | 192.168.20.71 | okdtest-master-01 |
| Master 2 | 192.168.20.72 | okdtest-master-02 |
| Master 3 | 192.168.20.73 | okdtest-master-03 |
| Worker 1 | 192.168.20.74 | okdtest-worker-01 |
| Worker 2 | 192.168.20.75 | okdtest-worker-02 |
| Worker 3 | 192.168.20.76 | okdtest-worker-03 |
| Bootstrap | 192.168.20.77 | bootstrap |
| Bastion | 192.168.20.32 | bm-bastion |
| Load balancer | 192.168.20.78 | okdtest-lb |

### 1.2. Bastion ve Kurulum Dosyaları

- [ ] `4.21.0-okd-scos.11` sürümüne ait installer indirildi ve arşivden çıkartıldı.
- [ ] `scos-10.0.20251103-0-live-iso.x86_64.iso` indirildi.
- [ ] Bastion üzerinde Nginx hazır.
- [ ] Node’lardan bastion’ın `192.168.20.32` adresindeki HTTP servisine erişilebiliyor.
- [ ] Çalışma dizini ve `install-dir` dizini oluşturuldu.
- [ ] `install-config.yaml` hazırlandı.
- [ ] Cluster adı `okdtest`, base domain `example.com` olarak ayarlandı.
- [ ] `controlPlane.replicas: 3`, `compute.replicas: 0` ve `platform.none: {}` değerleri kontrol edildi.
- [ ] Pull secret ve SSH public key alanları dolduruldu.
- [ ] SSH public key’in karşılığı olan private key bastion’dan bağlantı için kullanılabilir.
- [ ] `install-config.yaml`, `install-dir` içine kopyalandı.
- [ ] Manifestolar oluşturuldu.
- [ ] `cluster-scheduler-02-config.yml` içinde `mastersSchedulable: false` ayarlandı.
- [ ] Bu değişiklikten sonra Ignition dosyaları oluşturuldu.
- [ ] Ignition dosyaları `/usr/share/nginx/html/` dizinine kopyalandı.
- [ ] Ignition dosyalarının izinleri `644` olarak ayarlandı.
- [ ] `bootstrap.ign`, `master.ign` ve `worker.ign` HTTP üzerinden erişilebilir.
- [ ] Bastion üzerinde `oc` komutu kullanılabilir ve `install-dir/auth/kubeconfig` dosyası mevcut.

### 1.3. Network

- [ ] Makine ağı `192.168.20.0/23` olarak kontrol edildi.
- [ ] Node’larda kullanılacak ağ arayüzü doğrulandı; örnekte `ens18`.
- [ ] Ağ geçidi `192.168.20.1` erişilebilir.
- [ ] DNS sunucusu `192.168.20.2` erişilebilir.
- [ ] DNS arama alanı `example.com` olarak ayarlandı.
- [ ] IP adreslerinin benzersiz olduğu ve çakışma bulunmadığı kontrol edildi.
- [ ] Bastion’dan node’lara SSH erişimi için ağ yolu hazır.
- [ ] Node’ların imaj indirmek için `quay.io` erişimi mevcut.
- [ ] Aşağıdaki load balancer ve backend bağlantılarına ağ erişimi sağlandı.

| Kaynak / hedef | Port | Kullanım |
|---|---|---|
| Load balancer → bootstrap ve master’lar | TCP 6443 | Kubernetes API |
| Load balancer → bootstrap ve master’lar | TCP 22623 | Machine Config Server |
| Load balancer → worker’lar | TCP 80 ve 443 | Uygulama trafiği |
| Load balancer → worker’lar | TCP 1936 | Ingress sağlık kontrolü |
| Node’lar → bastion | TCP 80 | Ignition indirme |

### 1.4. DNS

- [ ] `api.okdtest.example.com` → `192.168.20.78`
- [ ] `api-int.okdtest.example.com` → `192.168.20.78`
- [ ] `*.apps.okdtest.example.com` → `192.168.20.78`
- [ ] API reverse DNS kaydı `192.168.20.78` için tanımlandı.
- [ ] `bootstrap.okdtest.example.com` → `192.168.20.77`
- [ ] Üç master’ın DNS kayıtları doğru IP adreslerine çözülüyor.
- [ ] Üç worker’ın DNS kayıtları doğru IP adreslerine çözülüyor.
- [ ] DNS çözümlemesi bastion ve node’ların bulunduğu ağdan doğrulandı.

### 1.5. Load Balancer

- [ ] HAProxy, `192.168.20.78` adresinde çalışıyor.
- [ ] `6443` frontend’i bootstrap ve master’ların `6443` portuna yönleniyor.
- [ ] `22623` frontend’i bootstrap ve master’ların `22623` portuna yönleniyor.
- [ ] `443` frontend’i worker’ların `443` portuna yönleniyor.
- [ ] `80` frontend’i worker’ların **80** portuna yönleniyor.
- [ ] Bootstrap, başlangıçta her iki control plane backend’inde tanımlı.
- [ ] API sağlık kontrolü `/readyz`, Machine Config Server sağlık kontrolü `/healthz` olarak yapılandırıldı.
- [ ] Worker sağlık kontrolleri `1936` portunda `/healthz/ready` yolunu kullanıyor.
- [ ] HAProxy stats socket ve durum kontrol komutu çalışıyor.
- [ ] Servisler başlamadan önce backend’lerin `DOWN` görünmesinin beklenen başlangıç durumu olduğu not edildi.

```bash
echo "show stat" | sudo socat stdio /var/lib/haproxy/stats | grep -vE ',(FRONTEND|BACKEND),' | cut -d, -f1,2,18 | column -s, -t
```

## 2. Bootstrap Sağlık Kontrolü ve Node Boot Zamanlaması

### 2.1. Bootstrap’ın Kurulması

- [ ] Bootstrap ISO üzerinden başlatıldı.
- [ ] IP adresi `192.168.20.77/23`, gateway ve DNS ayarları uygulandı.
- [ ] Hostname `bootstrap` olarak ayarlandı.
- [ ] Bastion’dan `bootstrap.ign` indirildi.
- [ ] Doğru hedef disk kontrol edilerek `coreos-installer` ile kurulum yapıldı.
- [ ] Kurulum diski boot sıralamasında ilk sıraya alındı.
- [ ] Bootstrap yeniden başlatılarak diskten açıldı.
- [ ] Bastion’dan `core` kullanıcısıyla SSH bağlantısı kurulabildi.

### 2.2. Bootstrap Sağlık Kontrolleri

- [ ] Release imajı ve node imajı indirme süreci loglardan takip edildi.
- [ ] `release-image`, `bootkube` ve `node-image-pull` servis logları kontrol edildi.
- [ ] Container durumları kontrol edildi.
- [ ] `quay.io` erişimi kontrol edildi.
- [ ] HAProxy’de bootstrap’ın `controlplaneapi` ve `controlplaneapiinternal` kayıtları `UP` oldu.
- [ ] Bootstrap açıldıktan yaklaşık 5–10 dakika sonra worker Ignition endpoint’inden **HTTP 200** alındı.
- [ ] Kubernetes API `/version` sorgusuna yanıt verdi.
- [ ] Load balancer üzerinden API `/readyz` sorgusu **`ok`** döndürdü.

**Bootstrap üzerinde:**

```bash
journalctl -b -f -u release-image.service -u bootkube.service -u node-image-pull.service
```

```bash
sudo journalctl -b -u bootkube.service --no-pager | tail -50
sudo crictl ps -a | head
curl -I https://quay.io
```

**Bastion üzerinde:**

```bash
curl -kv -m 10 https://192.168.20.77:22623/config/worker -o /dev/null
curl -k https://192.168.20.77:6443/version
curl -kv https://api.okdtest.example.com:6443/readyz
```

> API kök adresinin anonim isteğe `403 Forbidden` dönmesi, API’nin erişilebilir olduğunu gösterir. Hazır olma kontrolünde `/readyz` yanıtının `ok` olması esas alınır.

### 2.3. Master ve Worker’ların Hazırlanması

- [ ] Bootstrap hazırlanırken altı node’un tamamında ağ ayarları yapıldı.
- [ ] Her node için IP adresi ve hostname ayrı ayrı kontrol edildi.
- [ ] Master’lara `master.ign`, worker’lara `worker.ign` indirildi.
- [ ] Her node’un hedef kurulum diski doğrulandı.
- [ ] Altı node’un tamamında diske kurulum tamamlandı.
- [ ] Kurulum diski tüm node’larda boot sıralamasında ilk sıraya alındı.
- [ ] Bootstrap hazır olmadan master ve worker’lar reboot edilmedi.

### 2.4. Master ve Worker’ların Başlatılması

- [ ] Bootstrap API `/readyz` yanıtının **`ok`** olduğu doğrulandı.
- [ ] Ardından üç master ve üç worker reboot edildi.
- [ ] Altı node’un tamamının ISO yerine kurulum diskinden açıldığı doğrulandı.

> **Geçiş ölçütü:** Master ve worker’ların ilk diskten açılışı için bootstrap API’sinin hazır olması beklenir. Bootstrap’ın kaldırılması için ise daha sonra installer’ın güvenli kaldırma mesajı vermesi gerekir.

## 3. Kurulum Sırasında Kontroller

### 3.1. Installer ve Node Takibi

- [ ] Bastion’da bootstrap tamamlanma komutunun çıktısı takip ediliyor.
- [ ] Node’larda kubelet servisinin durumu kontrol ediliyor.
- [ ] Gerektiğinde prosesler, Podman container’ları ve imajları inceleniyor.
- [ ] `install-dir/auth/kubeconfig` kullanılarak `oc` üzerinden cluster objelerine erişilebiliyor.
- [ ] Otomatik yeniden başlatmalarda node’ların diskten açıldığı doğrulanıyor.

```bash
./openshift-install wait-for bootstrap-complete --dir=./install-dir
```

### 3.2. Worker CSR Onayları

- [ ] Worker’ların oluşturduğu bekleyen CSR’lar kontrol edildi.
- [ ] Otomatik veya tek seferlik toplu onay yöntemlerinden biri uygulandı.
- [ ] Yeni CSR’lar geldikçe takip edildi.

**Otomatik onay:**

```bash
while : ; do
        [ -n "$(oc get csr | egrep -v 'Approved|REQUESTEDDURATION')" ] && \
                oc get csr -o go-template='{{range .items}}{{if not .status}}{{.metadata.name}}{{"\n"}}{{end}}{{end}}' | xargs --no-run-if-empty oc adm certificate approve
        sleep 10
done
```

**Tek seferlik toplu onay:**

```bash
oc get csr -o go-template='{{range .items}}{{if not .status}}{{.metadata.name}}{{"\n"}}{{end}}{{end}}' | xargs --no-run-if-empty oc adm certificate approve
```

### 3.3. Load Balancer ve Control Plane

- [ ] Master’lar hazır oldukça API ve Machine Config Server backend durumları kontrol edildi.
- [ ] Master backend’lerinin `UP` durumuna geçtiği görüldü.
- [ ] Bootstrap’ın `DOWN` görünmesi tek başına kaldırma ölçütü olarak kullanılmadı.
- [ ] Worker ingress backend’lerinin kurulum sürerken `DOWN` olabileceği dikkate alınarak kurulum takibine devam edildi.

### 3.4. Bootstrap’ın Kaldırılması

- [ ] Installer çıktısında `Bootstrap etcd member has been removed` mesajı görüldü.
- [ ] Installer çıktısında **`It is now safe to remove the bootstrap resources`** mesajı görüldü.
- [ ] Bu mesajdan sonra bootstrap makinesi poweroff yapıldı.
- [ ] HAProxy `controlplaneapi` bölümündeki bootstrap satırı kaldırıldı.
- [ ] HAProxy `controlplaneapiinternal` bölümündeki bootstrap satırı kaldırıldı.
- [ ] Güncellenen HAProxy yapılandırması devreye alındı.
- [ ] Stats çıktısında bootstrap satırlarının artık bulunmadığı doğrulandı.
- [ ] Kalan backend’lerin durumları izlenmeye devam edildi.

## 4. Kurulumun Bitirilmesi

### 4.1. Node Kontrolü

```bash
oc get no
```

- [ ] Üç master’ın tamamı `Ready`.
- [ ] Üç worker’ın tamamı `Ready`.
- [ ] Master rolleri `control-plane,master`.
- [ ] Worker rolleri `worker`.

### 4.2. Machine Config Pool Kontrolü

```bash
oc get mcp
```

- [ ] Master havuzunda `UPDATED=True`, `UPDATING=False`, `DEGRADED=False`.
- [ ] Worker havuzunda `UPDATED=True`, `UPDATING=False`, `DEGRADED=False`.
- [ ] Her iki havuzda `MACHINECOUNT=3`.
- [ ] Her iki havuzda `READYMACHINECOUNT=3`.
- [ ] Her iki havuzda `UPDATEDMACHINECOUNT=3`.
- [ ] Her iki havuzda `DEGRADEDMACHINECOUNT=0`.

### 4.3. Cluster Operator Kontrolü

```bash
oc get co
```

- [ ] Tüm Cluster Operator’larda `AVAILABLE=True`.
- [ ] Tüm Cluster Operator’larda `PROGRESSING=False`.
- [ ] Tüm Cluster Operator’larda `DEGRADED=False`.
- [ ] Operator sürümleri hedef sürüm `4.21.0-okd-scos.11` ile uyumlu.

### 4.4. Kapanış

- [ ] Bootstrap kapatıldı ve load balancer tanımları kaldırıldı.
- [ ] Otomatik CSR onay döngüsü kullanıldıysa sonlandırıldı.
- [ ] Node, Machine Config Pool ve Cluster Operator son kontrol çıktıları kaydedildi.
- [ ] Genel kontroller tamamlandı ve kurulum işlemi bitirildi.
# OKD 4.21 Kurulum Dokümanı

## 1. Ön Bilgi

Bu doküman, **OKD 4.21.0-okd-scos.11** sürümünün kurulum adımlarını içerir.

Test sürecinde birçok farklı sürüm denenmiş ve bu testlerde kurulum hatası vermeyen tek sürüm **4.21.0-okd-scos.11** olmuştur.

Kurulum ortamı aşağıdaki makinelerden oluşur:

- 3 master
- 3 worker
- 1 bootstrap — geçici; bootstrap süreci tamamlandıktan sonra kaldırılacak
- 1 bastion (yönetim makinesi)

Kurulum işlemleri bastion üzerinde başlatılır.

## 2. Ön Hazırlık

### 2.1. Sanal Makine Kaynakları

Test ortamında her sunucu için aşağıdaki kaynaklar kullanılmıştır:

| Kaynak | Değer |
|---|---|
| CPU | 8 vCPU |
| RAM | 12 GB |
| Disk | 150 GB, tek disk |

### 2.2. Makine ve IP Bilgileri

| Makine | Rol | IP adresi | vCPU | RAM | Disk |
|---|---|---|---|---|---|
| okdtest-master-01.okdtest.example.com | Master | 192.168.20.71 | 8 | 12 GB | 150 GB |
| okdtest-master-02.okdtest.example.com | Master | 192.168.20.72 | 8 | 12 GB | 150 GB |
| okdtest-master-03.okdtest.example.com | Master | 192.168.20.73 | 8 | 12 GB | 150 GB |
| okdtest-worker-01.okdtest.example.com | Worker | 192.168.20.74 | 8 | 12 GB | 150 GB |
| okdtest-worker-02.okdtest.example.com | Worker | 192.168.20.75 | 8 | 12 GB | 150 GB |
| okdtest-worker-03.okdtest.example.com | Worker | 192.168.20.76 | 8 | 12 GB | 150 GB |
| bootstrap.okdtest.example.com | Bootstrap, geçici | 192.168.20.77 | 8 | 12 GB | 150 GB |
| bm-bastion | Bastion (yönetim makinesi) | 192.168.20.32 | 8 | 12 GB | 150 GB |

API ve ingress için kullanılan load balancer adresi **192.168.20.78**’dir.

### 2.3. Kurulum Dosyaları ve Boot Sırası

**OpenShift Installer:**

[openshift-install-linux-4.21.0-okd-scos.11.tar.gz](https://github.com/okd-project/okd/releases/download/4.21.0-okd-scos.11/openshift-install-linux-4.21.0-okd-scos.11.tar.gz)

**SCOS ISO:**

[scos-10.0.20251103-0-live-iso.x86_64.iso](https://rhcos.mirror.openshift.com/art/storage/prod/streams/c10s/builds/10.0.20251103-0/x86_64/scos-10.0.20251103-0-live-iso.x86_64.iso)

ISO indirilerek bootstrap, master ve worker makinelerine bağlanmalı ve bu makineler ilk kurulum için ISO üzerinden başlatılmalıdır.

> **Boot sırası:** ISO üzerinden diske kurulum tamamlandıktan sonra, boot sıralamasında kurulum diski ilk sıraya alınmalıdır. Kurulum sürecinde makineler yeniden başlayabileceğinden, sonraki açılışların ISO yerine kurulum diskinden yapılması sağlanmalıdır.

### 2.4. Bastion Üzerindeki Web Sunucusu

Ignition dosyalarının diğer makineler tarafından indirilebilmesi için bastion üzerinde bir web sunucusu bulunmalıdır.

Bu örnekte **Nginx** kullanılmıştır. Ignition dosyaları `/usr/share/nginx/html/` dizininden, **192.168.20.32** adresi üzerinden sunulur.

### 2.5. DNS Kayıtları

Makineler için DNS kayıtları hazırlanmalıdır. API, dahili API ve uygulama wildcard kayıtları load balancer adresine yönlendirilmelidir.

Örnek DNS yapılandırması:

```ini
# API
address=/api.okdtest.example.com/192.168.20.78
address=/api-int.okdtest.example.com/192.168.20.78

# Uygulama wildcard
address=/.apps.okdtest.example.com/192.168.20.78

# API reverse DNS
ptr-record=78.20.34.10.in-addr.arpa,api.okdtest.example.com

# Bootstrap - geçici
host-record=bootstrap.okdtest.example.com,192.168.20.77

# Master node'lar
host-record=okdtest-master-01.okdtest.example.com,192.168.20.71
host-record=okdtest-master-02.okdtest.example.com,192.168.20.72
host-record=okdtest-master-03.okdtest.example.com,192.168.20.73

# Worker node'lar
host-record=okdtest-worker-01.okdtest.example.com,192.168.20.74
host-record=okdtest-worker-02.okdtest.example.com,192.168.20.75
host-record=okdtest-worker-03.okdtest.example.com,192.168.20.76
```

### 2.6. Load Balancer Yapılandırması

Test ortamında load balancer olarak **HAProxy** kullanılmıştır.

| Dinlenen adres | Kullanım | Backend |
|---|---|---|
| 192.168.20.78:6443 | Kubernetes API | Bootstrap ve master makineleri, port 6443 |
| 192.168.20.78:22623 | Machine Config Server | Bootstrap ve master makineleri, port 22623 |
| 192.168.20.78:443 | HTTPS uygulama trafiği | Worker makineleri, port 443 |
| 192.168.20.78:80 | HTTP uygulama trafiği | Worker makineleri, port 80 |

HAProxy yapılandırması:

```haproxy
global
    log         127.0.0.1 local2
    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    user        haproxy
    group       haproxy
    daemon

    stats socket /var/lib/haproxy/stats

    #ssl-default-bind-ciphers PROFILE=SYSTEM
    #ssl-default-server-ciphers PROFILE=SYSTEM

defaults
    mode                    tcp
    log                     global
    option                  httplog
    option                  dontlognull
    option http-server-close
    option forwardfor       except 127.0.0.0/8
    option                  redispatch
    retries                 3
    timeout http-request    10s
    timeout queue           1m
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    timeout http-keep-alive  10s
    timeout check           10s
    maxconn                 3000

frontend api
    bind 192.168.20.78:6443
    default_backend controlplaneapi

frontend apiinternal
    bind 192.168.20.78:22623
    default_backend controlplaneapiinternal

frontend secure
    bind 192.168.20.78:443
    default_backend secure

frontend insecure
    bind 192.168.20.78:80
    default_backend insecure

backend controlplaneapi
    balance source
    option httpchk GET /readyz HTTP/1.0
    http-check expect status 200
    server bootstrap 192.168.20.77:6443 check check-ssl verify none inter 5s fall 3 rise 2   # bootstrap-complete sonrası sil
    server master0 192.168.20.71:6443 check check-ssl verify none inter 5s fall 3 rise 2
    server master1 192.168.20.72:6443 check check-ssl verify none inter 5s fall 3 rise 2
    server master2 192.168.20.73:6443 check check-ssl verify none inter 5s fall 3 rise 2

backend controlplaneapiinternal
    balance source
    option httpchk GET /healthz HTTP/1.0
    http-check expect status 200
    server bootstrap 192.168.20.77:22623 check check-ssl verify none inter 5s fall 3 rise 2   # bootstrap-complete sonrası sil
    server master0 192.168.20.71:22623 check check-ssl verify none inter 5s fall 3 rise 2
    server master1 192.168.20.72:22623 check check-ssl verify none inter 5s fall 3 rise 2
    server master2 192.168.20.73:22623 check check-ssl verify none inter 5s fall 3 rise 2

backend secure
    balance source
    option httpchk GET /healthz/ready HTTP/1.0
    http-check expect status 200
    server worker0 192.168.20.74:443 check port 1936 inter 5s fall 3 rise 2
    server worker1 192.168.20.75:443 check port 1936 inter 5s fall 3 rise 2
    server worker2 192.168.20.76:443 check port 1936 inter 5s fall 3 rise 2

backend insecure
    balance source
    option httpchk GET /healthz/ready HTTP/1.0
    http-check expect status 200
    server worker0 192.168.20.74:80 check port 1936 inter 5s fall 3 rise 2
    server worker1 192.168.20.75:80 check port 1936 inter 5s fall 3 rise 2
    server worker2 192.168.20.76:80 check port 1936 inter 5s fall 3 rise 2
```

Bootstrap kaynaklarının kaldırılmasının güvenli olduğu installer tarafından bildirildikten sonra, iki control plane backend bölümündeki `server bootstrap` satırları kaldırılacaktır.

### 2.7. HAProxy Backend Durum Kontrolü

HAProxy sunucusunda aşağıdaki komut kullanılır:

```bash
echo "show stat" | sudo socat stdio /var/lib/haproxy/stats | grep -vE ',(FRONTEND|BACKEND),' | cut -d, -f1,2,18 | column -s, -t
```

Makineler ilk oluşturulduğunda ve ilgili servisler henüz başlamadığında tüm backend’ler `DOWN` durumundadır:

```text
# pxname                 svname     status
controlplaneapi          bootstrap  DOWN
controlplaneapi          master0    DOWN
controlplaneapi          master1    DOWN
controlplaneapi          master2    DOWN
controlplaneapiinternal  bootstrap  DOWN
controlplaneapiinternal  master0    DOWN
controlplaneapiinternal  master1    DOWN
controlplaneapiinternal  master2    DOWN
secure                   worker0    DOWN
secure                   worker1    DOWN
secure                   worker2    DOWN
insecure                 worker0    DOWN
insecure                 worker1    DOWN
insecure                 worker2    DOWN
```

## 3. Bastion Üzerinde Kurulum Dosyalarının Hazırlanması

### 3.1. Installer ve Çalışma Dizini

Bastion üzerinde indirilen `openshift-install-linux-4.21.0-okd-scos.11.tar.gz` arşivi çıkartılır.

Yeni bir çalışma dizini hazırlanır ve bu dizinde `install-config.yaml` dosyası oluşturulur. Aşağıdaki komutlarda `openshift-install` programının çalışma dizininde bulunduğu varsayılmıştır.

### 3.2. install-config.yaml Oluşturulması

Örnek dosya içeriği:

```yaml
apiVersion: v1
baseDomain: example.com
compute:
- hyperthreading: Enabled
  name: worker
  replicas: 0
controlPlane:
  hyperthreading: Enabled
  name: master
  replicas: 3
metadata:
  name: okdtest
networking:
  machineNetwork:
  - cidr: 192.168.20.0/23
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  networkType: OVNKubernetes
  serviceNetwork:
  - 172.30.0.0/16
platform:
  none: {}
fips: false
pullSecret: 'redhat pull secret buraya girilir.'
sshKey: 'buraya ssh key girilir'
```

`pullSecret` alanına Red Hat pull secret, `sshKey` alanına SSH public key girilir.

Bu örnekte `compute.replicas` değeri **0** olarak kullanılmıştır. Ortamda hazırlanacak worker makinesi sayısı **3**’tür.

### 3.3. install-dir Dizinine Kopyalama

Çalışma dizini içinde `install-dir` adlı dizin oluşturulur. Ardından yapılandırma dosyası bu dizine kopyalanır:

```bash
cp install-config.yaml install-dir/
```

### 3.4. Manifestoların Oluşturulması

Bastion üzerinde aşağıdaki komut çalıştırılır:

```bash
./openshift-install create manifests --dir=install-dir
```

Örnek çıktı:

```text
INFO Consuming Install Config from target directory
WARNING Making control-plane schedulable by setting MastersSchedulable to true for Scheduler cluster settings
INFO Successfully populated MCS CA cert information: root-ca 2036-10-05T11:27:57Z 2026-10-08T11:27:57Z
INFO Successfully populated MCS TLS cert information: root-ca 2036-10-05T11:27:57Z 2026-10-08T11:27:57Z
INFO Manifests created in: install-dir, install-dir/manifests and install-dir/openshift
```

### 3.5. Master Düğümlerde Uygulama İş Yüklerinin Zamanlanmasının Kapatılması

Manifestolar oluşturulduktan sonra aşağıdaki dosya düzenlenir:

```bash
nano install-dir/manifests/cluster-scheduler-02-config.yml
```

Master düğümlere uygulama iş yüklerinin zamanlanmasını engellemek için `mastersSchedulable` değeri `false` yapılır:

```yaml
apiVersion: config.openshift.io/v1
kind: Scheduler
metadata:
  name: cluster
spec:
  mastersSchedulable: false
  policy:
    name: ""
  profileCustomizations:
    dynamicResourceAllocation: ""
status: {}
```

### 3.6. Ignition Dosyalarının Oluşturulması

Scheduler ayarı değiştirildikten sonra Ignition dosyaları oluşturulur:

```bash
./openshift-install create ignition-configs --dir=install-dir
```

Örnek çıktı:

```text
INFO Consuming Common Manifests from target directory
INFO Consuming Openshift Manifests from target directory
INFO Consuming Worker Machines from target directory
INFO Consuming Master Machines from target directory
INFO Consuming OpenShift Install (Manifests) from target directory
INFO Ignition-Configs created in: install-dir and install-dir/auth
```

### 3.7. Ignition Dosyalarının Nginx ile Yayınlanması

Oluşturulan Ignition dosyaları bastion üzerindeki Nginx yayın dizinine kopyalanır ve dosya izinleri ayarlanır:

```bash
cp ./install-dir/*.ign /usr/share/nginx/html/
chmod 644 /usr/share/nginx/html/*.ign
```

## 4. Bootstrap Makinesinin Kurulması

### 4.1. ISO Üzerinden Başlatma ve Ağ Yapılandırması

Bootstrap makinesi SCOS ISO üzerinden başlatılır.

Ağ ayarları aşağıdaki komutlarla yapılır:

```bash
nmcli connection add type ethernet ifname ens18 con-name ens18 ipv4.method manual ipv4.addresses 192.168.20.77/23 ipv4.gateway 192.168.20.1 ipv4.dns 192.168.20.2 ipv4.dns-search example.com connection.autoconnect yes
nmcli connection up ens18
hostnamectl set-hostname bootstrap
```

Bu örnekte kullanılan ağ bilgileri:

| Parametre | Değer |
|---|---|
| Ağ arayüzü | ens18 |
| IP adresi | 192.168.20.77/23 |
| Ağ geçidi | 192.168.20.1 |
| DNS sunucusu | 192.168.20.2 |
| DNS arama alanı | example.com |
| Hostname | bootstrap |

### 4.2. Ignition Dosyasının İndirilmesi ve Diske Kurulum

Bastion üzerinde yayınlanan bootstrap Ignition dosyası indirilir:

```bash
curl 192.168.20.32/bootstrap.ign > ignition.ign
```

Ardından kurulum hedef diske yapılır:

```bash
coreos-installer install --copy-network --ignition-file=ignition.ign /dev/sda --insecure-ignition
```

> **Disk seçimi:** Bu örnekte hedef disk `/dev/sda` olarak kullanılmıştır. Makinede birden fazla disk varsa kurulumun hangi diske yapılacağı dikkatle kontrol edilmelidir. Kurulum hedef diskin üzerine yazacaktır.

Kurulum tamamlandıktan sonra bootstrap makinesi kurulum yapılan diskten başlatılır.

## 5. Bootstrap Sürecinin İzlenmesi

Diskten başlatılan bootstrap makinesine, `install-config.yaml` içinde belirtilen public key’in karşılığı olan private key kullanılarak SSH bağlantısı kurulabilir.

Bastion üzerinden bootstrap’a bağlanılarak imaj indirme süreci, container’lar, servisler ve loglar kontrol edilebilir. HAProxy tarafında da bootstrap backend’lerinin UP/DOWN durumu izlenir.

Bootstrap hazırlanırken master ve worker makinelerinin ağ yapılandırması ve diske kurulum işlemleri yapılabilir. Bu makinelerin yeniden başlatılması için bootstrap API’sinin hazır olması beklenecektir.

### 5.1. Bastion Üzerinden Bootstrap Sürecinin Takibi

Bastion üzerinde aşağıdaki komut çalıştırılır:

```bash
./openshift-install wait-for bootstrap-complete --dir=./install-dir
```

Örnek başlangıç çıktısı:

```text
INFO Waiting up to 20m0s (until 3:04PM +03) for the Kubernetes API at https://api.okdtest.example.com:6443...
```

Bu çıktı, komutun Kubernetes API’nin erişilebilir olmasını beklediğini gösterir; bootstrap sürecinin tamamlandığı anlamına gelmez.

### 5.2. HAProxy Üzerinden Bootstrap Kontrolü

HAProxy sunucusunda backend durumları kontrol edilir:

```bash
echo "show stat" | sudo socat stdio /var/lib/haproxy/stats | grep -vE ',(FRONTEND|BACKEND),' | cut -d, -f1,2,18 | column -s, -t
```

Bootstrap üzerindeki ilgili servisler hazır olduğunda aşağıdaki iki kaydın `UP` olması beklenir:

```text
controlplaneapi          bootstrap  UP
controlplaneapiinternal  bootstrap  UP
```

### 5.3. Bootstrap Makinesine SSH Bağlantısı

Bastion üzerinden bağlantı kurulur:

```bash
ssh 192.168.20.77 -l core
```

İlk bağlantıda SSH sunucu anahtarına ilişkin doğrulama istemi görüntülenir. Bağlantı sonrasında bootstrap makinesinde aşağıdaki bilgilendirme bulunur:

```text
This is the bootstrap node; it will be destroyed when the master is fully up.

The primary services are release-image.service followed by bootkube.service and node-image-pull.service. To watch their status, run e.g.

  journalctl -b -f -u release-image.service -u bootkube.service -u node-image-pull.service
```

### 5.4. Bootstrap Servis Loglarının Takibi

Bootstrap üzerinde aşağıdaki komut çalıştırılır:

```bash
journalctl -b -f -u release-image.service -u bootkube.service -u node-image-pull.service
```

Bu komutla release imajı, bootstrap servisleri ve node imajının indirilmesiyle ilgili loglar takip edilir.

Paylaşılan kurulum çıktısından örnek satırlar:

```text
Currently on CoreOS version 10.0.20251103-0
Target node image is quay.io/okd/scos-content@sha256:2cc39d57e73c588d2c4fe4c70e2df406b29bd464e314c58acd077959a1c6d588
Not in live environment
Pulling quay.io/okd/scos-content@sha256:2cc39d57e73c588d2c4fe4c70e2df406b29bd464e314c58acd077959a1c6d588
Fetching ostree-unverified-registry:quay.io/okd/scos-content@sha256:2cc39d57e73c588d2c4fe4c70e2df406b29bd464e314c58acd077959a1c6d588
layers already present: 0; layers needed: 68 (975.3 MB)
Starting syncfs for system repo
```

### 5.5. Bastion Üzerinden Ek Erişim Kontrolleri

Bootstrap açıldıktan yaklaşık **5–10 dakika sonra**, bastion üzerinde aşağıdaki kontroller yapılır:

```bash
# HTTP 200 beklenir
curl -kv -m 10 https://192.168.20.77:22623/config/worker -o /dev/null

# Kubernetes API sürüm sorgusu
curl -k https://192.168.20.77:6443/version
```

### 5.6. Bootstrap Üzerinde Ek Kontroller

Bootstrap makinesinde aşağıdaki komutlar çalıştırılır:

```bash
# bootkube servisinin son 50 log satırı
sudo journalctl -b -u bootkube.service --no-pager | tail -50

# Container durumları
sudo crictl ps -a | head

# quay.io erişim kontrolü
curl -I https://quay.io
```

### 5.7. API Hazır Olma Kontrolü

Bastion üzerinde API’nin `/readyz` endpoint’i kontrol edilir:

```bash
curl -kv https://api.okdtest.example.com:6443/readyz
```

Beklenen yanıt:

```text
ok
```

## 6. Master ve Worker Makinelerinin Hazırlanması

Bootstrap’ın hazır olması beklenirken **3 master ve 3 worker’ın tamamında** ağ yapılandırması ve diske kurulum işlemleri gerçekleştirilir.

Her makine ISO üzerinden açıkken:

1. Makineye özel IP adresiyle ağ yapılandırması yapılır.
2. Makineye uygun hostname atanır.
3. Master makinelerde `master.ign`, worker makinelerde `worker.ign` bastion’dan indirilir.
4. İlgili Ignition dosyası kullanılarak hedef diske kurulum yapılır.

> **Disk, IP ve hostname kontrolü:** Komutlar her makineye uygulanmadan önce hedef disk, IP adresi ve hostname ilgili makineye göre kontrol edilmelidir. Özellikle birden fazla disk bulunan makinelerde doğru hedef diskin seçildiğinden emin olunmalıdır.

### 6.1. Master 1 İçin Örnek Yapılandırma

Aşağıdaki komutlar **okdtest-master-01**, IP adresi **192.168.20.71**, için uygulanır:

```bash
nmcli connection add type ethernet ifname ens18 con-name ens18 ipv4.method manual ipv4.addresses 192.168.20.71/23 ipv4.gateway 192.168.20.1 ipv4.dns 192.168.20.2 ipv4.dns-search example.com connection.autoconnect yes
nmcli connection up ens18
hostnamectl set-hostname okdtest-master-01
curl 192.168.20.32/master.ign > ignition.ign
coreos-installer install --copy-network --ignition-file=ignition.ign /dev/sda --insecure-ignition
```

### 6.2. Diğer Master ve Worker Makineleri

Aynı işlemler aşağıdaki makine bilgileri kullanılarak tüm master ve worker makinelerinde tekrarlanır:

| Makine | IP adresi | Hostname | İndirilecek Ignition dosyası |
|---|---|---|---|
| Master 1 | 192.168.20.71/23 | okdtest-master-01 | master.ign |
| Master 2 | 192.168.20.72/23 | okdtest-master-02 | master.ign |
| Master 3 | 192.168.20.73/23 | okdtest-master-03 | master.ign |
| Worker 1 | 192.168.20.74/23 | okdtest-worker-01 | worker.ign |
| Worker 2 | 192.168.20.75/23 | okdtest-worker-02 | worker.ign |
| Worker 3 | 192.168.20.76/23 | okdtest-worker-03 | worker.ign |

Worker makinelerde indirme adresi `192.168.20.32/worker.ign` olarak kullanılmalıdır.

> **Yeniden başlatma sırası:** Master ve worker makineleri, ağ ayarları ve diske kurulum işlemleri tamamlandıktan sonra hemen reboot edilmemelidir. Bootstrap API’sinin hazır olması beklenmelidir.

## 7. Bootstrap Hazır Olduktan Sonra Node’ların Başlatılması

### 7.1. Bootstrap API Kontrolü

Bastion üzerinde aşağıdaki komut çalıştırılır:

```bash
curl -k https://api.okdtest.example.com:6443
```

Örnek yanıt:

```json
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {},
  "status": "Failure",
  "message": "forbidden: User \"system:anonymous\" cannot get path \"/\"",
  "reason": "Forbidden",
  "details": {},
  "code": 403
}
```

Bu yanıt, API’ye ulaşıldığını ancak anonim kullanıcının `/` yoluna erişim izni bulunmadığını gösterir. Hazır olma durumunu kontrol etmek için `/readyz` endpoint’i sorgulanır:

```bash
curl -k https://api.okdtest.example.com:6443/readyz
```

Beklenen çıktı:

```text
ok
```

### 7.2. Master ve Worker Makinelerinin Diskten Başlatılması

Bootstrap API’sinin `/readyz` endpoint’i **`ok`** döndükten sonra, diske kurulumu tamamlanmış **3 master ve 3 worker** reboot edilerek kurulum yapılan diskten başlatılır.

Makinelerin boot sıralamasında kurulum diskinin ilk sırada olduğundan emin olunmalıdır.

Bu aşamadaki `/readyz` kontrolü API’nin hazır olduğunu gösterir. Bootstrap sürecinin tamamen bitmesi, `wait-for bootstrap-complete` komutuyla ayrıca takip edilir.

### 7.3. Node ve Cluster Kontrolleri

Başlatılan node’larda aşağıdakiler kontrol edilebilir:

- Kubelet servisinin durumu
- Çalışan prosesler arasında Podman prosesleri
- Container’lar ve indirilen imajlar

Bastion üzerinde `openshift-install wait-for bootstrap-complete` komutunun çıktısı takip edilir.

Ayrıca `install-dir/auth/` dizininde oluşturulan `kubeconfig` dosyası kullanılarak `oc` komutlarıyla Kubernetes objeleri kontrol edilebilir.

## 8. Worker CSR’larının Onaylanması

Worker node’lar başlatılırken sertifika imzalama istekleri, yani **CSR**’lar oluşur. Bu isteklerin onaylanması gerekir.

Aşağıdaki `oc` komutları, cluster’a erişim sağlayan `install-dir/auth/kubeconfig` kullanılarak çalıştırılmalıdır.

### 8.1. Otomatik CSR Onayı

Aşağıdaki döngü, bekleyen CSR’ları 10 saniyelik aralıklarla kontrol ederek topluca onaylar:

```bash
while : ; do
        [ -n "$(oc get csr | egrep -v 'Approved|REQUESTEDDURATION')" ] && \
                oc get csr -o go-template='{{range .items}}{{if not .status}}{{.metadata.name}}{{"\n"}}{{end}}{{end}}' | xargs --no-run-if-empty oc adm certificate approve
        sleep 10
done
```

Komut çıktısının koşul içinde değerlendirilmesi için komut ikamesi `"$(...)"` biçiminde kullanılmıştır.

Döngü `Ctrl+C` ile sonlandırılabilir.

### 8.2. Tek Seferlik Manuel CSR Onayı

Sürekli çalışan bir döngü yerine mevcut bekleyen CSR’ların tek seferde topluca onaylanması istenirse aşağıdaki komut çalıştırılır:

```bash
oc get csr -o go-template='{{range .items}}{{if not .status}}{{.metadata.name}}{{"\n"}}{{end}}{{end}}' | xargs --no-run-if-empty oc adm certificate approve
```

## 9. Master’ların Hazır Olmasının İzlenmesi

Master’lar hazır oldukça HAProxy üzerinde master backend’leri `UP` durumuna geçer. Paylaşılan kurulumda bu aşamada bootstrap backend’leri `DOWN` olarak görülmüştür.

HAProxy sunucusunda kontrol komutu:

```bash
echo "show stat" | sudo socat stdio /var/lib/haproxy/stats | grep -vE ',(FRONTEND|BACKEND),' | cut -d, -f1,2,18 | column -s, -t
```

Örnek çıktı:

```text
# pxname                 svname     status
controlplaneapi          bootstrap  DOWN
controlplaneapi          master0    UP
controlplaneapi          master1    UP
controlplaneapi          master2    UP
controlplaneapiinternal  bootstrap  DOWN
controlplaneapiinternal  master0    UP
controlplaneapiinternal  master1    UP
controlplaneapiinternal  master2    UP
secure                   worker0    DOWN
secure                   worker1    DOWN
secure                   worker2    DOWN
insecure                 worker0    DOWN
insecure                 worker1    DOWN
insecure                 worker2    DOWN
```

Bu çıktıda her iki control plane backend’inde master’lar `UP`, bootstrap ise `DOWN` durumundadır. Worker’ların ingress backend’leri bu aşamada hâlâ `DOWN` görünmektedir.

Bootstrap’ın yalnızca HAProxy’de `DOWN` görünmesi, makinenin kaldırılması için yeterli değildir. Installer’ın bootstrap kaynaklarının güvenle kaldırılabileceğini bildirmesi beklenir.

## 10. Bootstrap Sürecinin Tamamlanması ve Bootstrap’ın Kaldırılması

### 10.1. Installer Çıktısının Kontrolü

Bastion üzerinde bootstrap süreci aşağıdaki komutla takip edilir:

```bash
./openshift-install wait-for bootstrap-complete --dir=./install-dir
```

Paylaşılan kurulum çıktısı:

```text
INFO Waiting up to 20m0s (until 4:09PM +03) for the Kubernetes API at https://api.okdtest.example.com:6443...
INFO API v1.34.7 up
INFO Waiting up to 45m0s (until 4:34PM +03) for bootstrapping to complete...
E1008 15:49:46.418154  646409 reflector.go:205] "Failed to watch" err="Get \"https://api.okdtest.example.com:6443/api/v1/namespaces/kube-system/configmaps?allowWatchBookmarks=true&fieldSelector=metadata.name%3Dbootstrap&resourceVersion=24238&timeoutSeconds=301&watch=true\": context canceled" logger="UnhandledError" reflector="k8s.io/client-go/tools/watch/informerwatcher.go:162" type="*v1.ConfigMap"
INFO Waiting for the bootstrap etcd member to be removed...
INFO Bootstrap etcd member has been removed
INFO It is now safe to remove the bootstrap resources
INFO Time elapsed: 0s
```

Bootstrap kaynaklarının kaldırılması için aşağıdaki mesaj esas alınır:

```text
INFO It is now safe to remove the bootstrap resources
```

### 10.2. Bootstrap’ın Kapatılması ve HAProxy’den Çıkarılması

Installer güvenli kaldırma mesajını verdikten sonra işlemler şu sırayla gerçekleştirilir:

1. Bootstrap makinesi **poweroff** yapılır.
2. HAProxy yapılandırmasındaki `controlplaneapi` bölümünden `server bootstrap` satırı kaldırılır.
3. HAProxy yapılandırmasındaki `controlplaneapiinternal` bölümünden `server bootstrap` satırı kaldırılır.
4. Güncellenen HAProxy yapılandırması devreye alınır.

### 10.3. Bootstrap Kaldırıldıktan Sonraki HAProxy Kontrolü

HAProxy sunucusunda tekrar durum kontrolü yapılır:

```bash
echo "show stat" | sudo socat stdio /var/lib/haproxy/stats | grep -vE ',(FRONTEND|BACKEND),' | cut -d, -f1,2,18 | column -s, -t
```

Paylaşılan kurulumda bu işlemden sonra alınan çıktı:

```text
# pxname                 svname   status
controlplaneapi          master0  UP
controlplaneapi          master1  DOWN
controlplaneapi          master2  UP
controlplaneapiinternal  master0  UP
controlplaneapiinternal  master1  UP
controlplaneapiinternal  master2  UP
secure                   worker0  DOWN
secure                   worker1  DOWN
secure                   worker2  DOWN
insecure                 worker0  DOWN
insecure                 worker1  DOWN
insecure                 worker2  DOWN
```

Bootstrap satırları artık listede bulunmamaktadır. Bu anlık çıktıda `controlplaneapi/master1` ve worker ingress backend’leri `DOWN` durumundadır; çıktı cluster’ın tüm bileşenlerinin hazır olduğu son durumu temsil etmez.

## 11. Cluster Operator’ların Hazır Olmasının Beklenmesi

Bootstrap kaldırıldıktan sonra cluster bileşenlerinin kurulumu ve hazır olma süreci takip edilir.

Tüm Cluster Operator’ların aşağıdaki duruma gelmesi beklenir:

| Durum alanı | Beklenen değer |
|---|---|
| Available | True |
| Progressing | False |
| Degraded | False |

Node’lar `Ready` durumuna geldikten ve Cluster Operator’lar beklenen duruma ulaştıktan sonra genel kontroller yapılır.

## 12. Genel Kontroller ve Kurulumun Tamamlanması

Kontroller bastion üzerinden, `install-dir/auth/kubeconfig` dosyası kullanılarak gerçekleştirilir.

### 12.1. Cluster Operator Kontrolü

```bash
oc get co
```

Paylaşılan son kontrol çıktısı:

```text
NAME                                       VERSION              AVAILABLE   PROGRESSING   DEGRADED   SINCE   MESSAGE
authentication                             4.21.0-okd-scos.11   True        False         False      2m19s
baremetal                                  4.21.0-okd-scos.11   True        False         False      92m
cloud-controller-manager                   4.21.0-okd-scos.11   True        False         False      110m
cloud-credential                           4.21.0-okd-scos.11   True        False         False      129m
cluster-autoscaler                         4.21.0-okd-scos.11   True        False         False      92m
config-operator                            4.21.0-okd-scos.11   True        False         False      95m
console                                    4.21.0-okd-scos.11   True        False         False      2m19s
control-plane-machine-set                  4.21.0-okd-scos.11   True        False         False      89m
csi-snapshot-controller                    4.21.0-okd-scos.11   True        False         False      77m
dns                                        4.21.0-okd-scos.11   True        False         False      76m
etcd                                       4.21.0-okd-scos.11   True        False         False      89m
image-registry                             4.21.0-okd-scos.11   True        False         False      76m
ingress                                    4.21.0-okd-scos.11   True        False         False      53m
insights                                   4.21.0-okd-scos.11   True        False         False      88m
kube-apiserver                             4.21.0-okd-scos.11   True        False         False      81m
kube-controller-manager                    4.21.0-okd-scos.11   True        False         False      82m
kube-scheduler                             4.21.0-okd-scos.11   True        False         False      87m
kube-storage-version-migrator              4.21.0-okd-scos.11   True        False         False      76m
machine-api                                4.21.0-okd-scos.11   True        False         False      84m
machine-approver                           4.21.0-okd-scos.11   True        False         False      92m
machine-config                             4.21.0-okd-scos.11   True        False         False      94m
marketplace                                4.21.0-okd-scos.11   True        False         False      92m
monitoring                                 4.21.0-okd-scos.11   True        False         False      45m
network                                    4.21.0-okd-scos.11   True        False         False      91m
node-tuning                                4.21.0-okd-scos.11   True        False         False      58m
olm                                        4.21.0-okd-scos.11   True        False         False      76m
openshift-apiserver                        4.21.0-okd-scos.11   True        False         False      76m
openshift-controller-manager               4.21.0-okd-scos.11   True        False         False      77m
openshift-samples                          4.21.0-okd-scos.11   True        False         False      76m
operator-lifecycle-manager                 4.21.0-okd-scos.11   True        False         False      89m
operator-lifecycle-manager-catalog         4.21.0-okd-scos.11   True        False         False      93m
operator-lifecycle-manager-packageserver   4.21.0-okd-scos.11   True        False         False      76m
service-ca                                 4.21.0-okd-scos.11   True        False         False      95m
storage                                    4.21.0-okd-scos.11   True        False         False      89m
```

Bu çıktıda tüm Cluster Operator’lar **Available=True**, **Progressing=False** ve **Degraded=False** durumundadır.

### 12.2. Machine Config Pool Kontrolü

```bash
oc get mcp
```

Paylaşılan son kontrol çıktısı:

```text
NAME     CONFIG                                             UPDATED   UPDATING   DEGRADED   MACHINECOUNT   READYMACHINECOUNT   UPDATEDMACHINECOUNT   DEGRADEDMACHINECOUNT   AGE
master   rendered-master-26928fe5a2823d724d19ba872681077e   True      False      False      3              3                   3                     0                      96m
worker   rendered-worker-f3c9fa542e1fd50a57947984f7e77a27   True      False      False      3              3                   3                     0                      96m
```

Her iki havuz için beklenen durum sağlanmıştır:

| Kontrol | Master | Worker |
|---|---|---|
| Updated | True | True |
| Updating | False | False |
| Degraded | False | False |
| Toplam makine | 3 | 3 |
| Hazır makine | 3 | 3 |
| Güncel makine | 3 | 3 |
| Degraded makine | 0 | 0 |

### 12.3. Node Kontrolü

```bash
oc get no
```

Paylaşılan son kontrol çıktısı:

```text
NAME                              STATUS   ROLES                  AGE    VERSION
okdtest-master-01.okdtest.example.com     Ready    control-plane,master   110m   v1.34.6
okdtest-master-02.okdtest.example.com     Ready    control-plane,master   112m   v1.34.6
okdtest-master-03.okdtest.example.com     Ready    control-plane,master   112m   v1.34.6
okdtest-worker-01.okdtest.example.com     Ready    worker                 59m    v1.34.6
okdtest-worker-02.okdtest.example.com     Ready    worker                 62m    v1.34.6
okdtest-worker-03.okdtest.example.com     Ready    worker                 62m    v1.34.6
```

Bu çıktıda **3 master ve 3 worker’ın tamamı Ready** durumundadır. Master makineler `control-plane,master`, worker makineler ise `worker` rollerindedir.

### 12.4. Kurulumun Tamamlanması

Son kontrollerde aşağıdaki durumlar doğrulanmıştır:

- Tüm Cluster Operator’lar kullanılabilir durumdadır; ilerleyen veya degraded durumda operator bulunmamaktadır.
- Master ve worker Machine Config Pool’ları günceldir; her havuzdaki üç makine de hazırdır.
- Üç master ve üç worker’ın tamamı `Ready` durumundadır.
- Bootstrap süreci tamamlanmış, bootstrap makinesi kapatılmış ve load balancer yapılandırmasından çıkarılmıştır.

Bu kontrollerin ardından kurulum işlemi tamamlanır.

```yaml
# The node type - server or agent
rke2_type: "{{ 'server' if inventory_hostname in groups[rke2_servers_group_name] else 'agent' if inventory_hostname in groups[rke2_agents_group_name] }}"
```
```text
Bu değişken, RKE2'nin dinamik bir şekilde server ve agent nodelarına ayrılmasını sağlar. Control plane ve worker node rolleri, envanter dosyasındaki gruplar aracılığıyla kolayca atanır.
```



```yaml
# Deploy the control plane in HA mode
rke2_ha_mode: false
```
```text
Bu değişken, RKE2 cluster'ının control plane'nin (High Availability - HA) modunda çalıştırılıp çalıştırılmayacağını belirler.

- false: HA modu devre dışı. Tek bir control plane node'u (server) çalıştırılır.

- true: HA modu etkin. Birden fazla control plane node kurulur.

HA Modu Aktif Olduğunda Gereksinimler
- En Az 3 Control Plane Node
- Tüm control plane nodelar arasında trafiği dağıtacak bir Load Balancer (örneğin: HAProxy, kube-vip, Keepalived)
- Statik IP veya Virtual IP (VIP) (Kubernetes API serverına erişim IP adresi sabit olmalıdır. Bunun için "Keepalived" veya "kube-vip gibi" araçlar kullanılabilir )
```



```yaml
# Install and configure Keepalived on Server nodes
# Can be disabled if you are using pre-configured Load Balancer
rke2_ha_mode_keepalived: true
```
```text
Bu değişken, Keepalived aracını kullanarak RKE2 control plane (server) nodeşarı üzerinde bir Virtual IP (VIP) yapılandırmasını etkinleştirip etkinleştirmeyeceğinizi belirler. Keepalived, Kubernetes API sunucusuna sabit bir VIP sağlar ve bu VIP’nin birden fazla node arasında otomatik failover ile yönetilmesini mümkün kılar.

- true: Eğer bir düğüm arızalanırsa, VIP otomatik olarak başka bir düğüme taşınır (failover). Harici bir load balancer kullanmıyorsanız bu seçeneği kullanmanız önerilir.

- false: VIP atanmaz ve yönetilmez. Bunun yerine bir harici load balancer (örneğin, HAProxy veya NGINX) kullanmanız gerekir.

Keepalived Kullanımı İçin Gereksinimler
- rke2_ha_mode: true ayarlanmalıdır
- rke2_api_ip değişkeni ile sabit bir IP adresi belirlenmelidir
  rke2_api_ip: 192.168.1.100
```



```yaml
# Install and configure kube-vip LB and VIP for cluster
# rke2_ha_mode_keepalived needs to be false
rke2_ha_mode_kubevip: false
```
```text
true:
Bu değişken, Kubernetes cluster'ında kube-vip kullanılarak bir Load Balancer (LB) ve VIP (Virtual IP) yapılandırmasını etkinleştirip etkinleştirmeyeceğinizi belirler.
Kube-vip, Kubernetes API sunucusuna erişim için bir sabit IP sağlar.
Keepalived devre dışı olmalıdır (rke2_ha_mode_keepalived: false).

false:
VIP ve yük dengeleme için başka bir yöntem (örneğin, Keepalived veya harici bir load balancer) kullanılmalıdır.

Özellik	                  Kube-vip	                                 Keepalived
Amaç	                    VIP + yük dengeleme	                       Sadece VIP yönetimi
Failover	                Lider seçimi (Leader Election)	           VRRP ile failover
Kurulum Kolaylığı	        Daha basit	                               Orta düzey
Yük Dengeleme	            Evet	                                     Yok
```


```yaml
# Kubernetes API and RKE2 registration IP address. The default Address is the IPv4 of the Server/Master node.
# In HA mode choose a static IP which will be set as VIP in keepalived.
# Or if the keepalived is disabled, use IP address of your LB.
rke2_api_ip: "{{ hostvars[groups[rke2_servers_group_name].0]['ansible_default_ipv4']['address'] }}"
```
```text
Bu değişken, Kubernetes API serverı ve RKE2 nodelarının registration işlemleri için kullanılacak IP adresini belirler.

Default olarak, ilk Server/Master nodeunu IPv4 adresini kullanır.
HA Mode etkinleştirildiğinde, bu adres bir VIP (Virtual IP) veya bir Load Balancer (LB) IP adresi olmalıdır.

Default:
rke2_api_ip, envanter dosyasında tanımlanan ilk server/master nodeunun ansible_default_ipv4 adresini alır.

HA Mode Etkinse:
Keepalived veya Kube-vip kullanıyorsanız:

VIP adresi burada belirtilmelidir.
rke2_api_ip: "192.168.1.100"  # VIP Adresi

Harici bir Load Balancer kullanıyorsanız:
Load Balancer'ın statik IP adresi burada belirtilmelidir.
rke2_api_ip: "192.168.1.200"  # Load Balancer IP'si


Kodun İşleyişi

1. hostvars:
Tüm nodeların bilgilerini içeren bir sözlük (dictionary).

2. groups[rke2_servers_group_name]
Ansible envanterinde, rke2_servers_group_name grubunda tanımlı server/master nodeların listesi

3.0
Grubun ilk nodeunu seçer

4. ansible_default_ipv4.address
İlk nodeun varsayılan IPv4 adresini alır.
```



```yaml
# optional option for RKE2 Server to listen on a private IP address on port 9345
# rke2_api_private_ip:
```
```text
Bu değişken, RKE2 server nodelarının yalnızca özel bir IP adresinde dinlemesini sağlar. Özellikle HA modunda veya belirli bir ağ güvenliği gereksinimi olduğunda kullanılır. Default olarak, RKE2 API server'ı tüm ağ arayüzlerinden gelen trafiği dinler.

Port 9345, RKE2'nin node registration trafiği için kullandığı bir porttur.
RKE2 agent nodeları, bu porta bağlanarak server düğümüne kayıt olur ve cluster ile iletişim kurar.

Eğer server nodelarının yalnızca özel bir IP adresinden gelen trafiği kabul etmesini istiyorsanız bu değişkeni ayarlayın. Bu sayede Public ağ trafiği veya diğer ağlardan gelen istekler kabul edilmez.
```



```yaml
# optional option for kubevip IP subnet
# rke2_api_cidr: 24
```
```text
Bu değişken, Kube-vip kullanıldığında VIP (Virtual IP) için kullanılacak IP subnet değerini belirtir. VIP'nin hangi subnete atanacağını kontrol etmek için kullanılır.

Eğer VIP 192.168.1.100 olarak ayarlanmışsa, IP aralığı: 192.168.1.1 ile 192.168.1.254 arasında olacaktır. Amaç Ağ çakışmalarını önlemek ve ağ düzenini sağlamak.
```


```yaml
# optional option for kubevip
# rke2_interface: eth0
# optional option for IPv4/IPv6 addresses to advertise for node
# rke2_bind_address: "{{ hostvars[inventory_hostname]['ansible_' + rke2_interface]['ipv4']['address'] }}"
```
```text
rke2_interface: Kube-vip veya RKE2'nin hangi interface üzerinden çalışacağını belirler (örneğin, eth0 veya eth1).
rke2_bind_address: Seçilen interfaceden alınan IP adresini kullanarak nodeun hangi IP üzerinde dinleme yapacağını tanımlar.

Kullanım Amacı:
Birden fazla interface olan nodelarda hangi ağın kullanılacağını kontrol etmek.
Özel ağlar veya güvenlik gereksinimleri için IP adresini sınırlandırmak.
```



```yaml
# kubevip load balancer IP range
rke2_loadbalancer_ip_range: {}
#  range-global: 192.168.1.50-192.168.1.100
#  cidr-finance: 192.168.0.220/29,192.168.0.230/29
```
```text
rke2_loadbalancer_ip_range değişkeni, Kube-vip tarafından kullanılacak Load Balancer IP adres aralığını tanımlamak için kullanılır. Bu ayar, Kube-vip'in LoadBalancer türünde hizmetlere IP ataması yapmasına izin verir.

Eğer bu değişken boş bırakılırsa, varsayılan bir IP aralığı atanmaz. Bu durumda, Kube-vip bir IP adresi tahsis edemez ve manuel bir yapılandırma gerekebilir.

Belirli hizmetler veya ortamlar için ayrı IP aralıkları tanımlanabilir. Örneğin, finans sistemi için cidr-finance bloğu kullanılabilir.
```

```yaml
# Install kubevip cloud provider if rke2_ha_mode_kubevip is true
rke2_kubevip_cloud_provider_enable: true
```
```text
rke2_ha_mode_kubevip: true
Kube-vip'i bir cloud provider  gibi çalışacak şekilde yapılandırır.
Bu, Kubernetes API'yi kullanarak dış hizmetlere dinamik olarak IP adresleri tahsis etmesini sağlar.

HA modunda, Kube-vip, API sunucusu ve hizmetler için sanal IP adreslerini (VIP) yönetir.
Load Balancer ve yeniden yapılandırma sağlar.

Kubernetes'teki LoadBalancer türü hizmetler için gerekli olan dış IP atama işlemini otomatik hale getirir.


Kullanım Koşulu:
rke2_ha_mode_kubevip: true olması gerekir.
Kube-vip'in, VIP atamalarını ve cloud provider entegrasyonunu destekleyecek şekilde yapılandırılmış olması gerekir.

Kullanım Senaryosu:
Kube-vip'in, HA bir RKE2 kümesinde IP adreslerini otomatik olarak atamasını istiyorsanız bu değişken true olarak ayarlanır.
Eğer kendi network altyapınızı kullanıyorsanız ve otomatik IP atamalarını etkinleştirmek istiyorsanız bu değişken idealdir
```



```yaml
# Enable kube-vip to watch Services of type LoadBalancer
rke2_kubevip_svc_enable: true
```
```text
Kube-vip'in Kubernetes clusterında LoadBalancer türündeki Hizmetleri izlemesini (watch) etkinleştirmek için kullanılır.

true olarak ayarlandığında, Kube-vip Kubernetes'teki tüm LoadBalancer türündeki servisleri sürekli olarak izler ve bu serislere IP adresleri tahsis eder.

IP tahsisi, Kube-vip tarafından tanımlanan Load Balancer IP aralığına (örn. rke2_loadbalancer_ip_range) dayanır.

Eğer cloud providerın kendi Load Balancer'ını kullanmıyorsanız (örneğin, AWS ELB, Azure Load Balancer gibi), bu özellik Kube-vip'in kendi Load Balancer mekanizmasını devreye almasını sağlar.

Önemli Notlar:
Kullanılacak IP aralığı, rke2_loadbalancer_ip_range değişkeninde tanımlanmalıdır.
Kube-vip'in bu özelliği desteklemesi için gerekli kurulumların ve yapılandırmaların yapılmış olması gerekir.
```


```yaml
# Specify which image is used for kube-vip container
rke2_kubevip_image: ghcr.io/kube-vip/kube-vip:v0.6.4
```
```text
Değişken rke2_kubevip_image, Kube-vip'in hangi Docker image'ı kullanacağını belirtir. Bu, Kube-vip'in çalıştırılacağı sürümün ve hangi kaynaktan çekileceğinin belirlenmesini sağlar.

ghcr.io: Burada GitHub Container Registry kullanılmıştır
kube-vip/kube-vip: Image adı
v0.6.4: Versionu
```


```yaml
# Specify which image is used for kube-vip cloud provider container
rke2_kubevip_cloud_provider_image: ghcr.io/kube-vip/kube-vip-cloud-provider:v0.0.4
```
```text
Değişken rke2_kubevip_cloud_provider_image, Kube-vip'in cloud provider işlevi için kullanılacak olan Docker image'ını belirtir. Bu işlev, Kube-vip'in Kubernetes'teki cloud provider benzeri davranışlarını destekler.

Kullanım Senaryoları:
Cloud provider işlevini etkinleştirerek, HA modundaki Kubernetes clusterında IP yönetimini otomatikleştirir.
AWS, Azure veya GCP gibi providerlara bağlı kalmadan kendi altyapınızda Load Balancer yapılandırması sağlanır.

Bu değişken, Kube-vip'in cloud provider olarak davranmasını sağlamak için gerekli olan temel yapı taşlarından biridir. Doğru yapılandırıldığında, Kubernetes servisleri için dış IP yönetimini otomatikleştirir.
```


```yaml
# Enable kube-vip IPVS load balancer for control plane
rke2_kubevip_ipvs_lb_enable: false
# Enable layer 4 load balancing for control plane using IPVS kernel module
# Must use kube-vip version 0.4.0 or later
```
```text
true: IPVS (IP Virtual Server) Load Balancer'ını etkinleştirilir. Bu, Layer 4 (Transport Layer) Load Balancer sağlar.

Bu modül, gelen trafiği control olane nodeları arasında etkin bir şekilde dağıtır.
IPVS, Layer 4'te çalıştığı için düşük gecikmeli ve yüksek performanslı yük dengeleme sağlar.
IPVS Load Balancer özelliği için Kube-vip v0.4.0 veya daha yeni bir sürüm gereklidir. Eski sürümler bu özelliği desteklemez.

IPVS Load Balancer'ın çalışabilmesi için ipvs kernel modülünün yüklü ve aktif olması gerekir.

Aşağıdaki komutlarla kontrol edilebilir
lsmod | grep ip_vs
modprobe ip_vs

Kullanım Senaryoları:
Düşük gecikme süresi ve yüksek trafiğe dayanıklı bir Load Balancer gerekiyorsa bu özellik etkinleştirilir.
```



```yaml
rke2_kubevip_service_election_enable: true
# By default ARP mode provides a HA implementation of a VIP (your service IP address) which will receive traffic on the kube-vip leader.
# To circumvent this kube-vip has implemented a new function which is "leader election per service",
# instead of one node becoming the leader for all services an election is held across all kube-vip instances and the leader from that election becomes the holder of that service. Ultimately,
# this means that every service can end up on a different node when it is created in theory preventing a bottleneck in the initial deployment.
# minimum kube-vip version 0.5.0
```
```text
Değişken rke2_kubevip_service_election_enable, Kube-vip'in leader election per service mekanizmasını etkinleştirip etkinleştirmeyeceğini belirtir. Bu, Kube-vip'in Load Balancer işlevselliğinde daha fazla esneklik ve performans sağlar.

true: Service başına lider seçimi etkinleştirilir. Her service için farklı bir lider node seçilir.
false: Varsayılan ARP modu kullanılır. Tüm servisler için yalnızca bir lider node vardır.

Default ARP Modu:
Kube-vip, bir Virtual IP (VIP) adresini yalnızca tek bir lider node'ta barındırır.
Bu lider node, tüm service trafiğini yönetir.
Dezavantaj: Trafik, tek bir node'ta yoğunlaşabilir (bottleneck oluşabilir).

Service Başına Lider Seçimi:

Her service için ayrı bir lider seçimi yapılır.
Her service, farklı bir node'ta barındırılabilir.
Avantaj: Trafik yükü nodelar arasında dağıtılır, potansiyel tıkanıklıklar (bottlenecks) önlenir.

Performans İyileştirmesi:
Her service farklı bir node'ta çalışabileceği için, node başına düşen iş yükü azalır.
Bu, özellikle büyük ve yoğun trafiğe sahip Kubernetes clusterında önemlidir.
```



```yaml
# (Optional) A list of kube-vip flags
# All flags can be found here https://kube-vip.io/docs/installation/flags/
# rke2_kubevip_args: []
# - param: lb_enable
#   value: true
# - param: lb_port
#   value: 6443
```
```text

Değişken rke2_kubevip_args, Kube-vip için flags veya parametrelerin tanımlanmasına olanak tanır. Bu flagler, Kube-vip'in davranışını daha ayrıntılı bir şekilde özelleştirmek için kullanılır.

lb_enable: true: IPVS Load Balancerı etkinleştirir.
lb_port: 6443: Load Balancerın dinleyeceği portu belirler (RKE2 API serverı için default port 6443'tür).
```


```yaml
# Prometheus metrics port for kube-vip
rke2_kubevip_metrics_port: 2112
```
```text
Değişken rke2_kubevip_metrics_port, Kube-vip'in Prometheus metriklerini sağladığı portu belirtir. Bu port, Kube-vip tarafından üretilen metriklerin Prometheus gibi bir gözlemleme aracı tarafından çekilmesi (scraping) için kullanılır.

2112: Prometheus'un Kube-vip metriklerini toplamak için bağlanacağı port.
Bu, Kube-vip tarafından sağlanan default metrik portudur ve genellikle değiştirilmesine gerek yoktur. Ancak, port çakışması durumunda özelleştirilebilir.
```



```yaml
# Add additional SANs in k8s API TLS cert
rke2_additional_sans: []
```
```text
Değişken rke2_additional_sans, Kubernetes API serverına TLS sertifikasına ek Subject Alternative Names (SANs) eklemek için kullanılır. Bu SAN'lar, Kubernetes API'ye birden fazla farklı isim veya IP adresi üzerinden güvenli bir şekilde erişim sağlanmasını mümkün kılar.

Subject Alternative Name (SAN), bir TLS sertifikasının birden fazla domain name veya IP adresiyle çalışmasını sağlar.
Örneğin:
Farklı bir domain name adı (örneğin, api.example.com).
IP adresleri (örneğin, 192.168.1.100 veya 10.0.0.1).
DNS adları (örneğin, k8s-api.local).
```



```yaml
# Configure cluster domain
# rke2_cluster_domain: cluster.example.net
```
```text
Değişken rke2_cluster_domain, Kubernetes clusterında kullanılan DNS ad alanını (cluster domain) yapılandırmak için kullanılır. Bu alan, Kubernetes servislerinin DNS isimlendirme modelini belirler ve internal servis arasındaki iletişimde kullanılır.

Default değer genellikle cluster.local olarak gelir ve değiştirilebilir.

Birden fazla Kubernetes cluster aynı ağ içinde çalıştırıyorsanız, her cluster için farklı DNS ad alanları tanımlayarak çakışmaları önleyebilirsiniz.
```


```yaml
# API Server destination port
rke2_apiserver_dest_port: 6443
```
```text
6443: Kubernetes API serverının default portudur. API istekleri bu port üzerinden alınır ve işlenir.
```


```yaml
# Server nodes taints
rke2_server_node_taints: []
  # - 'CriticalAddonsOnly=true:NoExecute'
```
```text
Değişken rke2_server_node_taints, Kubernetes server nodes'a taint eklemek için kullanılır. Belirtilen taintler sayesinde, yalnızca uygun toleransa sahip pod'lar bu nodelara atanabilir.
```



```yaml
# Agent nodes taints
rke2_agent_node_taints: []
```
```text
Değişken rke2_agent_node_taints, Kubernetes agent nodes uygulanacak taint'leri belirtir.
```



```yaml
# Pre-shared secret token that other server or agent nodes will register with when connecting to the cluster
rke2_token: defaultSecret12345
```
```text
Değişken rke2_token, bir Kubernetes clusterına yeni server veya agent nodes katılabilmesi için kullanılan önceden paylaşılmış bir pre-shared secret token'dır. Bu belirteç, nodelar arasında güvenli bir iletişim sağlamak ve nodeların  clustera yetkilendirilmesini kontrol etmek için kullanılır.
```


```yaml
# RKE2 version
rke2_version: v1.25.3+rke2r1
```
```text
Değişken rke2_version, kurulacak veya kullanılacak RKE2'nin (Rancher Kubernetes Engine 2) spesifik sürümünü belirtir. Bu, clusterın hangi Kubernetes sürümünü ve buna bağlı RKE2 işlevselliklerini kullanacağını kontrol eder.
```


```yaml
# URL to RKE2 repository
rke2_channel_url: https://update.rke2.io/v1-release/channels
```
```text
Değişken rke2_channel_url, RKE2'nin hangi güncelleme kanallarını kullanacağını ve hangi sürümün yükleneceğini belirleyen RKE2 deposunun URL'sini tanımlar. Bu URL, RKE2'nin hangi sürüm kanalından paketleri ve güncellemeleri çekeceğini kontrol eder.
```


```yaml
# URL to RKE2 install bash script
# e.g. rancher chinase mirror http://rancher-mirror.rancher.cn/rke2/install.sh
rke2_install_bash_url: https://get.rke2.io
```
```text
Değişken rke2_install_bash_url, RKE2'nin kurulumunu başlatan bash script dosyasının URL'sini belirtir. Bu URL, RKE2'nin nasıl indirileceğini ve kurulacağını belirleyen temel bir yapılandırmadır.

Bu script, RKE2'nin uygun sürümünü indiren ve yükleme işlemini gerçekleştiren komut dosyasını içerir.
```

```yaml
# Local data directory for RKE2
rke2_data_path: /var/lib/rancher/rke2
```
```text
Değişken rke2_data_path, RKE2'nin yerel veri dizinini belirler. Bu dizin, RKE2'nin çalışması için gerekli olan tüm veri dosyalarını, yapılandırma dosyalarını ve geçici bileşenleri depolar. Default olarak /var/lib/rancher/rke2 olarak ayarlanmıştır.
```


```yaml
# Default URL to fetch artifacts
rke2_artifact_url: https://github.com/rancher/rke2/releases/download/
```
```text
Değişken rke2_artifact_url, RKE2 kurulum ve çalışma süreçlerinde kullanılacak olan artifaktların indirileceği default URL'yi belirtir. Bu URL, RKE2'nin hangi kaynaktan gerekli bileşenleri çekeceğini tanımlar.
```



```yaml
# Local path to store artifacts
rke2_artifact_path: /rke2/artifact
```
```text
RKE2 artifaktlarının yerel olarak saklanacağı özel bir dizin.
```



```yaml
# Airgap required artifacts
rke2_artifact:
  - sha256sum-{{ rke2_architecture }}.txt
  - rke2.linux-{{ rke2_architecture }}.tar.gz
  - rke2-images.linux-{{ rke2_architecture }}.tar.zst
```
```text
Değişken rke2_artifact, internet erişimi olmayan ortamlarda RKE2'nin çalışması için gerekli olan artifaktların listesini tanımlar. Bu artifaktlar, internet erişimi olmayan ortamlarda manuel olarak indirilmeli ve yerel bir depoya sağlanmalıdır.
sha256sum-{{ rke2_architecture }}.txt: ndirilen dosyaların bütünlüğünü doğrulamak için kullanılan SHA-256 özeti dosyasıdır.
rke2.linux-{{ rke2_architecture }}.tar.gz: RKE2'nin ana binari dosyasını içeren sıkıştırılmış arşiv dosyasıdır.
rke2-images.linux-{{ rke2_architecture }}.tar.zst: RKE2'nin çalışması için gereken Docker/Containerd imajlarını içeren sıkıştırılmış dosyadır.
```



```yaml
# Changes the deploy strategy to install based on local artifacts
rke2_airgap_mode: false
```
```text

Değişken rke2_airgap_mode, RKE2'nin kurulum stratejisini tanımlar ve yerel artifaktlar internetsiz kurulum üzerinden kurulumu etkinleştirip etkinleştirmeyeceğini belirler.

false: RKE2, internet üzerinden artifaktları indirerek kurulumu gerçekleştirir.

true: RKE2, internet bağlantısını kullanmadan yerel artifaktlardan kurulumu gerçekleştirir.
```


```yaml
# Airgap implementation type - download, copy or exists
# - 'download' will fetch the artifacts on each node,
# - 'copy' will transfer local files in 'rke2_artifact' to the nodes,
# - 'exists' assumes 'rke2_artifact' files are already stored in 'rke2_artifact_path'
rke2_airgap_implementation: download
```
```text
Değişken rke2_airgap_implementation, RKE2'nin air-gapped kurulumlarında artifaktların nasıl sağlanacağını tanımlar. Bu değişken, farklı ortam gereksinimlerine uygun olarak artifaktların sağlanma yöntemini seçer.

download: Artifaktlar her nodea internetten indirilecektir.

copy: Belirtilen artifakt dosyaları (rke2_artifact) ana bilgisayardan (örneğin, kontrol nodeu) diğer nodelara kopyalanır.

exists: Tüm nodelarda, gerekli artifaktların belirtilen dizinde (rke2_artifact_path) zaten mevcut olduğu varsayılır.

RKE2'nin air-gapped kurulumu, internet erişimi olmayan veya güvenlik nedenleriyle doğrudan dış kaynaklara erişimi kısıtlanmış sistemlerde Kubernetes clusterında kurulumunu ifade eder. Bu tür ortamlarda RKE2 ve Kubernetes bileşenlerinin kurulması için gereken tüm dosyalar ve bağımlılıklar önceden manuel olarak sağlanmalı ve yerel bir ortamda kullanılabilir hale getirilmelidir.
```



```yaml
# Local source path where artifacts are stored
rke2_airgap_copy_sourcepath: local_artifacts
```
```text
Değişken rke2_airgap_copy_sourcepath, RKE2'nin air-gapped kurulumlarında, artifaktların kaynağını belirten yerel dizin yolunu tanımlar. Bu, kontrol nodelarından diğer nodelara artifaktların kopyalanacağı dizindir.

Eğer özel bir dizin belirtilmemişse, yerel dizin genellikle şu şekilde olur:
rke2_airgap_copy_sourcepath: /var/lib/rancher/rke2/artifacts
```


```yaml
# Tarball images for additional components to be copied from rke2_airgap_copy_sourcepath to the nodes
# (File extensions in the list and on the real files must be retained)
rke2_airgap_copy_additional_tarballs: []
```
```text
Değişken rke2_airgap_copy_additional_tarballs, RKE2 air-gapped kurulumunda, rke2_airgap_copy_sourcepath dizininden nodelara kopyalanacak additional_tarball dosyalarının listesini tanımlar. Bu ek dosyalar, RKE2'nin çalışması için gerekli olmayan ancak belirli bileşenler veya eklentiler için gereken artifaktları içerebilir.

Örneğin, özel ağ eklentileri, uygulama imajları veya diğer Kubernetes bileşenleri için tarball dosyaları.
```



```yaml
# Destination for airgap additional images tarballs ( see https://docs.rke2.io/install/airgap/#tarball-method )
rke2_tarball_images_path: "{{ rke2_data_path }}/agent/images"
```
```text
Değişken rke2_tarball_images_path, air-gapped kurulumunda kullanılan additional_tarball imajlarının nodelarda nereye taşınacağını belirtir. Bu dizin, RKE2'nin çalışması için gereken tüm imajların bulunacağı hedef dizindir.

rke2_tarball_images_path, air-gapped ortamlarında RKE2 imajlarının nodelarda saklanacağı ve kullanılacağı dizini tanımlar
```



```yaml
# Architecture to be downloaded, currently there are releases for amd64 and s390x
rke2_architecture: amd64
```
```text
Değişken rke2_architecture, RKE2'nin hangi sistem mimarisi için indirilip kurulacağını belirler. Bu değişken, RKE2'nin çalıştırılacağı hedef donanım platformuna uygun artifaktların seçilmesini sağlar.
```


```yaml
# Destination directory for RKE2 installation script
rke2_install_script_dir: /var/tmp
```
```text
Değişken rke2_install_script_dir, RKE2'nin kurulum sürecinde kullanılan kurulum script'inin saklanacağı dizini belirtir. Bu dizin, RKE2 kurulumunun başlatılması için geçici olarak kullanılan script'in dosya yolunu tanımlar.
```


```yaml
# RKE2 channel
rke2_channel: stable
```
```text
Değişken rke2_channel, RKE2'nin hangi güncelleme kanalından yükleneceğini veya güncelleneceğini belirtir. Bu kanal, RKE2'nin hangi sürümünün kullanılacağını kontrol eden temel bir yapılandırma parametresidir.

stable: Kararlı sürüm kanalını temsil eder. Production ortamları için önerilen kanaldır.

latest: Yeni özellikleri test etmek veya geliştirme ortamlarında kullanmak için uygundur.

testing: Henüz kararlı olmayan, test aşamasındaki sürümleri içerir.
```



```yaml
# Do not deploy packaged components and delete any deployed components
# Valid items: rke2-canal, rke2-coredns, rke2-ingress-nginx, rke2-metrics-server
rke2_disable:
```
```text
Değişken rke2_disable, RKE2'nin default olarak birlikte gelen ve kurulacak bazı bileşenlerini devre dışı bırakır ve eğer kurulmuşlarsa kaldırır. Bu değişken, sisteminizin gereksinimlerine göre özelleştirilebilir bileşenler kullanmanıza olanak tanır.

Boş Liste ([]): Default olarak, RKE2'nin birlikte gelen hiçbir bileşeni devre dışı bırakılmaz veya kaldırılmaz.

Geçerli Bileşenler:
rke2-canal: RKE2 ile gelen varsayılan ağ eklentisi.
rke2-coredns: DNS yönetimi için kullanılan varsayılan DNS sunucusu.
rke2-ingress-nginx: ingress yönetimi için kullanılan varsayılan NGINX ingress controller.
rke2-metrics-server: Kubernetes kaynak kullanımını ölçmek için kullanılan varsayılan metrik sunucusu.


Özelleştirilmiş Ağ Çözümleri:
Eğer Calico veya Cilium gibi özel bir ağ eklentisi kullanmak istiyorsanız, rke2-canal devre dışı bırakılabilir.
Eğer Kubernetes cluster için kendi DNS çözümlemenizi sağlıyorsanız, rke2-coredns devre dışı bırakılabilir.
rke2-metrics-server devre dışı bırakılarak yerine Prometheus gibi başka bir metrik yönetimi çözümü kullanılabilir.
Eğer Traefik veya başka bir ingress çözümü kullanmayı planlıyorsanız, rke2-ingress-nginx devre dışı bırakılabilir.
```



```yaml
# Option to disable kube-proxy
disable_kube_proxy: false
```
```text
Değişken disable_kube_proxy, RKE2'nin kube-proxy bileşenini devre dışı bırakıp bırakmayacağını kontrol eder. kube-proxy, Kubernetes ağ modelinin bir parçası olarak hizmetler arasındaki trafiği yönlendiren bir ağ bileşenidi
```


```yaml
# Option to disable builtin cloud controller - mostly for onprem
rke2_disable_cloud_controller: false
```
```text
Değişken rke2_disable_cloud_controller, RKE2'nin dahili olarak sağladığı cloud controller bileşenini devre dışı bırakıp bırakmayacağını kontrol eder. Cloud controller, Kubernetes clusterında cloud provider entegrasyonunu yönetir ve genellikle bulut tabanlı kaynakların otomasyonunda kullanılır.

false: Dahili cloud controller etkinleştirilir.  (AWS, Azure, GCP vb.) çalışan cluster için kullanışlıdır.

true: Dahili cloud controller devre dışı bırakılır. on-prem Kubernetes kurulumlarında veya özel bir cloud controller çözümü kullandığınızda tercih edilir.
```



```yaml
# Cloud provider to use for the cluster (aws, azure, gce, openstack, vsphere, external)
# applicable only if rke2_disable_cloud_controller is true
rke2_cloud_provider_name: "external"
```
```text
Değişken rke2_cloud_provider_name, RKE2 clusterında kullanılacak cloud providerı belirtir. Bu ayar, rke2_disable_cloud_controller değişkeni true olarak ayarlandığında geçerli olur ve dahili cloud controller yerine harici bir sağlayıcının kullanılacağını tanımlar.

"external": Harici bir cloud providerı işaret eder. Bu ayar, kullanıcıların kendi cloud providerı çözümlerini veya özel entegrasyonlarını kullanmasına olanak tanır.

Eğer rke2_disable_cloud_controller: true olarak ayarlandıysa, Kubernetes clusterı için uygun bir cloud provider bu değişkenle belirtilmelidir.
```


```yaml
# Path to custom manifests deployed during the RKE2 installation
# It is possible to use Jinja2 templating in the manifests
rke2_custom_manifests:
```
```text
Değişken rke2_custom_manifests, RKE2 kurulumunda kullanılacak özel Kubernetes manifest dosyalarının saklandığı dosya yolunu belirtir. Bu manifest dosyaları, RKE2 kurulumuyla birlikte otomatik olarak uygulanır ve Kubernetes kaynaklarının (örneğin, DaemonSet, Deployment, Service) cluster üzerinde tanımlanmasını sağlar.

RKE2, Default olarak /var/lib/rancher/rke2/server/manifests dizinini kullanır:
```



```yaml
# Path to static pods deployed during the RKE2 installation
rke2_static_pods:
```
```text
Değişken rke2_static_pods, RKE2 kurulumunda Kubernetes static pod tanımlarının saklanacağı dosya yolunu belirtir. Static pods, doğrudan bir Kubernetes nodeunda tanımlanan ve kubelet tarafından yönetilen pod'lardır. Bunlar genellikle Kubernetes control plane bileşenleri veya özel bileşenler için kullanılır.
```


```yaml
# Configure custom Containerd Registry
rke2_custom_registry_mirrors: []
  # - name:
  #   endpoint: {}
#   rewrite: '"^rancher/(.*)": "mirrorproject/rancher-images/$1"'
```
```text
rke2_custom_registry_mirrors, RKE2'nin kullandığı containerd yapılandırmasını özelleştirmek ve konteyner registryı yönetmek için kullanılır. Bu değişken, RKE2'nin belirli registryı nasıl bağlanacağını ve gerektiğinde imaj isimlerini nasıl yeniden yazacağını kontrol eder.

RKE2'nin standart Docker Hub veya başka bir genel registryı erişmek yerine, belirttiğiniz yerel veya özel registryı kullanmasını sağlar

rke2_custom_registry_mirrors:
  - name: my.private.registry
    endpoint:
      - "https://my.private.registry:443"

İmajlar, özel registryı olan my.private.registry adresinden indirilir.
```



```yaml
# Configure custom Containerd Registry additional configuration
rke2_custom_registry_configs: []
#   - endpoint:
#     config:
```
```text
rke2_custom_registry_configs, RKE2'nin kullandığı containerd için özel registryi ek yapılandırmalarını tanımlamak için kullanılır. Bu, her registry için özelleştirilmiş ayarlar (örneğin, kimlik doğrulama, TLS sertifikaları, proxy ayarları) tanımlamanıza olanak tanır.

rke2_custom_registry_configs:
  - endpoint: my.private.registry
    config:
      auth:
        username: myuser
        password: mypassword

my.private.registry adresine bağlanmak için gerekli kullanıcı adı ve parola bilgileri sağlanır

rke2_custom_registry_configs:
  - endpoint: my.secure.registry
    config:
      tls:
        ca_file: /etc/rancher/registry/ca.crt

registryle güvenli bağlantı için özel bir CA sertifikası kullanılır.
```



```yaml
# Path to Container registry config file template
rke2_custom_registry_path: templates/registries.yaml.j2
```
```text
rke2_custom_registry_path, RKE2'nin kullanacağı containerd için registry yapılandırma dosyasının şablon yolunu belirtir. Bu şablon, konteyner registries ile ilgili ayarların dinamik olarak oluşturulmasını ve özelleştirilmesini sağlar.

Jinja2 şablonları kullanılarak, registry ayarları dinamik olarak oluşturulabilir. Bu, birden fazla ortamda farklı yapılandırmalar uygulamak için faydalıdır.
```



```yaml
# Path to RKE2 config file template
rke2_config: templates/config.yaml.j2
```
```text
rke2_config, RKE2'nin yapılandırma dosyasının şablonunun bulunduğu yolu belirtir. Bu şablon, RKE2'nin çalışması için gereken ayarların dinamik ve özelleştirilmiş şekilde oluşturulmasını sağlar. Jinja2 şablonlama desteği sayesinde, yapılandırma dosyası farklı ortamlara uyarlanabilir.
```


```yaml
# Etcd snapshot source directory
rke2_etcd_snapshot_source_dir: etcd_snapshots
```
```text
rke2_etcd_snapshot_source_dir, RKE2'nin etcd snapshot dosyalarını alacağı kaynak dizini belirtir. Bu dizin, etcd yedeklerinin saklandığı yerel dosya sistemindeki yolu ifade eder. Etcd snapshot'ları, Kubernetes kümesinin durumunu yedeklemek ve gerektiğinde geri yüklemek için kritik öneme sahiptir.
```


```yaml
# Etcd snapshot file name.
# When the file name is defined, the etcd will be restored on initial deployment Ansible run.
# The etcd will be restored only during the initial run, so even if you will leave the the file name specified,
# the etcd will remain untouched during the next runs.
# You can either use this or set options in `rke2_etcd_snapshot_s3_options`
rke2_etcd_snapshot_file:
```
```text
rke2_etcd_snapshot_file, RKE2 kurulumunda veya yeniden başlatmada kullanılacak etcd snapshot dosyasının adını tanımlar. Bu dosya, etcd veri tabanının yedekten geri yüklenmesi gerektiğinde kullanılır. Yalnızca ilk kurulum sırasında etcd geri yüklemesi yapılır ve sonraki çalıştırmalarda etcd etkilenmez.

rke2_etcd_snapshot_file: <snapshot_file_name>
<snapshot_file_name>: Bu dosya, rke2_etcd_snapshot_source_dir içinde bulunmalıdır.
```



```yaml
# Etcd snapshot location
rke2_etcd_snapshot_destination_dir: "{{ rke2_data_path }}/server/db/snapshots"
```
```text
rke2_etcd_snapshot_destination_dir, RKE2'nin etcd snapshot dosyalarını saklayacağı dizini belirtir. Bu dizin, etcd yedeklemelerinin hedef konumudur ve RKE2'nin otomatik veya manuel olarak aldığı snapshot dosyaları burada saklanır.
```



```yaml
# Etcd snapshot s3 options
# Set either all these values or `rke2_etcd_snapshot_file` and `rke2_etcd_snapshot_source_dir`
```
```text
rke2_etcd_snapshot_s3_options, RKE2'nin etcd snapshot dosyalarını bir S3 uyumlu depolama alanına yedeklemek veya geri yüklemek için kullanılan yapılandırma seçeneklerini içerir. Bu ayar, yedekleme ve geri yükleme süreçlerini yerel disk yerine S3 tabanlı bir çözüm üzerinden gerçekleştirmenizi sağlar.
```



```yaml
# rke2_etcd_snapshot_s3_options:
  # s3_endpoint: "" # required
  # access_key: "" # required
  # secret_key: "" # required
  # bucket: "" # required
  # snapshot_name: "" # required.
  # skip_ssl_verify: false # optional
  # endpoint_ca: "" # optional. Can skip if using defaults
  # region: "" # optional - defaults to us-east-1
  # folder: "" # optional - defaults to top level of bucket
# Override default containerd snapshotter
rke2_snapshotter: "{{ rke2_snapshooter }}"
rke2_snapshooter: overlayfs # legacy variable that only exists to keep backward compatibility with previous configurations
```
```text
rke2_etcd_snapshot_s3_options, RKE2'nin etcd snapshot dosyalarını S3 uyumlu bir depolama alanına kaydetmek veya bu alandan geri yüklemek için gereken yapılandırmayı tanımlar.
```

```yaml
# Deploy RKE2 with default CNI canal (should be a list)
rke2_cni: [canal]
```
```text
rke2_cni, RKE2 kurulumunda kullanılacak ağ eklentisini (Container Network Interface - CNI) belirler. Ağ eklentileri, Kubernetes pod'ları arasındaki ağ iletişimini sağlar ve RKE2'nin ağ modelini destekler.

Canal, RKE2'nin varsayılan ağ eklentisidir.

Diğer Olası Değerler: [calico], [cilium]

[] (Boş Liste): Eğer boş bırakılırsa, ağ eklentisi yüklenmez. Bu durumda özel bir CNI çözümü kurmanız gerekir.
```



```yaml
# Validate system configuration against the selected benchmark
# (Supported value is "cis-1.23" or eventually "cis-1.6" if you are running RKE2 prior 1.25)
rke2_cis_profile: ""
```
```text
rke2_cis_profile, RKE2'nin kurulum ve yapılandırmasının, CIS Kubernetes Benchmark standartlarına uygunluğunu kontrol etmek için kullanılan bir ayardır. CIS Benchmark, Kubernetes kümelerinin güvenliğini sağlamak için önerilen yapılandırma kurallarını içeren bir standarttır.

"" (Boş Değer): CIS Benchmark doğrulaması devre dışı bırakılır.

cis-1.6: Kubernetes 1.25'ten önceki RKE2 sürümleri için uygun.
```



```yaml
# Download Kubernetes config file to the Ansible controller
rke2_download_kubeconf: false
```
```text
rke2_download_kubeconf, Kubernetes cluster yönetiminde kullanılan kubeconfig dosyasını, RKE2 kurulumunda kullanılan Ansible control nodeuna indirip indirmeme seçeneğini belirler. Kubeconfig dosyası, Kubernetes API'sine erişim sağlamak için gerekli kimlik doğrulama bilgilerini içerir.

false: Kubeconfig dosyası Ansible control nodeuna indirilmez.

true: Kubeconfig dosyası, RKE2'nin kurulumunu yapan Ansible kontrol düğümüne indirilir. Ansible control nodeu Kubernetes API'sine doğrudan erişim sağlanabilir.
```


```yaml
# Name of the Kubernetes config file will be downloaded to the Ansible controller
rke2_download_kubeconf_file_name: rke2.yaml
```
```text
rke2_download_kubeconf_file_name, RKE2 kurulumunda Kubernetes clusterı için oluşturulan kubeconfig dosyasının Ansible kontrol dnodeuna indirilmesi durumunda hangi isimle kaydedileceğini belirler. Bu dosya, Kubernetes API'sine erişim için gerekli kimlik doğrulama bilgilerini içerir.
```


```yaml
# Destination directory where the Kubernetes config file will be downloaded to the Ansible controller
rke2_download_kubeconf_path: /tmp
```
```text
rke2_download_kubeconf_path, Kubernetes clusterı için oluşturulan kubeconfig dosyasının, Ansible kontrol nodeuna indirildiğinde hangi dizine kaydedileceğini belirler.
```

```yaml
# Default Ansible Inventory Group name for RKE2 cluster
rke2_cluster_group_name: k8s_cluster
```
```text
rke2_cluster_group_name, Ansible envanterinde (inventory) RKE2 clusterına atanacak default grup adını tanımlar. Bu grup adı, Ansible Playbook'larının hangi nodelarda çalışacağını belirlemek için kullanılır.
```



```yaml
# Default Ansible Inventory Group name for RKE2 Servers
rke2_servers_group_name: masters
```
```text
rke2_servers_group_name, Ansible envanterinde (inventory) RKE2 server (master) modelarını gruplamak için kullanılan varsayılan grup adını tanımlar. Server nodeları, Kubernetes control plane (API server, etcd, scheduler gibi) çalıştıran nodelardır ve cluster yönetiminden sorumludur.
```


```yaml
# Default Ansible Inventory Group name for RKE2 Agents
rke2_agents_group_name: workers
```
```text
rke2_agents_group_name, Ansible envanterinde (inventory) RKE2 agent (worker) nodelarını gruplamak için kullanılan varsayılan grup adını tanımlar
```



```yaml
# (Optional) A list of Kubernetes API server flags
# All flags can be found here https://kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver
# rke2_kube_apiserver_args: []
```
```text
rke2_kube_apiserver_args, Kubernetes API sunucusuna (kube-apiserver) özel flags eklemek için kullanılır. Bu flagler, API sunucusunun davranışını özelleştirmek ve cluster gereksinimlerine göre yapılandırmak için ayarlanabilir.

rke2_kube_apiserver_args:
  - "--enable-admission-plugins=NamespaceLifecycle,NodeRestriction,PodSecurityPolicy"
  - "--service-node-port-range=30000-32767"
```


```yaml
# (Optional) List of Node labels
# k8s_node_label: []
```
```text
k8s_node_label, Kubernetes nodelarına özel etiketler (labels) eklemek için kullanılan bir yapılandırma değişkenidir. 

k8s_node_label:
  - "environment=production"
  - "region=us-east"
```




```yaml
# (Optional) Additional RKE2 server configuration options
# You could find the flags at https://docs.rke2.io/reference/server_config
# rke2_server_options:
#   - "option: value"
#   - "node-ip: {{ rke2_bind_address }}"  # ex: (agent/networking) IPv4/IPv6 addresses to advertise for node
```
```text
rke2_server_options, RKE2 server nodeları için ek yapılandırma seçeneklerini belirtmek amacıyla kullanılan bir yapılandırma değişkenidir. Bu seçenekler, RKE2'nin server nodelarındaki davranışını özelleştirir ve control plane serverlarını yönetmek için kullanılır.

Güvenlik Ayarlarını Etkinleştirme:
rke2_server_options:
  - "protect-kernel-defaults: true"
  - "disable-selinux: false"

rke2_server_options:
  - "cluster-cidr: 10.42.0.0/16"
  - "service-cidr: 10.43.0.0/16"
```


```yaml
# (Optional) Additional RKE2 agent configuration options
# You could find the flags at https://docs.rke2.io/reference/linux_agent_config
# rke2_agent_options:
#   - "option: value"
#   - "node-ip: {{ rke2_bind_address }}"  # ex: (agent/networking) IPv4/IPv6 addresses to advertise for node
```
```text
rke2_agent_options, RKE2 agent (worker) nodeları için ek yapılandırma seçeneklerini tanımlamak amacıyla kullanılır. Bu seçenekler, worker nodelarının davranışını özelleştirmek ve belirli cluster gereksinimlerine uyum sağlamak için kullanılır.

rke2_agent_options:
  - "node-ip: 192.168.1.20"
  - "private-registry: /etc/rancher/registries.yaml"
  - "cgroup-driver: systemd"
  - "resolv-conf: /etc/resolv.custom.conf"
```


```yaml
# (Optional) Configure Proxy
# All flags can be found here https://docs.rke2.io/advanced#configuring-an-http-proxy
# rke2_environment_options: []
#   - "option=value"
#   - "HTTP_PROXY=http://your-proxy.example.com:8888"
```
```text
rke2_environment_options, RKE2 sunucuları ve agentları için HTTP/HTTPS proxy yapılandırmalarını ve çevresel değişkenleri belirlemek için kullanılan bir yapılandırma değişkenidir. Bu ayar, ağ kısıtlamaları altında çalışan Kubernetes clusterının gerekli proxy sunucularını kullanmasını sağlar.

rke2_environment_options:
  - "HTTP_PROXY=http://proxy.example.com:8888"
  - "HTTPS_PROXY=http://secure-proxy.example.com:8888"
  - "NO_PROXY=localhost,127.0.0.1,192.168.1.0/24"
  - "CUSTOM_ENV_VAR=my-value"
```


```yaml
# (Optional) Customize default kube-controller-manager arguments
# This functionality allows appending the argument if it is not present by default or replacing it if it already exists.
# rke2_kube_controller_manager_arg:
#   - "bind-address=0.0.0.0"
```
```text
rke2_kube_controller_manager_arg, RKE2 kurulumunda Kubernetes kube-controller-manager için arguments tanımlamak veya default argumentları özelleştirmek amacıyla kullanılır. Bu değişken, kube-controller-manager'ın davranışını değiştirmek ve cluster ihtiyaçlarına göre yapılandırmak için kullanılır.

rke2_kube_controller_manager_arg:
  - "node-monitor-grace-period=40s"

Bir nodeun erişilemez olarak işaretlenmeden önceki süre 40 saniye olarak ayarlanır.
```



```yaml
# (Optional) Customize default kube-scheduler arguments
# This functionality allows appending the argument if it is not present by default or replacing it if it already exists.
# rke2_kube_scheduler_arg:
#   - "bind-address=0.0.0.0"
```
```text
rke2_kube_scheduler_arg, RKE2 kurulumunda Kubernetes kube-scheduler için arguments tanımlamak veya default argumentları özelleştirmek amacıyla kullanılır. Bu değişken, kube-scheduler'ın davranışını değiştirerek pod'ların nodelara zamanlanmasıyla ilgili ayarları özelleştirmek için kullanılır.

rke2_kube_scheduler_arg:
  - "bind-address=0.0.0.0"
  - "leader-elect=true"
  - "leader-elect-lease-duration=30s"
  - "default-priority-class=system-cluster-critical"
```


```yaml
# (Optional) Configure nginx via HelmChartConfig: https://docs.rke2.io/networking/networking_services#nginx-ingress-controller
# rke2_ingress_nginx_values:
#   controller:
#     config:
#       use-forwarded-headers: "true"
rke2_ingress_nginx_values: {}
```
```text
rke2_ingress_nginx_values, RKE2'nin NGINX Ingress Controller'ı için HelmChartConfig kullanılarak özelleştirme yapılmasını sağlar. Bu değişken, NGINX Ingress Controller'ın çalışma davranışını ve özelliklerini yapılandırmak için kullanılır.
```



```yaml
# Cordon, drain the node which is being upgraded. Uncordon the node once the RKE2 upgraded
rke2_drain_node_during_upgrade: false
```
```text
rke2_drain_node_during_upgrade, bir RKE2 nodeun yükseltme işlemi sırasında cordon, drain, ve uncordon işlemlerinin otomatik olarak uygulanıp uygulanmayacağını kontrol eder

false: Yükseltme sırasında node üzerinde herhangi bir cordon veya drain işlemi yapılmaz. Pod'lar nodeta kalır ve yükseltme sırasında geçici bir kesinti yaşanabilir.

true: Yükseltme sırasında node üzerindeki pod'lar güvenli bir şekilde diğer nodelara taşınır. Yükseltme tamamlandıktan sonra node uncordon edilerek iş yükleri tekrar bu nodea dönebilir.
```


```yaml
# Wait for all pods to be ready after rke2-service restart during rolling restart.
rke2_wait_for_all_pods_to_be_ready: false
```
```text
rke2_wait_for_all_pods_to_be_ready, bir RKE2 servisi yeniden başlatılması sırasında veya rolling restart işlemlerinde, tüm pod'ların hazır (ready) duruma gelmesini bekleyip beklemeyeceğini kontrol eder. Bu değişken, özellikle Kubernetes clusterında hizmet sürekliliğini ve iş yükü stabilitesini sağlamak için kullanılır.

false: RKE2 servisi yeniden başlatıldıktan sonra pod'ların hazır duruma gelmesi beklenmez. Daha hızlı bir yeniden başlatma süreci sağlar ancak iş yüklerinin hazır duruma gelip gelmediğini kontrol etmez.

true: RKE2 hizmeti yeniden başlatıldıktan sonra tüm pod'ların ready duruma gelmesini bekler. Pod'ların duruma bağlı olarak ek süre gerektirir, ancak hizmet sürekliliği için daha güvenlidir.
```



```yaml
# Enable debug mode (rke2-service)
rke2_debug: false
```
```text
rke2_debug, RKE2 hizmeti için debug modunu etkinleştirip devre dışı bırakmayı kontrol eder. Debug modu, RKE2'nin çalışması sırasında daha ayrıntılı loglar oluşturarak sorun giderme ve analiz süreçlerinde yardımcı olur.
```


```yaml
# (Optional) Customize default kubelet arguments
# rke2_kubelet_arg:
#   - "--system-reserved=cpu=100m,memory=100Mi"
```
```text
rke2_kubelet_arg, RKE2 kurulumunda Kubernetes kubelet için özel arguments eklemek veya default ayarları özelleştirmek amacıyla kullanılır. Bu değişken, kubelet'in node üzerindeki kaynak yönetimi, performans ve ağ davranışlarını düzenlemek için yapılandırılmasını sağlar.


rke2_kubelet_arg:
  - "--system-reserved=cpu=100m,memory=100Mi"
  - "--eviction-hard=memory.available<500Mi,nodefs.available<10%"

--system-reserved, node üzerinde sistem işlemleri için ayrılacak CPU ve bellek kaynaklarını tanımlar.
--eviction-hard, node kaynaklarının yetersiz hale gelmesi durumunda pod'ların nasıl kaldırılacağını kontrol eder
--max-pods, bir nodeta çalıştırılabilecek maksimum pod sayısını belirler.
--node-ip, node için kullanılacak IP adresini tanımlar.
```


```yaml
# (Optional) Customize default kube-proxy arguments
# rke2_kube_proxy_arg:
#   - "proxy-mode=ipvs"
```
```text
rke2_kube_proxy_arg, Kubernetes kube-proxy için arguments tanımlamak veya default ayarları özelleştirmek için kullanılır.
```



```yaml
# The value for the node-name configuration item
rke2_node_name: "{{ inventory_hostname }}"
```
```text
rke2_node_name, Kubernetes clusterındaki bir nodeun adını belirlemek için kullanılır. Bu değer, nodeun cluster içindeki benzersiz kimliğini tanımlar ve Kubernetes API tarafından node u tanımlamak için kullanılır.

{{ inventory_hostname }}: Ansible envanterindeki (inventory) host adını kullanarak nodeun adı dinamik olarak atanır.
```


```yaml
# the network to use for Pods.. Set to '10.42.0.0/16' by default.
rke2_cluster_cidr:
  - 10.42.0.0/16
```
```text
rke2_cluster_cidr, Kubernetes clusterında pod'lar için kullanılacak ağ adres aralığını tanımlar. Bu ayar, Pod CIDR olarak bilinir ve Kubernetes'teki her pod'un benzersiz bir IP adresi almasını sağlar.
```



```yaml
# the network to use for ClusterIP Services. Set to '10.43.0.0/16' by default.
rke2_service_cidr:
  - 10.43.0.0/16
```
```text
rke2_service_cidr, Kubernetes clusterındaki ClusterIP Services için kullanılacak ağ adres aralığını tanımlar. Bu CIDR bloğu, Kubernetes servisleri için ayrılmıştır ve yalnızca servis IP adreslerini barındırır.
```



```yaml
# Enable SELinux for rke2
rke2_selinux: false
```
```text
rke2_selinux, RKE2 kurulumunda SELinux desteğini etkinleştirip devre dışı bırakmayı kontrol eden bir yapılandırma değişkenidir. SELinux (Security-Enhanced Linux), güvenliği artırmak için kullanılan bir çekirdek güvenlik modülüdür ve Kubernetes ile birlikte çalışabilir.
```
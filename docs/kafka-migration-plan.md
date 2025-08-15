# Kafka Upgrade & Migration Strategy (Case Bölüm-2)

Case'in ikinci bölümünde Kafka’nın büyük ölçekli, çok veri merkezli (multi-DC) özel bulut ortamında **sürüm yükseltmesi ve/veya yeni cluster taşımasının**; **kesintiyi minimize ederek**, **veri güvenliğini koruyarak** ve **performansı iyileştirerek** gerçekleştirilmesi istenmektedir.

Bu durumda:



---

## Özet & Akış

![Kafka Migration Flow](sandbox:/mnt/data/kafka_migration_flow.png)

1. **Current Cluster Analysis** → 2) **Backup & Validation** → 3) **New Cluster Setup** → 4) **Data Replication** → 5) **Traffic Switch** → 6) **Decommission**

**Seçimim:** Kritik iş yükleri ve versiyon sıçraması (ZK→KRaft veya majör 2.x→3.x) olduğu varsayımıyla **Parallel (Blue/Green) Migration + MirrorMaker2**. Bu yöntem **sıfıra yakın downtime** ve **güvenli geri dönüş** (rollback) sağlayacak.

---

## 1) Hedefler & Kısıtlar

* **Veri Güvenliği:** 0 veri kaybı (RPO≈0), ISR koruması, RF≥3, şifreleme ve yetkilendirme.
* **Kesinti:** RTO \~ dakikalarla sınırlı; kritik akışlar için Zero/Minimal Downtime.
* **Performans:** Throughput/latency ≥ mevcut; storage IOPS darboğazı kaldırılır.
* **Uyumluluk/Security:** TLS (mTLS), SASL/SCRAM veya Kerberos, kapsamlı ACL’ler, denetim izi.
* **Operasyonel Basitlik:** Otomasyon, gözlemlenebilirlik ve net rollback planı.

**Başarı Kriterleri:**

* Cutover sonrası **Under-Replicated/Offline Partition = 0**, **consumer lag** normal seviyede.
* Üretim uygulamalarında hata/timeout artışı **yok** (SLO ihlali yok).
* Eski küme güvenle kapatılmış ve yedeklenmiş.

---

## 2) Mevcut Durum Analizi (Discovery)

* **Envanter:** Broker sayısı, sürüm (2.x/3.x), Zookeeper mı KRaft mı, topic/partition/replication factor, retention/cleanup politikaları.
* **Kapasite & Performans:** CPU, RAM, disk (IOPS, throughput), network (10/25/40G), rack/topoloji.
* **Sağlık:** `UnderReplicatedPartitions`, `OfflinePartitions`, `RequestHandlerAvgIdlePercent`, GC, disk doluluk.
* **Kullanım:** Üretici/consumer tipleri, idempotent & transactional usage, exactly-once gereksinimi.
* **Güvenlik:** TLS/SASL, ACL kapsamı, secret yönetimi, denetim kayıtları.
* **Bağımlılıklar:** Schema Registry, Connect, ksqlDB, MirrorMaker, Cruise Control, Prometheus-JMX Exporter.

**Çıktı:** Risk listesi, kapasite boşlukları, iyileştirme alanları, önerilen hedef mimari.

---

## 3) Hedef Mimari Kararları

* **Sürüm & Denetleyici:** Kafka **3.x** + **KRaft** (Zookeeper bağımlılığını ortadan kaldırmak ve yönetimi sadeleştirmek). ZK’da kalınacaksa önce ZK’yı iyileştirip major upgrade yapılır, sonraki fazda KRaft’a geçilir.
* **Depolama:** NVMe/SSD, **XFS**, `noatime`; JBOD (broker-level replication ile) veya RAID10 – kurum standardına göre. **RF≥3**, `min.insync.replicas ≥ 2`.
* **Ağ & Topoloji:** Rack awareness (`broker.rack`), inter-DC replikasyon politikası, leader yerleşimini dengeleyen otomasyon (Cruise Control opsiyonel).
* **Gözlemlenebilirlik:** Prometheus + JMX Exporter + Grafana; Alertmanager ile sayfaya çıkarma.
* **Güvenlik:** mTLS, SASL/SCRAM, network policy (K8s), secret rotasyonu, ilke tabanlı erişim.
* **Operasyon:** IaC (Helm/Kustomize/Ansible), GitOps (Argo CD), rolling strategy & throttling.

---

## 4) Neden Parallel (Blue/Green) Migration?

| Kriter                | In-Place Rolling                                               | Parallel (Blue/Green)                                    |
| --------------------- | -------------------------------------------------------------- | -------------------------------------------------------- |
| **Downtime**          | Düşük ama riskli noktalar var (leader election, ISR daralması) | **Sıfıra yakın**, kontrollü cutover                      |
| **Risk/Backout**      | Geri dönüş zor (broker’lar tek tek ilerler)                    | **Kolay rollback** (DNS/CFG geri al, replication durdur) |
| **Büyük Sürüm/KRaft** | Karmaşık                                                       | **İzolasyon** ile daha güvenli                           |
| **Performans Etkisi** | Canlı küme üzerinde operasyonel yük                            | **Yeni donanımda hazırlık**, throttle ile koruma         |

> Kritik iş akışları ve majör mimari değişimlerde Parallel yaklaşımı **best-practice** kabul ediyorum.

---

## 5) Detaylı Plan (Runbook)

### 5.1 Current Cluster Analysis

* Topic/partition dağılımı, leader/ISR durumu, "hot broker" tespiti.
* Consumer lag ve throughput tabanı (baseline) çıkarılır.
* Üretici tarafında **idempotent** ve gerekliyse **transactional** mod doğrulanır.
* Konfig yedekleri (broker/topic/ACL/quotas) çekilir.

### 5.2 Backup & Validation

* **Metadata yedekleri:** ZK snapshot+txn log veya KRaft metadata shell.
* **Veri yedeği:** Depolama snapshotları (volume/disk seviyesinde) + seçili topic log arşivleri.
* **Geri Dönüş Testi:** Staging’de geri yükleme prova edildi, RTO/RPO doğrulandı.

### 5.3 New Cluster Setup

* Yeni küme (ayrı VPC/K8s namespace) **KRaft mode** ile kurulur.
* **Rack awareness** ve kapasite dengeleme yapılır, `auto.create.topics.enable=false`.
* **RF/ISR politikaları** ve **quotas** tanımlanır; TLS/SASL & ACL’ler kopyalanır.
* Observability stack (JMX Exporter, Prometheus, Grafana, Alertmanager) devrededir.

**Örnek broker ayarları (özet):**

```properties
process.roles=broker,controller
node.id=1
controller.quorum.voters=1@node1:9093,2@node2:9093,3@node3:9093
listeners=PLAINTEXT://:9092,CONTROLLER://:9093
inter.broker.listener.name=PLAINTEXT
log.dirs=/kafka-logs
num.partitions=8
min.insync.replicas=2
unclean.leader.election.enable=false
num.network.threads=8
num.io.threads=16
log.segment.bytes=1073741824
log.retention.hours=168
```

### 5.4 Data Replication (Old → New)

* **MirrorMaker 2** (MM2) ile seçili topic/ACL/offset replication başlatılır.
* Önce **düşük riskli** topic’ler, sonra kritik topic’ler.
* **Throttle** ayarlanır (IO/network koruması), cutover’a kadar sürekli senkron tutulur.

**Örnek MM2 konfig (kısa):**

```properties
clusters=OLD,NEW
OLD.bootstrap.servers=old1:9092,old2:9092
NEW.bootstrap.servers=new1:9092,new2:9092
OLD->NEW.enabled=true
replication.policy.class=org.apache.kafka.connect.mirror.DefaultReplicationPolicy
topics=.*
groups=.*
config.storage.topic=mm2-configs
offset.storage.topic=mm2-offsets
status.storage.topic=mm2-status
tasks.max=8
```

### 5.5 Traffic Switch (Cutover)

* **Değişiklik dondurma** (change freeze) başlatılır.
* **Kademeli geçiş:** Canary üretici/consumer → yüzde bazlı artış → tüm trafiğin yeni kümeye taşınması.
* Uygulama konfigleri (DSN/DNS/Service) GitOps ile güncellenir.
* Cutover boyunca: `UnderReplicatedPartitions=0`, consumer lag normal; hata göstergeleri yakından izlenir.

### 5.6 Old Cluster Decommission

* MM2 senkronizasyonu tamamlandıktan sonra eski kümede **yazma durdurulur**.
* Son bir tutarlılık kontrolü (offset/lag/topic sayısı) yapılır.
* Eski kümeden **aşamalı decommission**: topic/ACL temizliği, broker drain & kapatma.
* Son **arşiv yedekleri** alınıp dokümante edilir.

---

## 6) Monitoring & Alerting (Örnek Kurallar)

**Kritik metrikler:**

* `kafka_server_replicamanager_underreplicatedpartitions` > 0 → **Uyarı/Alarm**
* `kafka_controller_kafkacontroller_offlinepartitionscount` > 0 → **Alarm**
* Disk doluluk > %85 (node veya volume) → **Uyarı/Alarm**
* **Opsiyonel:** Consumer group lag, request timeout oranı, GC duraklamaları.

**Prometheus Alert örnekleri:**

```yaml
- alert: KafkaUnderReplicatedPartitions
  expr: kafka_server_replicamanager_underreplicatedpartitions > 0
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: Under-replicated partitions detected

- alert: KafkaOfflinePartitions
  expr: kafka_controller_kafkacontroller_offlinepartitionscount > 0
  for: 2m
  labels:
    severity: critical
  annotations:
    summary: Offline partitions present

- alert: NodeDiskHighUsage
  expr: (node_filesystem_size_bytes - node_filesystem_free_bytes) / node_filesystem_size_bytes > 0.85
  for: 10m
  labels:
    severity: warning
  annotations:
    summary: Disk usage over 85%
```

---

## 7) Performans İyileştirme Kontrol Listesi

* **Broker:** `num.network.threads`, `num.io.threads`, `socket.send/receive.buffer.bytes`, `compression.type=producer`, `message.max.bytes`/`replica.fetch.max.bytes` uyumu, `log.segment.bytes` & retention tuning.
* **OS/FS:** `nofile`/`nproc`, swappiness düşük, XFS, `noatime`, RAID/striping politikası.
* **Ağ:** Jumbo frame (ortam destekliyorsa), NIC queue tuning, IRQ dağılımı.
* **Uygulama:** Idempotent producer, acks=all, uygun batch.size/linger.ms; consumer max.poll.interval/fetch ayarları.

---

## 8) Güvenlik & Uyumluluk

* **Şifreleme:** mTLS; sertifika yaşam döngüsü otomasyonu.
* **Kimlik & Yetki:** SASL/SCRAM; en az ayrıcalık prensibi ile **ACL**.
* **Ağ İzolasyonu:** K8s NetworkPolicy, güvenlik grupları, iptables.
* **Gizli Bilgiler:** Secret/keystore yönetimi, per-topic/prod-consumer ayrımı.
* **Denetim:** Audit log, değişiklik kayıtları, konfig checksum.

---

## 9) Riskler, Önlemler & Rollback

**Başlıca Riskler**

* Replikasyon gecikmesi → Throttle/QA penceresi büyüt, canary.
* Hot partition/broker → Leader rebalance, partition re-assignment.
* Konfig uyuşmazlığı (ACL/Quota) → Göç öncesi tam envanter ve otomasyon.
* Cutover sonrası hata → Hızlı rollback.

**Rollback (Özet)**

1. DNS/Service yönlendirmesini **eski kümeye** geri al.
2. Yeni kümede üretici/consumer trafiğini durdur.
3. MM2’yi durdur; gerekiyorsa ters yönde replikasyon (NEW→OLD) başlat.
4. Hata kök neden analizi (RCA) ve düzeltme.

---

## 10) Zaman Planı (Örnek)

* **T–14 \~ T–7:** Discovery, benchmark, HLD/LLD, kapasite planı.
* **T–7 \~ T–3:** Yeni küme kurulumu, güvenlik & observability, MM2 dry-run.
* **T–2 \~ T–1:** Düşük kritik topic’lerin replikasyonu, canary geçişleri.
* **T (Cutover):** Değişiklik dondurma, tam geçiş, yoğun izleme.
* **T+1 \~ T+7:** Stabilizasyon, performans tuning, decommission başlangıcı.

**RACI (Özet):**

* Owner: Platform/DevOps
* Katkı: Uygulama ekipleri, Güvenlik, Ağ, Depolama
* Onay: Ürün sahipleri, Operasyon yöneticisi

---

## 11) Eğer Daha Fazla Kaynak/Zaman Olsaydı…

* Otomatik kapasite/yerleşim için **Cruise Control** entegrasyonu.
* **Tiered Storage** (S3/OB) ile daha uzun retention ve maliyet optimizasyonu.
* Cross-Region Active/Active veya DR için **Stretch Cluster** / **Cluster Linking**.
* Gelişmiş **chaos/DR tatbikatları** ve sürekli validasyon.

---

## Ek: Faydalı Komutlar (hızlı referans)

```bash
# Topic sağlığı
kafka-topics.sh --describe --bootstrap-server <broker:9092>

# Consumer lag
kafka-consumer-groups.sh --bootstrap-server <broker:9092> \
  --group <group> --describe

# Leader re-balance (örn. maintenance öncesi)
kafka-preferred-replica-election.sh --bootstrap-server <broker:9092>

# Partition reassignment (örnek JSON ile)
kafka-reassign-partitions.sh --bootstrap-server <broker:9092> \
  --reassignment-json-file reassignment.json --execute
```

---

**Not:** Diyagram ile bu runbook birlikte kullanılmalıdır. Excalidraw üzerinde şekil/nota ekleyerek ekip içi paylaşım için sunuma dönüştürülebilir.

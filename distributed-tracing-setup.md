# Distributed Tracing Setup Documentation

Bu dokümantasyon, dağıtık izleme (distributed tracing) sistemimizin kurulum ve yapılandırma detaylarını içermektedir.

## Genel Bakış

Sistemimiz aşağıdaki bileşenlerden oluşmaktadır:

1. **Grafana**: Görselleştirme ve dashboard'lar için
2. **Loki**: Log yönetimi için
3. **Tempo**: Distributed tracing için
4. **OpenTelemetry Collector**: Telemetri verilerinin toplanması ve yönlendirilmesi için

## Neden Tempo Kullanıyoruz?

**OpenTelemetry Collector**, telemetry (log, metric, trace) verilerini toplamak, işlemek ve farklı backend sistemlerine yönlendirmek için kullanılan bir "collector" ve yönlendiricidir. Collector'ın ana amacı, veriyi toplamak ve uygun bir backend'e (örneğin Tempo, Jaeger, Zipkin) göndermektir.
Collector, bazı exporter'lar (ör. file, kafka, debug) ile veriyi dosyaya veya başka sistemlere yazabilir. Ancak bu yöntem, büyük ölçekli, arama/sorgulama gerektiren production ortamları için uygun değildir. Collector'ın kendi başına bir "trace backend" (arama, indeksleme, uzun süreli saklama, hızlı sorgulama, Grafana entegrasyonu vb.) yeteneği yoktur.


**Tempo** ise, yüksek ölçeklenebilirlikte, maliyet-etkin ve bulut-native bir distributed tracing backend'idir. Trace verilerini uzun süreli saklama, hızlı arama ve Grafana ile entegre görselleştirme gibi yetenekler sunar. Tempo, OpenTelemetry Collector'dan gelen trace verilerini depolar ve sorgulanabilir hale getirir.

**Kısacası:**
OpenTelemetry Collector = Toplayıcı/Yönlendirici (kısıtlı/ham storage, production için uygun değil)
Tempo = Production-grade Trace Storage ve Query Backend

Bu nedenle, sadece Collector ile kalırsak trace verilerini saklayamayız ve analiz edemeyiz. Tempo gibi bir backend ile entegre ederek uçtan uca izleme ve analiz imkanı elde ediyoruz.

## Bileşenler ve Yapılandırmaları

### 1. OpenTelemetry Collector

OpenTelemetry Collector, farklı kaynaklardan gelen telemetri verilerini toplar ve işler.

#### Önemli Yapılandırmalar:
- **Desteklenen Protokoller**:
  - OTLP (gRPC): Port 4317
  - OTLP (HTTP): Port 4318
  - Jaeger (gRPC): Port 14250
  - Jaeger (HTTP): Port 14268
  - Zipkin: Port 9411

#### Resource İşlemleri:

#### Kaynak Limitleri:
```yaml
CPU: 1000m (limit), 500m (request)
Memory: 2Gi (limit), 1Gi (request)
```

### 2. Tempo (Tracing Backend)
### 2. Tempo (Tracing Backend)
Tempo, Grafana Labs tarafından geliştirilen, yüksek ölçeklenebilir bir distributed tracing backend'idir.

#### Önemli Yapılandırmalar:
- HTTP port: 3200
- Storage: Local filesystem
- Persistence: 10Gi (local-storage)
- Storage Path: /home/ubuntu/logs/tempo

#### Kaynak Limitleri:
```yaml
CPU: 500m (limit), 250m (request)
Memory: 1Gi (limit), 512Mi (request)
```

### 3. Loki (Log Aggregation)
### 3. Loki (Log Aggregation)
Loki, Grafana Labs'in geliştirdiği yüksek ölçeklenebilir log aggregation sistemidir.

#### Önemli Yapılandırmalar:
- Single Binary mode
- Authentication devre dışı
- Local filesystem storage
- Storage Path: /home/ubuntu/logs/loki
- Persistence: 8Gi (local-storage)

### 4. Grafana (Visualization)

Grafana, tüm telemetri verilerinin görselleştirilmesi için kullanılmaktadır.

#### Önemli Yapılandırmalar:
- Admin kullanıcı: admin/admin
- Root URL: http://localhost:3000
- Persistence: 5Gi (local-storage)
- Storage Path: /home/ubuntu/logs/grafana

#### Etkin Özellikler:
- Tempo Search
- Tempo Service Graph
- Tempo APM Table
- Trace to Metrics
- Trace to Logs

#### Yapılandırılmış Veri Kaynakları:
1. **Loki**:
   - URL: http://loki-gateway.observability.svc.cluster.local
   - Trace ID'lerini otomatik algılama ve linking

2. **Tempo**:
   - URL: http://tempo.observability.svc.cluster.local:3200
   - Node Graph özelliği aktif
   - Loki search entegrasyonu
   - Service map entegrasyonu
   - Trace to metrics ve logs entegrasyonları

## Kurulum Komutları

Sistemin kurulumu için aşağıdaki Helm komutları kullanılmalıdır:

```bash
# Namespace oluşturma
kubectl create namespace observability

# Grafana kurulumu
helm install grafana grafana/grafana -n observability -f grafana-values.yaml

# Loki kurulumu
helm install loki grafana/loki -n observability -f loki-values.yaml

# Tempo kurulumu
helm install tempo grafana/tempo -n observability -f tempo-values.yaml

# OpenTelemetry Collector kurulumu
helm install otel-collector open-telemetry/opentelemetry-collector -n observability -f otel-collector-values.yaml
```

## Veri Akışı

1. Uygulamalar telemetri verilerini OpenTelemetry Collector'a gönderir
2. Collector bu verileri işler ve etiketler
3. Trace'ler Tempo'ya, loglar Loki'ye yönlendirilir
4. Grafana bu verileri birleştirir ve görselleştirir

## Best Practices

1. Her servis için unique bir `service.name` belirleyin
2. Trace sampling oranını iş yüküne göre ayarlayın
3. Resource attributes'ları doğru şekilde yapılandırın
4. Log seviyelerini production ortamına uygun ayarlayın
5. Retention period'ları ihtiyaca göre yapılandırın

## Monitoring ve Troubleshooting

- Grafana UI: http://localhost:3000
- Tempo UI: http://tempo:3200
- Loki Query: http://loki-gateway/loki/api/v1/query

## Storage Considerations

Tüm bileşenler local-storage kullanacak şekilde yapılandırılmıştır:
- Grafana: 5Gi
- Loki: 8Gi
- Tempo: 10Gi

Üretim ortamında, bu storage yapılandırmalarının ihtiyaca göre güncellenmesi ve daha dayanıklı storage sınıflarının kullanılması önerilir.

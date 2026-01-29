# OpenTelemetry, Tempo, Prometheus ve Grafana ile Observability Stack

Bu proje, Kubernetes üzerinde OpenTelemetry Collector, Grafana Tempo, Prometheus ve Grafana kullanarak kapsamlı bir gözlemlenebilirlik (observability) stack'i kurmak için gerekli konfigürasyonları içerir.

## 📋 İçindekiler

- [Genel Bakış](#genel-bakış)
- [Bileşenler](#bileşenler)
- [Kurulum](#kurulum)
- [Konfigürasyon Dosyaları](#konfigürasyon-dosyaları)
- [Kullanım](#kullanım)
- [OpenTelemetry Auto-Instrumentation](#opentelemetry-auto-instrumentation)
- [Mimari](#mimari)

## 🔍 Genel Bakış

Bu stack aşağıdaki özellikleri sağlar:

- **Distributed Tracing**: OpenTelemetry ve Tempo ile dağıtık izleme
- **Metrics Collection**: Prometheus ile metrik toplama ve saklama
- **Visualization**: Grafana ile görselleştirme ve dashboard'lar
- **Auto-Instrumentation**: .NET uygulamaları için otomatik enstrümantasyon
- **Metrics Generation**: Trace'lerden otomatik metrik üretimi

## 🧩 Bileşenler

### 1. OpenTelemetry Collector
- OTLP protokolü ile trace ve metrik toplama (gRPC & HTTP)
- Tempo'ya trace gönderimi
- Prometheus'a metrik gönderimi
- Memory limiter ve batch processing

### 2. Grafana Tempo
- Distributed tracing backend
- Metrics generator ile trace'lerden metrik üretimi
- Service graphs ve span metrics desteği

### 3. Prometheus
- Metrik saklama ve sorgulama
- Remote write receiver özelliği aktif
- 10 gün veri saklama

### 4. Grafana
- Prometheus ve Tempo data source'ları yapılandırılmış
- Service map desteği
- Dashboard ve visualization

## 🚀 Kurulum

### Ön Gereksinimler

- Kubernetes cluster (v1.24+)
- kubectl CLI aracı
- Helm 3.x (Prometheus Operator için)

### Adım 1: Namespace Oluşturma

```bash
kubectl create namespace monitoring
```

### Adım 2: OpenTelemetry Operator Kurulumu

```bash
kubectl apply -f https://github.com/open-telemetry/opentelemetry-operator/releases/latest/download/opentelemetry-operator.yaml
```

### Adım 3: Prometheus Operator Kurulumu

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install prometheus-operator prometheus-community/kube-prometheus-stack \
  -f prometheus-operator.yaml \
  -n monitoring
```

### Adım 4: Prometheus Server Kurulumu

```bash
kubectl apply -f prometheus-server.yaml -n monitoring
```

### Adım 5: Tempo Kurulumu

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
helm install tempo grafana/tempo \
  -f tempo.yaml \
  -n monitoring
```

### Adım 6: OpenTelemetry Collector Kurulumu

```bash
kubectl apply -f otel-collector.yaml -n monitoring
```

### Adım 7: Grafana Kurulumu

```bash
helm install grafana grafana/grafana \
  -f grafana.yaml \
  -n monitoring
```

### Adım 8: OpenTelemetry Instrumentation Kurulumu

```bash
kubectl apply -f otel-instrumentation.yaml -n monitoring
```

## 📁 Konfigürasyon Dosyaları

### `otel-collector.yaml`
OpenTelemetry Collector konfigürasyonu:
- **Receivers**: OTLP (gRPC:4317, HTTP:4318)
- **Processors**: Memory limiter, batch processing
- **Exporters**: Debug, Tempo (OTLP), Prometheus Remote Write

### `tempo.yaml`
Tempo konfigürasyonu:
- HTTP port: 3200
- Metrics generator aktif
- Service graphs ve span metrics processors

### `prometheus-server.yaml`
Prometheus CRD konfigürasyonu:
- Remote write receiver aktif
- 30 saniye scrape interval
- 10 gün retention

### `grafana.yaml`
Grafana data source konfigürasyonu:
- Prometheus data source (default)
- Tempo data source (service map ile)

### `otel-instrumentation.yaml`
.NET auto-instrumentation konfigürasyonu:
- OTLP exporter (gRPC)
- Trace context propagation
- Parent-based trace ID ratio sampling
- ASP.NET Core ve HTTP client metrics

### `prometheus-operator.yaml`
Prometheus Operator Helm values:
- Sadece operator bileşeni aktif
- Diğer bileşenler (Alertmanager, Grafana, Node Exporter) devre dışı

### `annotations`
Pod annotation örneği:
- .NET auto-instrumentation için annotation

## 💡 Kullanım

### OpenTelemetry Auto-Instrumentation

.NET uygulamalarınıza otomatik instrumentation eklemek için, pod spec'inize şu annotation'ı ekleyin:

```yaml
spec:
  metadata:
    annotations:
      instrumentation.opentelemetry.io/inject-dotnet: 'monitoring/app-instrumentation'
```

Bu annotation eklendiğinde, OpenTelemetry Operator otomatik olarak:
- Init container ekler
- Sidecar container'ı yapılandırır
- Environment variable'ları inject eder
- Auto-instrumentation library'lerini yükler

### Grafana'ya Erişim

Grafana admin şifresini almak için:

```bash
kubectl get secret --namespace monitoring grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```

Port forwarding ile Grafana'ya erişim:

```bash
kubectl port-forward --namespace monitoring svc/grafana 3000:80
```

Tarayıcınızda `http://localhost:3000` adresine gidin.

### Prometheus'a Erişim

```bash
kubectl port-forward --namespace monitoring svc/prometheus-operated 9090:9090
```

### Tempo'ya Erişim

```bash
kubectl port-forward --namespace monitoring svc/tempo 3200:3200
```

## 🏗️ Mimari

```
┌─────────────────┐
│  .NET Apps      │
│  (Auto-Instr.)  │
└────────┬────────┘
         │ OTLP
         ↓
┌─────────────────────────┐
│ OpenTelemetry Collector │
│  - Receivers (OTLP)     │
│  - Processors           │
│  - Exporters            │
└──────┬──────────┬───────┘
       │          │
       │ Traces   │ Metrics
       ↓          ↓
┌──────────┐  ┌────────────┐
│  Tempo   │  │ Prometheus │
│          │←─│ (Remote    │
│ Metrics  │  │  Write)    │
│ Generator│  │            │
└─────┬────┘  └──────┬─────┘
      │              │
      └──────┬───────┘
             ↓
      ┌─────────────┐
      │   Grafana   │
      │  Dashboard  │
      └─────────────┘
```

### Veri Akışı

1. **Application → OpenTelemetry Collector**
   - .NET uygulamaları OTLP protokolü ile trace ve metric gönderir
   - HTTP (4318) veya gRPC (4317) portları kullanılır

2. **OpenTelemetry Collector → Tempo**
   - Trace verileri Tempo'ya OTLP formatında gönderilir
   - Tempo 4317 portundan alır

3. **OpenTelemetry Collector → Prometheus**
   - Metrikler Prometheus Remote Write API'sine gönderilir
   - Prometheus 9090 portundan alır

4. **Tempo → Prometheus**
   - Tempo, trace'lerden otomatik metrik üretir (span metrics, service graphs)
   - Bu metrikler Prometheus'a remote write ile gönderilir

5. **Grafana → Tempo & Prometheus**
   - Grafana, Tempo'dan trace verileri sorgular
   - Grafana, Prometheus'tan metrik verileri sorgular
   - Service map için her iki data source'u kullanır

## 🔧 Troubleshooting

### OpenTelemetry Collector Loglarını Kontrol Etme

```bash
kubectl logs -n monitoring deployment/trace-collector-collector -f
```

### Tempo Loglarını Kontrol Etme

```bash
kubectl logs -n monitoring deployment/tempo -f
```

### Prometheus Loglarını Kontrol Etme

```bash
kubectl logs -n monitoring statefulset/prometheus-prometheus -f
```

### Auto-Instrumentation Çalışmıyor mu?

Pod'un annotation'larını ve environment variable'larını kontrol edin:

```bash
kubectl describe pod <pod-name> -n <namespace>
```

Init container ve sidecar container'ların eklendiğinden emin olun.

## 📝 Notlar

- Tüm servisler `monitoring` namespace'inde çalışır
- Tempo metrics generator özelliği aktif, bu da trace'lerden otomatik metrik üretir
- Prometheus remote write receiver aktif, bu sayede OpenTelemetry Collector metrik gönderebilir
- .NET auto-instrumentation için OTLP gRPC protokolü kullanılır
- Sampling rate %100 olarak ayarlanmış (production'da düşürülmeli)

## 🤝 Katkıda Bulunma

Bu projeyi geliştirmek için pull request gönderebilir veya issue açabilirsiniz.

## 📄 Lisans

Bu proje MIT lisansı altında lisanslanmıştır.

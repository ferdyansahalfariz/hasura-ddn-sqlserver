# Send Observability from Sql Server connector to APM

Repo ini merupakan hasil eksplorasi penggunaan versi baru dari SQL Server Connector yaitu __v2.0.2__.

Adapun beberapa config yang perlu disesuaikan untuk mengirimkan observabilitynya antara lain :

### Pastikan penggunaan image SQL Server Connector sudah menggunakan versi terbaru di v2.0.2

image ini di define di __connector-metadata.yaml__

```
version: v1
packagingDefinition:
  type: PrebuiltDockerImage
  dockerImage: ghcr.io/hasura/ndc-sqlserver:v2.0.2
supportedEnvironmentVariables:
  - name: CONNECTION_URI
    description: The SQL server connection URI
    required: true
commands:
  update: hasura-ndc-sqlserver update
cliPlugin:
  type: null
  name: ndc-sqlserver
  version: v2.0.2
dockerComposeWatch:
  - path: ./
    action: sync+restart
    target: /etc/connector
```

dan di __Dockerfile.{connector_name}}__

```
FROM ghcr.io/hasura/ndc-sqlserver:v2.0.2
COPY ./ /etc/connector
```

### Gunakan image Open Telemetry contrib di __compose.yaml__

Pada __compose.yaml__ bagian Otel, Ubah image menggunakan image otel/opentelemetry-collector-contrib:0.104.0 yang sebelumnya secara default menggunakan open-telemetry biasa (otel/opentelemetry-collector:0.104.0).

```
  otel-collector:
    command:
      - --config=/etc/otel-collector-config.yaml
    environment:
      HASURA_DDN_PAT: ${HASURA_DDN_PAT}
    image: otel/opentelemetry-collector-contrib:0.104.0
    ports:
      - 4317:4317
      - 4318:4318
    volumes:
      - ./otel-collector-config.yaml:/etc/otel-collector-config.yaml
```

### Sesuaikan config OTEL di `otel-collector-config.yaml`

Pada config otel di `otel-collector-config.yaml` dapat menggunakan config di bawah yang isinya meng-export ke dua opsi:

1. Kirim langsung ke APM

Pada opsi ini, jika memungkinkan maka dapat langsung mengirim ke elastic APM menggunakan exporter `otlp/elastic` yang mengirim ke instance APM melalui port 8200

2. Kirim terlebih dahulu ke OTEL existing yang sudah terkoneksi ke APM

Pada opsi ini digunakan jika terdapat kendala saat mengirim langsung ke APM karena APM yang hanya dapat menerima jalur http sementara observability dari hasura di kirim menggunakan grpc. Jadi solusinya yakni otel sidecar ini akan mengirim ke otel existing yang sudah terkoneksi ke APM. 

Digunakan exporter otlphttp untuk mengirim ke OTLP pada jaur http yang secara default ada di port 4318.

```
exporters:
      otlp/elastic:
          endpoint: "http://10.100.14.212:8200"
          tls:
            insecure: true
      # otlphttp:
      #     endpoint: "http://OTLP:4318"
      #     tls:
      #       insecure: true
processors:
      batch: {}
      transform:
        error_mode: ignore
        trace_statements:
          - context: resource
            statements:
              - set(attributes["service.name"], "sqlserver-ferdy")
              - set(attributes["deployment.environment"], "development")
receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
          http:
            endpoint: 0.0.0.0:4318
service:
      pipelines:
        logs:
          exporters:
            - otlp/elastic
            # - otlphttp
          processors:
            - batch
            - transform
          receivers:
            - otlp
        metrics:
          exporters:
            - otlp/elastic
            # - otlphttp
          processors:
            - batch
            - transform
          receivers:
            - otlp
        traces:
          exporters:
            - otlp/elastic
            # - otlphttp
          processors:
            - batch
            - transform
          receivers:
            - otlp
```

# Hasil Testing

Setelah project ini dijalankan menggunakan `ddn run docker-start` dan dilakukan run query ke database sql server dari local seperti berikut, maka observability akan terkirim ke APM dan dapat langsung di lihat di kibana.

<img width="1919" height="965" alt="image" src="https://github.com/user-attachments/assets/17df6daf-3b02-4b46-b8c6-a82b11f81b4d" />

Kemudian jika di cek di kibana, bisa dilihat bahwa observability sudah masuk ke APM dengan pengaturan service.name dan deployment.environment yang sudah di setting pada `otel-collector-config.yaml`.

Terlihat ada dua jenis transaction yakni `/graphql` yang berisi hasil query yang dilakukan ke connector sql server dan `request` yang berisi history dari hit ke `/health`

<img width="1919" height="920" alt="image" src="https://github.com/user-attachments/assets/026d260e-00bd-4ea5-8540-1f57b075f995" />

Jika masuk ke `/graphql`, maka dapat ditemukan history dari query yang dilakukan seperti berikut yang menunjukan query ke `EmployeeTestingQuery` yang sudah sesuai dengan query yang sebelumnya baru saja di hit ke sql server connector. 

<img width="1919" height="928" alt="image" src="https://github.com/user-attachments/assets/99d71ab6-0076-4158-91a9-a1e655c2eaa7" />

<img width="1919" height="932" alt="image" src="https://github.com/user-attachments/assets/6dea2e42-078f-4cf0-ad81-2ff3fa594d04" />


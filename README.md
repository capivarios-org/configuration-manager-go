# 🐹 Configuration Manager SDK for Go

SDK cliente oficial em **Go (Golang 1.22+)** para integração de altíssima performance (< 0.5ms) com o [Configuration Manager Engine](https://github.com/capivarios-org/configuration-manager-engine), parte do ecossistema [Capivarios](https://github.com/capivarios-org).

---

## ⚡ Principais Recursos

- **Estratégia Dual TTL:**
  - `SoftTTL`: Revalidação assíncrona em goroutine em segundo plano (*stale-while-revalidate*) para zero impacto na latência principal.
  - `HardTTL`: Invalidação síncrona compulsória após longos períodos de inatividade (*cold routes*).
- **Smart Polling (ETag / HTTP 304):** Concorrência segura e verificação ultra rápida com hashes em memória.
- **Server-Sent Events (SSE):** Streaming push em tempo real com reconexão automática e sincronização contínua.
- **Thread-Safe & Zero Allocation:** Cache local otimizado com `sync.RWMutex` / `sync.Map`.

---

## 📦 Instalação

```bash
go get github.com/capivarios-org/configuration-manager-go
```

---

## 🚀 Exemplo de Uso

```go
package main

import (
	"context"
	"fmt"
	"log"
	"time"

	configmanager "github.com/capivarios-org/configuration-manager-go"
)

func main() {
	client, err := configmanager.NewClient(configmanager.Options{
		BaseURL:       "http://localhost:8080",
		APIKey:        "cm_live_123456",
		TransportMode: configmanager.TransportSmartPolling,
		ConsumeMode:   configmanager.ConsumeFolder,
		TargetKey:     "checkout-service",
		SoftTTL:       30 * time.Second, // Revalidação assíncrona
		HardTTL:       1 * time.Hour,    // Invalidação síncrona
	})
	if err != nil {
		log.Fatalf("Falha ao inicializar SDK: %v", err)
	}
	defer client.Close()

	ctx := context.Background()

	// Checar feature flag
	enabled, err := client.IsEnabled(ctx, "new-payment-gateway")
	if err != nil {
		log.Printf("Erro na verificação: %v", err)
	}

	if enabled {
		fmt.Println("Novo gateway de pagamento ativo!")
	}
}
```

---

## 📄 Licença

Distribuído sob a licença open source. Veja [LICENSE](LICENSE) para mais detalhes.

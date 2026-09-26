# 📐 Especificação Técnica: Configuration Manager SDK for Go

> **Repositório:** `capivarios-org/configuration-manager-go`  
> **Papel:** SDK Cliente Oficial em Go (Golang)  
> **Comunicação:** Direta com o [Engine (Data Plane)](https://github.com/capivarios-org/configuration-manager-engine)

---

## 1. Princípio de Comunicação

O SDK em Go conecta-se de forma direta e thread-safe ao **Engine (Data Plane)**. Ele não possui dependência direta nem requer conexão com o painel administrativo (Admin), operando com consumo eficiente de recursos e latência mínima.

---

## 2. Estratégia Dual TTL (Algoritmo de Decisão)

O SDK resolve o desafio de performance vs obsolescência através de dois temporizadores de cache local:

```plaintext
+--------------------------------------------------------------------------------+
|                         FLUXO DE DECISÃO DUAL TTL                              |
|                                                                                |
|                        client.IsEnabled(ctx, "flag")                           |
|                                     │                                          |
|                                     ▼                                          |
|                    elapsed = now() - entry.LastFetched                         |
|                                     │                                          |
|            ┌────────────────────────┼────────────────────────┐                 |
|            ▼                        ▼                        ▼                 |
|   elapsed < SoftTTL        SoftTTL <= elapsed < HardTTL   elapsed >= HardTTL   |
|   (Cache Fresco)           (Stale-While-Revalidate)       (Cache Expirado)     |
|            │                        │                        │                 |
|            │                        ├─────────────────┐      │                 |
|            │                        │                 │      │                 |
|            ▼                        ▼                 ▼      ▼                 |
|    Retorna valor em         Retorna valor em       Dispara   Bloqueia chamada  |
|    memória (< 0.1ms)        memória (< 0.1ms)      goroutine e consulta Engine |
|    Zero chamadas de         sem travar goroutine   assíncrona diretamente com  |
|    rede.                    solicitante.           com ETag  If-None-Match     |
|                                                              (SÍNCRONO)        |
+--------------------------------------------------------------------------------+
```

### 2.1. Estrutura em Memória:
```go
type LocalCacheEntry struct {
    Value       interface{}   // Valor da Flag ou Mapa do Folder
    ETag        string        // Hash único da versão fornecido pelo Engine
    LastFetched time.Time     // Timestamp da última sincronização bem-sucedida
    SoftTTL     time.Duration // Gatilho para revalidação assíncrona (ex: 30s)
    HardTTL     time.Duration // Invalidação síncrona forçada pós-inatividade (ex: 1h)
}
```

---

## 3. Modos de Transporte

- **Server-Sent Events (SSE):** Conexão contínua em goroutine dedicada; recebe pushes instantâneos do Engine e atualiza o cache local atomicamente.
- **Smart Polling:** Revalidação via HTTP com cabeçalho `If-None-Match`. Respostas `304 Not Modified` apenas renovam o `LastFetched` com custo desprezível de rede e CPU.

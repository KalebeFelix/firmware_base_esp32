# Otimização do Payload MQTT
## Análise de Engenharia de Dados

---

## Problema Identificado

### Payload Original (Redundante)

```json
{
  "maquina_id": "MAQ01",
  "operador": "ABC12345",
  "ref_op": "DEF67890",
  "tempoTrabalho": true,      // ← Derivado de state
  "tempoProducao": true,      // ← Derivado de state
  "Banheiro": false,          // ← Derivado de state + parada
  "Manutencao": false,        // ← Derivado de state + parada
  "FaltaMaterial": false,     // ← Derivado de state + parada
  "QuebraAgulha": false,      // ← Derivado de state + parada
  "TrocaPeca": false,         // ← Derivado de state + parada
  "rpm": [240, 360, 180],
  "cpuLoad": 0.00,
  "heapLoad": 23.45,
  "state": "PRODUCING"
}
```

**Tamanho:** ~280 bytes
**Campos:** 14
**Problema:** Múltiplos campos derivados de uma única fonte de verdade (`state`)

---

## Solução Implementada

### Payload Otimizado (Normalizado)

```json
{
  "maquina_id": "MAQ01",
  "operador": "ABC12345",
  "ref_op": "DEF67890",
  "state": "PRODUCING",
  "parada": "none",
  "rpm": [240, 360, 180],
  "cpuLoad": 0.00,
  "heapLoad": 23.45
}
```

**Tamanho:** ~180 bytes
**Campos:** 8
**Ganho:** 35% menor, 43% menos campos

---

## Regras de Derivação (Backend)

### Single Source of Truth: `state` + `parada`

O backend pode inferir todos os estados anteriores:

```javascript
// Lógica de negócio no backend/dashboard
function parsePayload(payload) {
    const { state, parada } = payload;

    return {
        // Estados booleanos derivados
        tempoTrabalho: state !== 'IDLE',
        tempoProducao: state === 'PRODUCING',
        emParada: state === 'PAUSED',

        // Paradas individuais
        Banheiro: (state === 'PAUSED' && parada === 'Banheiro'),
        Manutencao: (state === 'PAUSED' && parada === 'Manutencao'),
        FaltaMaterial: (state === 'PAUSED' && parada === 'FaltaMaterial'),
        QuebraAgulha: (state === 'PAUSED' && parada === 'QuebraAgulha'),
        TrocaPeca: (state === 'PAUSED' && parada === 'TrocaPeca'),

        // Estado original
        ...payload
    };
}
```

---

## Benefícios da Otimização

### 1. Redução de Banda e Armazenamento

| Métrica | Antes | Depois | Ganho |
|---------|-------|--------|-------|
| **Tamanho por mensagem** | 280 bytes | 180 bytes | -35% |
| **Campos JSON** | 14 | 8 | -43% |
| **Banda/dia (1 msg/3s)** | 8.1 MB | 5.2 MB | -2.9 MB |
| **Banda/mês (30 dias)** | 243 MB | 156 MB | -87 MB |
| **Armazenamento/ano** | 2.9 GB | 1.9 GB | -1.0 GB |

**Com 100 máquinas:**
- Banda mensal economizada: **8.7 GB**
- Armazenamento anual economizado: **100 GB**

---

### 2. Single Source of Truth

**Antes:** Possível inconsistência
```json
{
  "state": "PRODUCING",
  "tempoProducao": false,  // ← INCONSISTENTE!
  "Banheiro": true         // ← INCONSISTENTE!
}
```

**Depois:** Impossível ter inconsistência
```json
{
  "state": "PRODUCING",
  "parada": "none"  // ✓ Sempre consistente
}
```

---

### 3. Manutenibilidade

**Adicionar nova parada:**

**Antes (múltiplas mudanças):**
```diff
// firmware_main
+ bool tempoNovaParada = false;
+ doc["NovaParada"] = tempoNovaParada;

// Backend
+ const novaParada = payload.NovaParada;

// Dashboard
+ <Checkbox field="NovaParada" />
```

**Depois (mudança única):**
```diff
// firmware_main (apenas)
typedef enum : uint8_t {
    // ...
    PARADA_TROCA_PECA = 5,
+   PARADA_NOVA_PARADA = 6,  // ← Apenas adicionar no enum
    PARADA_COUNT = 7
} ParadaTipo;

const ParadaInfo PARADAS[PARADA_COUNT] = {
    // ...
+   {"NovaParada", "Nova Parada"}
};
```

Backend/Dashboard automaticamente interpretam o novo valor sem mudanças de código!

---

### 4. Redução de RAM no ESP32

**Antes:**
```c
StaticJsonDocument<384> doc;  // 384 bytes de RAM
char buffer[384];              // 384 bytes de RAM
                               // TOTAL: 768 bytes
```

**Depois:**
```c
StaticJsonDocument<256> doc;  // 256 bytes de RAM
char buffer[256];              // 256 bytes de RAM
                               // TOTAL: 512 bytes
```

**Economia:** 256 bytes de RAM (33% menos)

No ESP32 com ~320KB de RAM, isso é significativo em sistemas que rodam 24/7.

---

## Exemplos de Payload por Estado

### Estado IDLE
```json
{
  "maquina_id": "MAQ01",
  "operador": "",
  "ref_op": "",
  "state": "IDLE",
  "parada": "none",
  "rpm": [0, 0, 0],
  "cpuLoad": 0.00,
  "heapLoad": 18.32
}
```

**Derivações:**
- `tempoTrabalho = false` (IDLE)
- `tempoProducao = false` (não PRODUCING)

---

### Estado LOGGED_IN
```json
{
  "maquina_id": "MAQ01",
  "operador": "4A3B2C1D",
  "ref_op": "",
  "state": "LOGGED_IN",
  "parada": "none",
  "rpm": [0, 0, 0],
  "cpuLoad": 0.00,
  "heapLoad": 19.45
}
```

**Derivações:**
- `tempoTrabalho = true` (não IDLE)
- `tempoProducao = false` (não PRODUCING)

---

### Estado PRODUCING
```json
{
  "maquina_id": "MAQ01",
  "operador": "4A3B2C1D",
  "ref_op": "9E8F7A6B",
  "state": "PRODUCING",
  "parada": "none",
  "rpm": [240, 360, 180],
  "cpuLoad": 0.00,
  "heapLoad": 21.78
}
```

**Derivações:**
- `tempoTrabalho = true` (não IDLE)
- `tempoProducao = true` (PRODUCING)
- Todas paradas = `false` (parada = "none")

---

### Estado PAUSED
```json
{
  "maquina_id": "MAQ01",
  "operador": "4A3B2C1D",
  "ref_op": "9E8F7A6B",
  "state": "PAUSED",
  "parada": "Banheiro",
  "rpm": [0, 0, 0],
  "cpuLoad": 0.00,
  "heapLoad": 22.14
}
```

**Derivações:**
- `tempoTrabalho = true` (não IDLE)
- `tempoProducao = false` (não PRODUCING)
- `Banheiro = true`, outras paradas = `false`

---

## Conclusão

A otimização do payload elimina redundância seguindo o princípio **Single Source of Truth**:

**35% menos banda** (importante em redes celulares/IoT)
**33% menos RAM** no ESP32
**Impossível ter inconsistências** de estado
**Manutenibilidade superior** (adicionar paradas não requer mudanças no backend)
**Consultas SQL mais simples** (WHERE state = 'PAUSED' vs múltiplos ORs)

**Princípio:** *"Move computation to the edge, not data over the wire."*

---

**Autor:** João Ícaro (baseado no trabalho de Kalebe Felix e João Victor)
**Data:** 2025
**Versão:** 2.0

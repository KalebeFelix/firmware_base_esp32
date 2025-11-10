# 📟 Sistema de Monitoramento de Produção - ESP32

Este projeto implementa um firmware completo para ESP32, voltado ao monitoramento de produção industrial com controle de operador, registro de paradas, velocidade de máquina (RPM), uso de recursos e comunicação via MQTT.

## 🚀 Funcionalidades Principais

- 📲 **Login por RFID**: operadores se identificam ao aproximar seu cartão RFID.
- 🏷️ **Referência por RFID**: segunda leitura de RFID define a referência de produção..
- ⌨️ **Controle por Teclado**: sistema dinâmico para registrar paradas de produção.
- 📉 **Cálculo de RPM**: utiliza um sensor de pulso (encoder) conectado à máquina.
- 🧠 **Medição de uso da CPU e RAM** do ESP32.
- 📡 **Envio de dados para broker MQTT** em tempo real, a cada 5 segundos.
- 📺 **Interface LCD 20x4**: exibe status do operador, referência, paradas e RPM.

---

## ⚙️ Componentes Utilizados

- ESP32
- Leitor RFID MFRC522
- Display LCD I2C 20x4
- Keypad matricial 4x3
- Sensor de pulso (encoder óptico ou magnético)
- Broker MQTT (Mosquitto)
- Conexão Wi-Fi

---

## 🧩 Lógica de Funcionamento

### ✅ 1. Login e Logout via RFID
- 🔐 Login do Operador
  - Primeiro RFID: registra como operador
  - Ativa `tempoTrabalho = true`
  - Exibe mensagem `"Login: <UID>"` e `"Insira Referencia..."`

- 🏷️ Definição de Referência
  - Segundo RFID: registra como referência de produção
  - Ativa `tempoProducao = true`
  - Exibe `"Login: [UID]"` e `"REF: [UID]"`

- 🔓 Logout Inteligente
  - Mesmo RFID do operador: logout completo
     - Limpa operador, referência e paradas
     - Volta ao estado inicial
  - Mesmo RFID da referência: finaliza apenas a referência
     - Mantém operador logado
     - Exibe mensagem `"REF finalizada"` e `"Insira Referencia..."`
       
- ⚠️ Validações
    - Tentativa de usar terceiro RFID com referência ativa é rejeitada
    - Mensagem: `"REF ja ativa! Use mesmo REF p/sair"` 
  
---

### 🔢 2. Sistema Dinâmico de Paradas via Teclado

#### 🎯 Como Funciona
 - Pressione `*`: ativa modo parada → "Insira parada..."
 - Digite código (1-5): `"Insira parada: X"`
 - Pressione `#`: confirma parada → `"Parada: [Nome]"`
 - Pressione `*` novamente: desativa parada → linha fica vazia

  | Código | Parada             |
  |--------|--------------------|
  | 1      | Banheiro           |
  | 2      | Manutenção         |
  | 3      | Falta de Material  |
  | 4      | Quebra de Agulha   |
  | 5      | Troca de Peça      |

- Somente uma parada pode estar ativa por vez.
- Ao ser ativado qualquer tipo de parada, o tempo produção vira false.

#### 🔄 Comportamento das Paradas
 - Ativação: pausa produção (`tempoProducao = false`)

 - Desativação: retoma produção (`tempoProducao = true`)

 - Uma parada por vez: sistema impede múltiplas paradas simultâneas

---

### ⚙️ 3. Medição de RPM
- Um **sensor de pulso (encoder)** gera interrupções no pino `ENCODER_PIN`.
- A cada segundo, o sistema calcula:
  
RPM = (pulsos * 60) 
- Mostra o valor no LCD: `RPM: xx.xx`

---

### 📊 4. Monitoramento de Sistema
- A cada 3 segundos, o sistema envia um JSON via MQTT com os seguintes dados:

```json
{
  "maquina_id": "MAQ01",
  "operador": "UID_RFID_OPERADOR",
  "ref_op": "UID_RFID_REFERENCIA",
  "tempoTrabalho": true,
  "tempoProducao": true,
  "Banheiro": false,
  "Manutencao": false,
  "FaltaMaterial": false,
  "QuebraAgulha": false,
  "TrocaPeca": false,
  "rpm": 240.00,
  "cpuLoad": 52.5,
  "heapLoad": 34.7
}

# 📟 Sistema de Monitoramento de Produção - ESP32

Este projeto implementa um firmware completo para ESP32, voltado ao monitoramento de produção industrial com controle de operador, registro de paradas, velocidade de máquina (RPM), uso de recursos e comunicação via MQTT.

## 🚀 Funcionalidades Principais

- 📲 **Login por RFID**: operadores se identificam ao aproximar seu cartão RFID.
- ⌨️ **Controle por Teclado**: permite inserir códigos de referência de produção e registrar paradas.
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
- Broker MQTT (ex: Mosquitto)
- Conexão Wi-Fi

---

## 🧩 Lógica de Funcionamento

### ✅ 1. Login e Logout via RFID
- Quando **nenhum operador está logado**, ao passar um cartão RFID válido, o sistema realiza o **login**.
  - Ativa `tempoMaquina = true`
  - Exibe mensagem `"Login: <UID>"` e `"Insira Referencia..."`

- Se o mesmo operador passar o cartão novamente, realiza o **logout**:
  - Zera operador, referência, produção e paradas.
  - Exibe `"Logout Realizado"` e `"Efetue o Login!"`

- Se outro operador tentar logar enquanto já há um ativo, a tentativa é rejeitada.

---

### 🔢 2. Inserção de Referência e Paradas (via Teclado)

- Após login, o operador pode digitar **um número no teclado e pressionar `#`**.
- O valor digitado determina a ação:

#### ℹ️ Referência de Produção
- **Se o número for `15` ou maior**, é considerado uma **referência válida**:
  - Se nenhuma referência ativa, ativa `tempoProducao = true`
  - Exibe `"REF: <referencia>"`
- Se já houver uma referência ativa, ignora e exibe `"REF já ativa!"`.

#### ⛔ Paradas de Produção
- **Se o número for de `1` a `5`**, ativa/desativa os tipos de parada:
  | Código | Parada             |
  |--------|--------------------|
  | 1      | Banheiro           |
  | 2      | Manutenção         |
  | 3      | Falta de Material  |
  | 4      | Quebra de Agulha   |
  | 5      | Troca de Peça      |

- Somente uma parada pode estar ativa por vez.

#### 🔄 Finalizar Referência
- Para **finalizar a produção atual**:
  - Pressione `*` e depois `#`
  - Isso **limpa a referência** e desativa `tempoProducao`
  - Exibe `"REF finalizada"`

---

### ⚙️ 3. Medição de RPM
- Um **sensor de pulso (encoder)** gera interrupções no pino `ENCODER_PIN`.
- A cada segundo, o sistema calcula:
  
RPM = (pulsos * 60) / PULSES_PER_REV
- Mostra o valor no LCD: `RPM: xx.xx`

---

### 📊 4. Monitoramento de Sistema
- A cada 5 segundos, o sistema envia um JSON via MQTT com os seguintes dados:

```json
{
"maquina_id": "MAQ01",
"operador": "UID_RFID",
"referencia": "123",
"tempoMaquina": true,
"tempoProducao": true,
"tempoBanheiro": false,
"tempoManutencao": false,
"tempoFaltaMaterial": false,
"tempoQuebraAgulha": false,
"tempoTrocaPeca": false,
"rpm": 240.00,
"cpuLoad": 52.5,
"heapLoad": 34.7
}

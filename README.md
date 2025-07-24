# 🧵 Projeto ESP32 - Firmware base

## 🚀 Funcionalidades

- 📶 Conexão Wi-Fi com autenticação
- 🔐 Login e logout de operadores via **cartão RFID**
- 🔢 Registro de **referências de produção** e tipos de paradas via teclado
- 🧭 Cálculo e exibição do **RPM da máquina**
- 📟 Feedback ao operador via **LCD I2C**
- 📡 Envio periódico de dados em formato **JSON** para o broker MQTT
- 🧠 Monitoramento do uso da **CPU** e da **memória heap**

---

## 🔌 Componentes Utilizados

| Componente            | Função                                     |
|-----------------------|--------------------------------------------|
| ESP32                 | Microcontrolador principal                 |
| MFRC522 RFID          | Leitura de cartões RFID                    |
| Encoder               | Cálculo do RPM da máquina                  |
| Teclado Matricial 4x3 | Entrada de dados (referência ou parada)   |
| Display LCD I2C 20x4  | Exibição de informações ao operador        |
| Broker MQTT           | Recebimento remoto dos dados               |

---

## 🧠 Lógica de Funcionamento

### 👤 Identificação do Operador (RFID)

- Ao aproximar um cartão RFID válido, o sistema realiza o login.
- O LCD exibe o ID do operador e solicita uma referência.
- Um segundo toque no mesmo cartão efetua logout e reseta estados.

### ⌨️ Registro via Teclado

- Números ≥ 15 → Referência de produção (ativa `tempoProducao`)
- Números de 1 a 6 → Alternam os seguintes estados de parada:
  - `1` → Banheiro
  - `2` → Manutenção
  - `3` → Falta de material
  - `4` → Quebra de agulha
  - `5` → Troca de peça

> Só uma parada pode estar ativa por vez.

### 🔄 Encoder e RPM

- A cada segundo, os pulsos são contados para calcular o **RPM**.

### 📡 Envio MQTT

- A cada 3 segundos, um JSON com todos os dados é publicado em:
Tópico: esp32
---

## 🧪 Exemplo de Payload JSON

```json
{
"maquina_id": "MAQ01",
"operador": "AB12CD34",
"referencia": "REF_123",
"tempoMaquina": true,
"tempoProducao": true,
"tempoBanheiro": false,
"tempoManutencao": false,
"tempoFaltaMaterial": false,
"tempoQuebraAgulha": false,
"tempoTrocaPeca": false,
"rpm": 1508.50,
"cpuLoad": 52.4,
"heapLoad": 35.3
}

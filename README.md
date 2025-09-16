# Monitor de Energia - Projeto Final - EmbarcaTech 2025 2° Fase

Autores: **Gabriel Mattano, Luana Vacari e Marina Donaire**  
Curso: Residência Tecnológica em Sistemas Embarcados  
Instituição: EmbarcaTech – HBr  
Campinas, Julho/Agosto/Setembro de 2025

---

## ✨ Visão geral

Projeto de **monitoramento de energia em tempo real** usando **Raspberry Pi Pico W**, **ADC ADS1115** e **FreeRTOS**. O firmware amostra **tensão** e **corrente**, calcula **Vrms**, **Irms**, **potência instantânea**, **tensão por unidade (PU)**, registra dados em **cartão SD** e envia métricas ao **ThingSpeak** via Wi‑Fi (LwIP).  
A aplicação é organizada em **tarefas FreeRTOS** e **bibliotecas modulares** (ADC, energia, Wi‑Fi, NTP/RTC, logger, SD e ThingSpeak).

---

## 🗂️ Sumário

- [Entregáveis](#-entregaveis)
- [Recursos](#-recursos)
- [Arquitetura](#-arquitetura)
- [Hardware](#-hardware)
- [Dependências e Toolchain](#-dependências-e-toolchain)
- [Configuração](#-configuração)
- [Build & Gravação](#️-build--gravação)
- [Execução & Logs](#-execução--logs)
- [Estrutura do Projeto](#-estrutura-do-projeto)
- [Métricas Calculadas](#-métricas-calculadas)
- [Cartão SD (SPI)](#-cartão-sd-spi)
- [ThingSpeak](#-thingspeak)
- [Dicas de Calibração](#-dicas-de-calibração)
- [Licença](#-licença)

---

## 📧 Entregáveis

| Projeto | Descrição |
|---------|-----------|
| [Etapa 1](./Etapa%201/) | Definição de Requisitos e Lista de Materiais |
| [Etapa 2](./Etapa%202/) | Arquitetura e Modelagem |
| [Etapa 3](./Etapa%203/) | Prototipagem e Ajustes |
| [Etapa 4](./Etapa%204/) | Entrega do Projeto Final |

---

## ✅ Recursos

- **Amostragem síncrona** de 2 canais do **ADS1115** com cálculo de FFT‑free RMS (janelas de **128 amostras @ 200 Hz** ~ 0,64 s).
- **Cálculos de energia**: Vrms, Irms, potência instantânea (Pinst) e **tensão por unidade (Vpu)** com base **127 V**.
- **Limiares PRODIST** com **contadores acumulativos**:
  - **Sobretensão** quando `Vpu > 1.10`
  - **Subtensão** quando `Vpu < 0.90`
- **Envio periódico ao ThingSpeak** (TCP HTTP) com reconexão e timeouts.
- **Registro local** em **cartão SD** (FAT) com **timestamp** via **RTC**.
- **Sincronização de hora por NTP** (UDP) na inicialização e sob eventos de rede.
- **Modo MOCK do ADS1115** para testes sem hardware (senoide 60 Hz + ruído).

---

## 🧱 Arquitetura

**Tarefas principais (FreeRTOS):**

- `energy_monitor_task` — lê o ADS1115, calcula **Vrms/Irms/Pinst/Vpu**, atualiza contadores PRODIST e o último quadro de medidas.
- `thingspeak_task` — empacota e envia os campos ao ThingSpeak após conexão Wi‑Fi e em intervalos regulares.
- `rtc_ntp_task` — inicializa o RTC e executa sincronização por NTP.
- `sd_card_log_task` — grava CSV em cartão SD com timestamp e valores‑chave.
- `logger` — utilitário central de logs com carimbo de data/hora.

**Bibliotecas (lib/):**

- `ads1115_adc.*` (driver I²C), `ads1115_adc_mock.*` (simulador) e `ads1115_adc_wrapper.*` (seleção por macro).
- `energy_monitor.*` (cálculos, filtros, PU e contadores).
- `wifi_manager.*` (CYW43 + LwIP).
- `rtc_ntp.*` (RTC + cliente NTP).
- `thingspeak.*` (construção de payload HTTP e envio TCP).
- `logger.*`, `sd_card.*`, `sd_card_log_task.*` e `hw_config.*`.

**Fluxograma:**

<img src="https://github.com/EmbarcaTech-2025/projeto-final-gabriel_marina_luana/blob/main/figuras/fluxograma.jpg" width=100% height=100%>

---

## 🔌 Hardware

- **Placa:** Raspberry Pi **Pico W** (RP2040 + CYW43)
- **ADC:** **ADS1115** @ **I²C 0x48**
  - **SDA → GPIO 0**
  - **SCL → GPIO 1**
- **Cartão SD (SPI0):**
  - **MISO → GPIO 16**
  - **MOSI → GPIO 19**
  - **SCK  → GPIO 18**
  - **CS   → GPIO 17**

> **Calibração elétrica (valores de exemplo):**  
> `VOLT_CONV_FACTOR = 301.15` (Vadc→Vreal) e `CURR_CONV_FACTOR = 54.87` (V/V→A).  
> Ajuste conforme seu divisor, shunt/TC e ganhos do ADS1115.


**Esquema:**

<img src="https://github.com/EmbarcaTech-2025/projeto-final-gabriel_marina_luana/blob/main/figuras/esquema.jpg" width=100% height=100%>

---

## 📦 Dependências e Toolchain

Clone os repositórios **na raiz** do projeto antes da build:

```bash
git clone https://github.com/FreeRTOS/FreeRTOS-Kernel.git
git clone https://github.com/carlk3/no-OS-FatFS-SD-SPI-RPi-Pico
```

**Toolchain sugerido**

- **Pico SDK** configurado (via `PICO_SDK_PATH` ou submódulo).
- **CMake** ≥ 3.12 e **Ninja** (ou Make).
- **GCC ARM** (arm-none-eabi-gcc).
- **VS Code** (+ extensão Pico) opcional.

---

## 🔧 Configuração

Crie o arquivo **privado** `include/credentials.h` (não versionar) com suas credenciais:

```c
#ifndef CREDENTIALS_H
#define CREDENTIALS_H

#define SSID     "nome_da_rede"       /* SSID da rede Wi-Fi */
#define PASSWORD "senha"              /* Senha da rede Wi-Fi */
#define API_KEY  "12345678ABCDEFGH"   /* Chave de escrita ThingSpeak */

#endif /* CREDENTIALS_H */
```

> Dica: adicione `include/credentials.h` ao `.gitignore`.  
> Mantenha um `include/credentials.example.h` no repositório para referência.

### Modo simulado (sem hardware)

Para usar o **MOCK** do ADS1115 (gera senoide 60 Hz):

```c
/* Em lib/ads1115_adc_wrapper.c */
#define ADS1115_USE_MOCK 1   /* 0 = driver I2C real; 1 = simulador */
```

---

## 🛠️️ Build & Gravação

1. **Configurar e compilar** (ex.: Ninja):

   ```bash
   mkdir -p build && cd build
   cmake -G Ninja ..
   ninja
   ```

2. **Gravar `.uf2`**:
   - Conecte o Pico **com BOOTSEL pressionado**.
   - Monte a unidade `RPI-RP2`.
   - **Copie** o arquivo `.uf2` gerado para a unidade.

> Logs via USB CDC (stdio) podem ser habilitados na configuração do projeto.

---

## ▶️ Execução & Logs

- **Serial (USB CDC):** abra um terminal (115200 bps padrão) e acompanhe as mensagens do logger. O timestamp combina **RTC** (data/hora) com alta resolução do `time_us_64()`.
- **Wi‑Fi + NTP:** na inicialização, o sistema conecta à rede e sincroniza o RTC via NTP; reconecta quando necessário.
- **ThingSpeak:** após a primeira conexão e de tempos em tempos, a tarefa envia campos como **Vrms**, **Irms**, **Pinst**, **Energia (Wh)**, **Uptime (s)**, **Vpu**, **Over/Under counters**.

---

## 🗃️ Estrutura do Projeto

```
/src
  main.c
/lib
  ads1115_adc.c
  ads1115_adc_mock.c
  ads1115_adc_wrapper.c
  energy_monitor.c
  energy_monitor.h
  wifi_manager.c
  rtc_ntp.c
  thingspeak.c
  logger.c
  sd_card.c
  sd_card_log_task.c
  hw_config.c
/include
  credentials.h            (privado)
  FreeRTOSConfig.h
  lwipopts.h
CMakeLists.txt
```

---

## 📈 Métricas Calculadas

- **Vrms/Irms** e **Pinst** calculados por janelas de **128 amostras @ 200 Hz** (sem FFT).
- **Vpu** = `Vrms / 127.0` (`VBASE_RMS` = 127 V).
- **Contadores PRODIST** (acumulativos por janela):
  - incrementa **Sobretensão** quando `Vpu > 1.10`
  - incrementa **Subtensão** quando `Vpu < 0.90`

---

## 💾 Cartão SD (SPI)

- Interface **SPI0** com pinos:
  - **MISO 16 / MOSI 19 / SCK 18 / CS 17 / CD 13**
- A tarefa de log grava **CSV** com timestamp e principais grandezas (Vrms, Irms, Pinst, Vpu, contadores).  
- Formate o cartão em **FAT/FAT32**.

---

## 🌐 ThingSpeak

1. Crie um **canal** no ThingSpeak e substitua `API_KEY` em `include/credentials.h`.
2. **Mapeamento sugerido** de campos (edite conforme seu canal):
   - **field1**: Vrms (V)
   - **field2**: Irms (A)
   - **field3**: Potência instantânea (W)
   - **field4**: Energia acumulada estimada (Wh)
   - **field5**: Uptime (s)
   - **field6**: Vpu
   - **field7**: Contador **Sobretensão**
   - **field8**: Contador **Subtensão**

> A tarefa usa **TCP (HTTP)** via LwIP, com retry e timeout.

---

## 🔧 Dicas de Calibração

- Ajuste `VOLT_CONV_FACTOR` e `CURR_CONV_FACTOR` ao seu divisor/shunt/TC e configuração do **PGA** do ADS1115.
- Corrija offsets DC (`VOLT_DC_OFFSET` e `CURR_DC_OFFSET`) se houver deriva.
- Valide com carga conhecida (ex.: lâmpada resistiva) e multímetro/true‑RMS.

---

## 📜 Licença

Distribuído sob **GNU GPL‑3.0**.

---

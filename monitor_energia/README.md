
# Trabalho Final - Monitor de Energia

Autores: **Gabriel Mattano, Luana Vacari e Marina Donaire**

Curso: Residência Tecnológica em Sistemas Embarcados

Instituição: EmbarcaTech - HBr

Campinas, Julho/Agosto/Setembro de 2025

---

Para fazer a build é necessário clonar o repositório do kernel FreeRTOS e o repositório da biblioteca para o cartão SD na raiz do diretório:

`git clone https://github.com/FreeRTOS/FreeRTOS-Kernel.git`

`git clone https://github.com/carlk3/no-OS-FatFS-SD-SPI-RPi-Pico`

Também é necessario criar o arquivo `credentials.h` no diretório `include`:

    ```bash
    #ifndef CREDENTIALS_H
    #define CREDENTIALS_H

    #define SSID "nome_da_rede"          /* SSID da rede Wi-Fi. */
    #define PASSWORD "senha"             /* Senha da rede Wi-Fi. */
    #define API_KEY "12345678ABCDEFGH"   /* Chave de escrita do canal ThingSpeak. */

    #endif // CREDENTIALS_H

---

## 📜 Licença
GNU GPL-3.0.

# MQ02_AcionamentoRele_Dashboard_ESP32_Wi_Fi_LowEnergy

📌 **Resumo Executivo**

Este repositório documenta o desenvolvimento de um firmware de alta disponibilidade dedicado ao monitoramento de gases inflamáveis e fumaça (via sensor MQ-02), acionamento de atuadores e operação em modo *low energy*. O sistema foi concebido sob rigorosos paradigmas de Engenharia de Sistemas para microcontroladores da família ESP32, implementando uma infraestrutura híbrida de rede (STA + AP) para o gerenciamento remoto e determinístico de cargas de potência (Relés), integrando uma interface Web assíncrona (AJAX/Fetch API) e controle via UART.

O diferencial arquitetônico reside na abordagem **Green IoT**. A aplicação utiliza recursos nativos do FreeRTOS para gerenciamento de concorrência e mitigação de consumo de rádio e CPU através de *Dynamic Frequency Scaling* (DFS) e *Modem Sleep*, garantindo eficiência energética sem comprometer a segurança crítica da detecção de gás.

---

🏗️ **Arquitetura do Sistema e FreeRTOS**

O firmware abandona o paradigma procedural clássico em favor de uma topologia de tarefas distribuídas (Multithreading), assegurando que o processamento pesado da pilha TCP/IP não interfira na latência da leitura do sensor MQ-02.

### Topologia de Tarefas (Tasks)

* **TaskGasMonitoring (Prioridade Crítica):** Responsável pelo polling analógico do sensor MQ-02 e processamento de sinal. Opera com amostragem estatística para filtragem de ruído, garantindo que o limiar de acionamento do relé seja atingido de forma determinística.
* **TaskRelayControlCode (Alta Prioridade):** Tarefa dedicada ao acionamento físico do GPIO. Em estado de estabilidade, permanece em *Task Suspension* (vTaskSuspend), sendo reativada assincronamente via IPC (vTaskResume) quando o sensor detecta níveis críticos ou via comando externo.
* **TaskPowerManagerCode (Prioridade Média):** Máquina de estados responsável pela telemetria e gerenciamento da frequência do processador (DFS) baseada no estado de alerta do sensor.
* **TaskSerialControl (Core 0):** Escuta assíncrona da interface UART para redundância de controle e depuração de níveis de PPM (partes por milhão) detectados.
* **Application Loop (Core 1):** Dedicado exclusivamente ao roteamento do Servidor Web HTTP e entrega do Dashboard.

---

⚡ **Gestão Avançada de Energia (Green IoT)**

A eficiência energética deste nó é assegurada por camadas independentes de otimização:

1.  **Dynamic Frequency Scaling (DFS):** O sistema monitora a estabilidade do sensor. Em estado nominal (ar limpo), o clock da CPU é reduzido para **80MHz**. Caso o sensor MQ-02 detecte uma tendência de subida nos valores analógicos, a frequência é escalada para **160MHz** (Boost) para garantir resposta imediata aos protocolos de segurança.
2.  **Modem Sleep 802.11:** A flag `WIFI_PS_MIN_MODEM` é invocada para coordenar os *DTIM beacons*, permitindo que o rádio entre em repouso entre os intervalos de transmissão, reduzindo drasticamente a corrente média de consumo.
3.  **Proteção de Região Crítica (Mutex):** O estado do relé e os limiares de leitura do MQ-02 são protegidos por `xSemaphoreCreateMutex()`, prevenindo *Race Conditions* entre o núcleo de processamento de rede e o núcleo de controle de hardware.

---

🌐 **Conectividade e Interface Híbrida**

O dispositivo atua como um nó de borda (*Edge Computing*) robusto, inicializando simultaneamente:

* **Interface WAN (Station):** Conecta-se à infraestrutura local para reporte de telemetria e sincronização NTP (pool.ntp.org).
* **Interface LAN (Access Point):** Gera um SSID de contingência (ESP32-MQ02-Manager), permitindo calibração do sensor e acionamento forçado do relé mesmo em cenários de isolamento de rede externa.

### Dashboard Web (UI/UX)
Servida nativamente na porta 80, a interface utiliza **Bootstrap 5** para responsividade:
* **Visualização de Dados:** Gráficos em tempo real dos níveis de gás e status do relé.
* **Comunicação Assíncrona:** API `fetch()` para requisições RESTful non-blocking nos endpoints `/api/status` e `/api/control`.
* **Diagnóstico de Rede:** Exibição de RSSI, Uptime e lista de clientes conectados ao AP.

---

🛠️ **Especificações de Hardware e Deploy**

### Pinout Configurado
| Componente | Pino GPIO | Função |
| :--- | :--- | :--- |
| **Sensor MQ-02** | GPIO 34 (ADC1) | Entrada Analógica (Leitura de Gás/Fumaça) |
| **Módulo Relé** | GPIO 22 | Saída Digital (Controle de Carga) |
| **Status LED** | GPIO 2 | Indicador de Conectividade/Alerta |

### Requisitos de Software
* ESP32 Core (v3.0.x ou superior)
* Bibliotecas: `WiFi.h`, `WebServer.h`, `esp_wifi.h`, `time.h`, `freertos/FreeRTOS.h`.

### Procedimento de Inicialização
1.  Configure as credenciais em `SSID_STA` e `PASS_STA` no arquivo principal.
2.  Ajuste o `THRESHOLD_GAS` conforme a necessidade de sensibilidade do seu ambiente.
3.  Realize o upload via Arduino IDE ou PlatformIO (115200 baud).
4.  Aguarde o aquecimento (*burn-in*) do sensor MQ-02 (aprox. 24-48h para estabilidade máxima em campo).

---

🎓 **Autor e Direitos Intelectuais**

**CFSJ TECH | Engenharia de Sistemas Inteligentes** **Autor:** Cláudio Francisco  
**Cargo:** CEO & Lead Engineer / Engenharia de Computação (PUC Minas)  
**Data:** Fevereiro de 2026  

*Projeto desenvolvido como framework de referência para segurança industrial e residencial utilizando sistemas embarcados de baixo consumo.*

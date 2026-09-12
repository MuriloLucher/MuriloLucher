# Murilo Lucher da Silveira

Engenheiro Eletricista formado pela UFSM. Desenvolvo eletrônica embarcada na Fluxo Equipamentos Eletrônicos, em Chapecó, Santa Catarina: projeto a placa, escrevo o firmware e cuido do servidor que recebe o dado.

Antes disso foram três anos em laboratório acreditado pelo Inmetro, ensaiando inversores fotovoltaicos para certificação, e depois P&D em parceria com a indústria. Cheguei ao desenvolvimento pelo lado de quem reprova equipamento no ensaio e precisa dizer exatamente onde ele falhou.

📍 Chapecó, SC · 📫 lucher.murilo@gmail.com · [LinkedIn](https://www.linkedin.com/in/murilo-lucher/)

---

## FX Gateway

Telemetria para equipamentos de frigorífico, da placa à nuvem. Substitui o tronco RS-485 que atravessa a planta por rádio, e entrega o dado de forma que ele sirva de evidência em auditoria do SIF.

O projeto é meu do começo ao fim: **placa, firmware, protocolo, servidor e banco**.

### O problema

Equipamentos de insensibilização e hidrômetros ficam espalhados pela planta, cada um com sua saída Modbus. A ligação convencional é um tronco RS-485 longo, que atravessa área molhada, passa perto de inversores de frequência e contatores, e que só se descobre estar rompido quando alguém vai buscar o dado. Trocar esse tronco por rádio resolve a fiação, mas cria uma exigência nova: o sistema precisa saber, na hora, que um nó parou de reportar. Silêncio não pode ser confundido com "tudo bem".

### Hardware

PCB própria em KiCad, hoje na quinta revisão, fabricada e em operação.

- **ESP32-S3** (módulo Heltec Wireless Stick Lite V3, com **SX1262** e TCXO internos), 8 MB de flash
- **FRAM MB85RS256B** e **RTC DS3231** em barramento SPI compartilhado, com trava de acesso
- **Dois canais RS-485**, em UARTs separadas, com transceptores próprios
- **Isolação galvânica** no canal que sai da caixa, com módulo isolado que já traz a fonte. Terra de painel industrial não está no mesmo potencial: sem a barreira, o gateway vira caminho de retorno entre dois pontos de aterramento, e é assim que se queima transceptor em campo
- **Ethernet W5500** prevista para instalações que tenham rede cabeada
- Protótipos fresados em CNC e gravados a laser de fibra aqui mesmo, e montados em **insersora SMD** na própria empresa: do arquivo de fabricação à placa montada e testada, sem depender de fornecedor para fechar uma revisão. É o que permite a placa estar na quinta revisão

### Firmware

C sobre **ESP-IDF** e **FreeRTOS**, com drivers próprios para SX1262, FRAM, RTC e RS-485.

- **Identidade vem do eFuse**, não de `#define`. Um único binário atende toda a linha de produtos: o papel do equipamento é lido no boot a partir do MAC gravado em fábrica
- **Atualização remota por OTA**, com partições A/B
- **Duas filas persistentes e independentes**: FRAM para o rádio, flash para o MQTT. Não é uma fila com dois cursores, e a distinção importa: cada caminho falha por motivo diferente e se recupera em ritmo diferente
- **Autonomia de 92 horas** com três pontos de medição, quando o enlace cai
- **Aquisição Modbus-RTU** com perfis por equipamento, endereço configurável e máscara de pontos ativos
- **Servidor web embarcado** para acesso local, com páginas servidas de partição SPIFFS própria, gráficos em SVG puro (sem biblioteca, porque não há internet no chão de fábrica) e histórico local em partição dedicada

### Enlace

**LoRa** ponto a ponto, sem LoRaWAN, com protocolo próprio.

- **ACK de entrega**: o pacote só sai da fila quando o receptor confirma
- **Chave de criptografia derivada por dispositivo** a partir da chave da célula, de modo que comprometer um nó não compromete a célula
- Timings calculados a partir do *airtime* real do spreading factor em uso, não constantes fixas: em SF12 um pacote leva 2,5 s no ar, e um timeout herdado de SF9 derruba o enlace com o rádio perfeito
- Spreading factor por perfil de equipamento, porque insensibilizador em célula curta e hidrômetro distribuído têm exigências opostas de alcance e cadência

### Servidor e dados

O dado é gravado **na planta e na nuvem**, e nenhuma das duas é cache da outra.

**Ingestão.** Um ingestor assina o broker MQTT e grava no banco. A chave natural é `(cmac, seq)` com restrição de unicidade, o que torna a reentrega idempotente: o gateway pode reenviar à vontade que não duplica.

**Camadas de persistência.**

| camada | onde | para quê |
|---|---|---|
| flash do ESP32 | no equipamento | sobreviver à queda de enlace, cerca de 11 dias |
| SQLite | servidor da planta | operação local, independente de internet |
| PostgreSQL + TimescaleDB | VPS | histórico longo, relatórios, acesso remoto |

**Encaminhador.** Entrega para a nuvem apenas o que o banco local já confirmou, e avança o cursor só depois do aceite. Se o VPS ficar fora, a planta continua registrando e o atraso é recuperado sozinho.

**Transporte.** MQTT sobre TLS, com autoridade certificadora própria e certificado por cliente.

**Cadência configurável.** A periodicidade de registro não é um número que eu escolhi: vem do Programa de Autocontrole aprovado pelo SIF de cada planta. Por isso a cadência de gravação do histórico é parâmetro por instalação, separada da cadência de leitura. O painel mostra em 5 segundos o que o banco guarda a cada 60, e a amostragem **força o registro em toda transição de alarme**, para que a política de armazenamento nunca engula o evento que interessa.

**Painel web.** Frota organizada por ponto de medição, não por gateway, porque um gateway pode ler vários equipamentos. Atualização por SSE em vez de *polling*: a leitura chega à tela em cerca de 0,6 s. Relatórios em PDF gerados no servidor. A **interface visual do painel é de [Victor Bonadiman](https://github.com/Vbonadiman)**; a arquitetura do servidor, a API, as consultas e o modelo de dados são meus.

**Detecção de falha de link.** Um nó mudo dispara alarme por ponto. É requisito, não conveniência: sem isso, a substituição do tronco cabeado por rádio não se sustenta perante a norma.

Código fechado. Aqui está a engenharia, não a fonte.

`C` · `ESP-IDF` · `FreeRTOS` · `ESP32-S3` · `SX1262` · `Modbus RTU` · `RS-485` · `MQTT/TLS` · `PostgreSQL` · `TimescaleDB` · `SQLite` · `Python` · `KiCad` · `Montagem SMD`

> **Crédito.** O **supervisório industrial** que roda ao lado deste sistema e a **interface visual do painel** são de [Victor Bonadiman](https://github.com/Vbonadiman), que responde pelos sistemas internos da empresa. A arquitetura do servidor, o modelo de dados e a integração são minhas: adaptei o supervisório ao protocolo e ao banco do FX Gateway.

---

## Pesquisa

### Detecção de falta de arco CC em inversores fotovoltaicos, com a WEG

Campanha experimental conduzida no Instituto de Energia e Mobilidade da UFSM, com a [AUFTEK Tecnologia](https://github.com/AUFTEK-TECNOLOGIA) e em parceria com a WEG, sob os requisitos da IEC 63027.

A pergunta era prática: **onde medir a corrente muda a chance de detectar o arco?** Comparei quatro pontos em um inversor comercial com Boost de entrada, a string em falta, a string de referência, o ponto após o paralelismo das strings e o capacitor de entrada, e medi o efeito de cada um sobre a sensibilidade espectral.

A aquisição foi feita sobre o kit de referência da Texas Instruments para detecção de arco, TIDA-010955 com o DSP TMS320F28P55x, que traz acelerador de rede neural dedicado à classificação da assinatura. A análise usou espectrogramas, resposta em frequência e métricas espectrais na faixa de 10 a 50 kHz.

📄 *Experimental Evaluation of the Influence of Current Measurement Point on DC Arc Fault Detection in Photovoltaic Inverters*, **SEPOC 2025**, primeiro autor, em coautoria com a equipe de Drives & Controls da WEG.

`Texas Instruments C2000` · `Processamento digital de sinais` · `Análise espectral` · `IEC 63027`

### Datalogger de bancada: o hardware

Antes de existir a campanha acima, faltava um instrumento próprio para gravar corrente em alta taxa durante os ensaios. **Projetei o hardware**: uma placa em KiCad no formato de expansão do kit STM32F429I, fabricada e usada nos ensaios.

O que ela resolve, e por quê:

- **Condicionamento do sensor Hall** com dois OPA376: o sinal bruto do sensor passa por ganho e filtragem antes do ADC, e uma rede de referência própria fixa o ponto de operação da entrada
- **Proteção por TVS** SM6T10CA nas entradas. O cabo sai do painel, e o ensaio consiste em provocar arco de propósito: a entrada precisa sobreviver ao mesmo fenômeno que está medindo
- **RS-485** com SN65HVD10, para a placa conversar com o resto da bancada
- **Alimentação de 24 V**, que é a tensão disponível no painel do inversor, com conversor CC-CC embarcado
- Ligações dedicadas para a **entrada e a saída do conversor Boost**, que são dois dos quatro pontos de medição comparados no artigo

> **Crédito.** O **firmware do STM32 e o software de bancada** são da [AUFTEK Tecnologia](https://github.com/AUFTEK-TECNOLOGIA), parceira do projeto. Meu escopo nesta placa foi o hardware.

`KiCad` · `Condicionamento de sinal` · `Instrumentação` · `STM32F429` · `RS-485`

---

## Antes: metrologia

Ensaios de certificação acreditados pelo Inmetro em inversores on-grid, off-grid, híbridos e controladores de carga, em laboratório com selo ILAC. Depois, supervisão da equipe de ensaio e padronização dos procedimentos.

É de onde vem o resto. Ensaio de certificação não admite meio-termo: ou o equipamento atende à norma, ou não atende, e alguém precisa demonstrar em que ponto.

---

## Como eu trabalho

**Dado errado é pior que dado ausente.** Num sistema que gera evidência para auditoria, uma leitura faltando é visível; uma leitura plausível e errada passa. Já tive um ponto publicando a medição de outro medidor, com todos os indicadores de sucesso ligados e nada no log. Desde então, todo caminho de dado precisa de um contador capaz de denunciar a contradição.

**Medir antes de concluir.** "Não respondeu" no RS-485 é indistinguível de fiação invertida. O que separa as hipóteses não é opinião: é o contador que fica congelado enquanto o outro avança.

**O instrumento não pode produzir o fenômeno.** Abrir a serial reinicia a placa, e varrer endereços trava o medidor que se quer medir. Quando o instrumento interfere, a medição mede a si mesma.

**Verificação que degrada para "seguindo" não é verificação.** Uma trava que erra para o lado permissivo deixa passar exatamente o caso que ela existe para impedir, e em série. Estado que governa comportamento precisa ser observável, não deduzível.

**O porquê fica escrito.** Cada decisão e cada armadilha que me custou uma tarde vão para o documento do projeto, porque em três meses eu não lembro por que aquela linha existe.

---

## Formação

**Engenharia Elétrica**, ênfase em Eletrônica de Potência e Controle, UFSM, 2016 a 2022.

TCC: equalizador de carga para baterias em série, baseado em capacitor comutado em topologia de duplo nível. A topologia dispensa elementos magnéticos e transfere carga por chaveamento entre capacitores; o arranjo em duplo nível reduz o número de etapas para equalizar células distantes entre si na série, que é a limitação da topologia clássica.

📄 *Implementation and Analysis of a Double-Tiered Switched Capacitor Series Battery Equalizer Circuit*, **COBEP/SPEC 2023**, primeiro autor, com Mauricio Mendes da Silva (Universidad Tecnológica del Uruguay) e António M. S. Spencer Andrade (UFRGS e UFSM).

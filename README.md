# DopplerVibes

## Ideia

Um dispositivo que recebe entradas de uma máquina de ultrassom Doppler capturando batimentos cardíacos fetais, transforma o áudio em vibrações que podem ser sentidas com as mãos, direcionado a gestantes com deficiências auditivas. Este projeto visa promover inclusão e acessibilidade, permitindo que mães com deficiência auditiva experimentem a alegria de sentir os batimentos cardíacos do bebê, algo que normalmente é ouvido. A ideia é inovadora, combinando tecnologias de ultrassom, processamento de sinais digitais e feedback háptico para criar uma experiência tátil imersiva.

## Plano Geral

- Pesquisar como capturar entrada de áudio de dispositivos Doppler de ultrassom: priorizar jack de 3.5mm (áudio analógico), considerar Bluetooth para sem fio, evitar API para protótipo inicial.
- Implementar análise de áudio: usar filtro passa-banda para isolar frequências de batimentos cardíacos fetais (120-160 bpm), aplicar detecção de picos para extração de ritmo, pesquisar bibliotecas DSP para ESP32.
- Projetar sistema háptico: usar atuador LRA com driver DRV2605 para pulsos precisos, conectar via I2C ao ESP32.
- Prototipar formato do dispositivo: criar almofada ou luva wearable com case impresso em 3D, materiais macios, bateria LiPo para portabilidade.
- Garantir segurança e aprovações: classificar como dispositivo de bem-estar para simplificar processo ANVISA, validar ausência de interferência no Doppler.
- Construir prova de conceito: usar ESP32, simular áudio Doppler de vídeos YouTube, integrar entrada ADC, filtragem, detecção de picos e saída háptica.

## Sugestões Detalhadas de Desenvolvimento

### 1. Captura de Áudio do Doppler

A captura de áudio é o ponto de entrada do sistema. Os dispositivos Doppler fetais emitem um sinal de áudio através de um alto-falante ou saída de fone de ouvido. Para capturar esse sinal, precisamos de uma interface que converta o áudio analógico em dados digitais processáveis.

#### Opções de Captura
- **Jack de Áudio de 3.5mm (Recomendado para Protótipo Inicial)**: A maioria dos Dopplers domésticos possui uma saída padrão de fone de ouvido. Este sinal é analógico e pode ser conectado diretamente a um conversor analógico-digital (ADC). Vantagens: Simples, confiável e não requer pareamento. Desvantagens: Cabo físico limita a mobilidade.
- **Bluetooth**: Alguns modelos modernos suportam transmissão sem fio. Isso permitiria um dispositivo totalmente portátil. Vantagens: Sem cabos, maior conforto. Desvantagens: Adiciona complexidade de pareamento, latência potencial e consumo de energia.
- **API Direta**: Rara em dispositivos de consumo; mais comum em equipamentos hospitalares. Não é viável para protótipos iniciais devido à necessidade de integração com sistemas proprietários.

#### Hardware Sugerido
Para implementar a captura, recomendamos o ESP32 como microcontrolador principal. O ESP32 possui:
- ADCs internos de 12 bits para leitura de sinais analógicos.
- Suporte a Bluetooth 4.2/5.0 para futuras expansões.
- Wi-Fi para possíveis atualizações over-the-air (OTA).
- Processamento suficiente para DSP básico.

Alternativas: Raspberry Pi Pico (mais barato, mas sem Bluetooth integrado) ou Arduino Nano 33 BLE (sem Wi-Fi).

#### Implementação
Conecte a saída do Doppler ao pino ADC do ESP32 (ex: GPIO 34). Use uma resistência pull-down para estabilizar o sinal. No código, configure o ADC com resolução máxima e taxa de amostragem de pelo menos 8kHz para capturar frequências de áudio relevantes.

Exemplo de código básico em Arduino IDE:
```cpp
#define AUDIO_PIN 34
void setup() {
  Serial.begin(115200);
  analogReadResolution(12);
}
void loop() {
  int audioSample = analogRead(AUDIO_PIN);
  // Processar amostra
}
```

Desafios: Ruído elétrico pode interferir; use capacitores de desacoplamento. Teste com diferentes Dopplers para compatibilidade.

### 2. Análise do Áudio e Extração do Ritmo

Esta é a etapa mais crítica e técnica. O áudio Doppler não é um tom puro, mas um "whoosh" rítmico causado pelo efeito Doppler nos batimentos cardíacos. Precisamos extrair o ritmo (BPM) em tempo real.

#### Processamento de Sinais Digitais (DSP)
- **Filtragem Passa-Banda**: Aplique um filtro para isolar frequências entre 1-5 Hz (para ritmo) e remover ruídos de fundo. Use um filtro Butterworth ou FIR implementado via biblioteca.
- **Detecção de Picos**: Identifique os pontos altos de cada "whoosh". Algoritmos como o de Scharr ou detecção baseada em threshold podem ser usados.
- **Cálculo de BPM**: Meça intervalos entre picos consecutivos. Use uma média móvel para suavizar variações.

#### Bibliotecas e Ferramentas
Pesquise bibliotecas como:
- ArduinoFFT para transformada de Fourier rápida (FFT) em ESP32.
- Filtros DSP em C++ puro para eficiência.
- Exemplos de projetos similares: Monitores cardíacos DIY ou detectores de batidas musicais.

Implementação passo a passo:
1. Amostrar o áudio a 8kHz.
2. Aplicar filtro passa-banda.
3. Detectar picos usando derivada ou comparação com threshold dinâmico.
4. Calcular BPM: bpm = 60 / (tempo_entre_picos).

Código exemplo:
```cpp
// Usando ArduinoFFT
#include <arduinoFFT.h>
arduinoFFT FFT = arduinoFFT();
double vReal[SAMPLES], vImag[SAMPLES];
// ... amostragem
FFT.Windowing(vReal, SAMPLES, FFT_WIN_TYP_HAMMING, FFT_FORWARD);
FFT.Compute(vReal, vImag, SAMPLES, FFT_FORWARD);
FFT.ComplexToMagnitude(vReal, vImag, SAMPLES);
// Analisar magnitudes para frequências de interesse
```

Desafios: Variações no volume do Doppler podem afetar a detecção. Implemente calibração automática. Teste com gravações reais de batimentos fetais.

### 3. Sistema de Feedback Tátil (Háptico)

Traduzir o ritmo extraído em sensações táteis. O objetivo é simular batimentos cardíacos palpáveis.

#### Componentes Hápticos
- **Atuador Linear Ressonante (LRA)**: Superior a motores ERM para precisão. Responde a frequências específicas, ideal para pulsos curtos.
- **Driver DRV2605**: Controla o LRA via I2C. Suporta bibliotecas de efeitos pré-programados (ex: pulsos, vibrações).

#### Implementação
Conecte o DRV2605 ao ESP32 via I2C (pinos SDA/SCL). Configure efeitos de vibração sincronizados com os picos detectados.

Código exemplo:
```cpp
#include <Wire.h>
#include <DRV2605.h>
DRV2605 haptic;
void setup() {
  haptic.begin();
  haptic.setMode(DRV2605_MODE_INTTRIG);
}
void loop() {
  if (peakDetected) {
    haptic.setWaveform(0, 47); // Efeito de pulso
    haptic.go();
  }
}
```

Desafios: Latência deve ser mínima (<100ms) para realismo. Teste intensidade para conforto.

### 4. Protótipo e Formato do Dispositivo

O formato deve ser ergonômico e seguro para gestantes.

#### Design
- **Almofada Wearable**: Disco macio de 10-15cm de diâmetro, com LRA interno. Material: Silicone hipoalergênico.
- **Luva**: Integrar atuadores nas palmas para sensação direta.

#### Fabricação
- Case impresso em 3D (PLA ou ABS).
- Bateria LiPo 3.7V 500mAh para 2-4 horas de uso.
- Botões para ligar/desligar e calibração.

Desafios: Peso leve (<100g). Teste usabilidade com usuários reais.

### 5. Segurança e Validações

#### Classificação
Como dispositivo de bem-estar, não médico, evita regulações rigorosas. Ainda assim, valide:
- Isolação elétrica: Bateria isolada, sem contato com pele úmida.
- Não-interferência: Teste EMC com Doppler.

#### Aprovações
Consultar ANVISA para certificação. Documentar testes de segurança.

### 6. Prova de Conceito (PoC)

Construa PoC usando ESP32 + DRV2605 + LRA.
- Simule áudio: Reproduza vídeos YouTube de Doppler em PC, conecte saída ao ESP32.
- Integre: ADC -> Filtragem -> Detecção -> Háptico.
- Métricas: Precisão BPM >90%, latência <200ms.

#### Cronograma
- Semana 1: Montar hardware básico.
- Semana 2: Implementar captura e filtragem.
- Semana 3: Adicionar detecção de picos e háptico.
- Semana 4: Testes e refinamentos.



# Simulador de Bomba de Jogos🤖

Este projeto é uma simulação de bomba usada em jogos como Valorant ou CS:GO, desenvolvida em Arduino. Ele utiliza LEDs, buzzer, botões e um display 
LCD 16x2 (sem I2C) para recriar a dinâmica de armar, desarmar e explodir uma bomba de forma divertida e interativa.

## Materiais🛠️

<img width="683" height="800" alt="download" src="https://github.com/user-attachments/assets/b7f2fdb3-414f-40fb-bef6-ff9bef6eae6b" />

**1.** Arduino UNO R3

**2.** Dois botões

**3.** Tres LEDs

**4.** Buzzer

**5.** Display LCD 16x2

**6.** Tres resistores 220Ω

OBS: Também é necessário cabos jumper e ferramentas como chaves philips.

## Montagem 


<img width="927" height="524" alt="image" src="https://github.com/user-attachments/assets/433f8b7e-a8d6-4e42-84db-9faa93cecdaf" />



* Conecte o pino RS do LCD no pino 12 do Arduino

* Conecte o pino E (Enable) do LCD no pino 11 do Arduino

* Conecte o pino D4 do LCD no pino 5 do Arduino

* Conecte o pino D5 do LCD no pino 6 do Arduino

* Conecte o pino D6 do LCD no pino 7 do Arduino

* Conecte o pino D7 do LCD no pino 8 do Arduino

* Conecte o VSS do LCD ao GND do Arduino

* Conecte o VDD do LCD ao 5V do Arduino

* Conecte o RW do LCD também ao GND (para ficar sempre em modo escrita)

* Coloque um potenciômetro de 10k:

* Pino do meio → VO (pino de contraste do LCD)

* Um lado do potenciômetro → 5V

* Outro lado → GND

* Conecte o A (LED+) do LCD ao 5V (ou através de resistor 220Ω se quiser limitar brilho)

* Conecte o K (LED-) do LCD ao GND

## Conexões⚡

| Ligação        | Pino do Arduino UNO |
| -------------- | ------------------- |
| RS → D12       | LCD                 |
| E  → D11       | LCD                 |
| D4 → D5        | LCD                 |
| D5 → D6        | LCD                 |
| D6 → D7        | LCD                 |
| D7 → D8        | LCD                 |
| RW → GND       | LCD                 |
| VSS → GND      | LCD                 |
| VDD → 5V       | LCD                 |
| VO → Meio Pot  | LCD (contraste)     |
| A (LED+) → 5V  | LCD (backlight)     |
| K (LED-) → GND | LCD (backlight)     |


| Ligação                     | Pino do Arduino UNO                     |
| --------------------------- | --------------------------------------- |
| Botão 1 → D2                | Entrada digital                         |
| Botão 2 → D3                | Entrada digital                         |
| Botão 3 → D4                | Entrada digital                         |
| Outro lado dos botões → GND | Pull-down físico ou usar `INPUT_PULLUP` |


| Ligação                    | Pino do Arduino UNO          |
| -------------------------- | ---------------------------- |
| LED Verde → D9             | Saída digital                |
| LED Vermelho → D10         | Saída digital                |
| LED Azul → D13             | Saída digital (com resistor) |
| Todos os cátodos (–) → GND | Resistores 220Ω em série     |

## Código💻

```
#include <LiquidCrystal.h>

// Definição dos pinos do LCD
LiquidCrystal lcd(12, 11, 5, 6, 7, 8);

// Definição dos pinos dos componentes
const int LED_VERMELHO = 13;
const int LED_AMARELO = 9;
const int LED_VERDE = 10;
const int BUZZER = 4;
const int BOTAO_ARMAR = 2;
const int BOTAO_REINICIAR = 3;

// Estados do jogo
enum Estado {
  AGUARDANDO,
  ARMANDO,
  ATIVA,
  DESARMANDO,
  EXPLODIDA,
  DESARMADA
};

Estado estadoAtual = AGUARDANDO;

// Variáveis de tempo
unsigned long tempoArmarInicio = 0;
unsigned long tempoDesarmarInicio = 0;
unsigned long tempoBombaInicio = 0;
unsigned long tempoPiscarInicio = 0;
unsigned long tempoUltimoBipe = 0;
unsigned long tempoAtualizacaoLCD = 0;
unsigned long tempoMostrarPlacar = 0;
unsigned long tempoBotaoReiniciar = 0;

// Constantes de tempo
const unsigned long TEMPO_ARMAR = 4000;        // 4 segundos para armar
const unsigned long TEMPO_EXPLOSAO = 45000;    // 45 segundos até explodir
const unsigned long TEMPO_DESARMAR = 10000;    // 10 segundos para desarmar
const unsigned long TEMPO_DESARMAR_RAPIDO = 5000; // 5 segundos após primeira interrupção
const unsigned long TEMPO_REINICIAR_CURTO = 3000;  // 3 segundos para reset circuito
const unsigned long TEMPO_REINICIAR_LONGO = 10000; // 10 segundos para zerar placar
const unsigned long INTERVALO_LCD = 500;       // Atualizar LCD a cada 500ms
const unsigned long TEMPO_MOSTRAR_PLACAR = 5000; // Mostrar placar por 5 segundos

// Frequências do buzzer
const int FREQ_DESARMANDO = 1000;  // Frequência para desarme
const int FREQ_EXPLOSAO = 500;     // Frequência para explosão
const int FREQ_BIPE = 1500;        // Frequência para bipe simples
const int FREQ_ARMAMENTO = 2000;   // Frequência para início do armamento

// Variáveis de controle
bool ledVerdeEstado = false;
const unsigned long INTERVALO_BIPE = 500; // Bipe a cada 500ms

// VARIÁVEIS PARA CONTROLE DO DESARME RÁPIDO
bool desarmeRapidoAtivo = false; // Indica se já passou dos 5 segundos em alguma tentativa
bool bipeArmamentoExecutado = false; // Controla se o bipe de armamento já foi executado

// VARIÁVEIS DO PLACAR
int placarDefensores = 0;
int placarAtacantes = 0;
bool mostrandoPlacar = false;

// Variáveis para botão de reinício
bool botaoReiniciarPressionado = false;
bool mensagemEncerramentoMostrada = false;

void setup() {
  // Configuração dos pinos
  pinMode(LED_VERMELHO, OUTPUT);
  pinMode(LED_AMARELO, OUTPUT);
  pinMode(LED_VERDE, OUTPUT);
  pinMode(BUZZER, OUTPUT);
  pinMode(BOTAO_ARMAR, INPUT_PULLUP);
  pinMode(BOTAO_REINICIAR, INPUT_PULLUP);
  
  // Inicialização do LCD
  lcd.begin(16, 2);
  lcd.print("Sistema Bomba");
  lcd.setCursor(0, 1);
  lcd.print("Valorant - Ready");
  
  // Estado inicial dos LEDs
  digitalWrite(LED_VERMELHO, LOW);
  digitalWrite(LED_AMARELO, LOW);
  digitalWrite(LED_VERDE, LOW);
  
  Serial.begin(9600);
  Serial.println("Sistema da Bomba Iniciado!");
  Serial.println("Pressione o botão ARMAR por 4 segundos para ativar");
  
  delay(2000); // Mostra mensagem inicial por 2 segundos
  limparLCD();
}

void loop() {
  unsigned long tempoAtual = millis();
  
  // Verificar botão de reinício
  verificarBotaoReiniciar(tempoAtual);
  
  // Se está mostrando placar, não processa outros estados
  if (mostrandoPlacar) {
    if (tempoAtual - tempoMostrarPlacar >= TEMPO_MOSTRAR_PLACAR) {
      mostrandoPlacar = false;
      estadoAtual = AGUARDANDO;
      limparLCD();
    } else {
      mostrarPlacar();
      delay(50);
      return;
    }
  }
  
  // Atualizar LCD periodicamente
  if (tempoAtual - tempoAtualizacaoLCD >= INTERVALO_LCD) {
    atualizarLCD();
    tempoAtualizacaoLCD = tempoAtual;
  }
  
  switch (estadoAtual) {
    
    case AGUARDANDO:
      // Estado inicial - aguardando armar
      digitalWrite(LED_VERMELHO, LOW);
      digitalWrite(LED_AMARELO, LOW);
      digitalWrite(LED_VERDE, LOW);
      noTone(BUZZER);
      bipeArmamentoExecutado = false;
      
      // Reset das variáveis de desarme quando volta ao estado inicial
      desarmeRapidoAtivo = false;
      
      if (digitalRead(BOTAO_ARMAR) == LOW) {
        tempoArmarInicio = tempoAtual;
        estadoAtual = ARMANDO;
        digitalWrite(LED_AMARELO, HIGH);
        
        // BIPE quando começa o armamento
        tone(BUZZER, FREQ_ARMAMENTO, 300);
        bipeArmamentoExecutado = true;
        
        Serial.println("Armando bomba...");
      }
      break;
      
    case ARMANDO:
      // Botão sendo pressionado para armar
      if (digitalRead(BOTAO_ARMAR) == HIGH) {
        // Botão solto antes do tempo
        estadoAtual = AGUARDANDO;
        digitalWrite(LED_AMARELO, LOW);
        limparLCD();
        Serial.println("Armamento cancelado!");
      } else if (tempoAtual - tempoArmarInicio >= TEMPO_ARMAR) {
        // Bomba armada com sucesso
        estadoAtual = ATIVA;
        tempoBombaInicio = tempoAtual;
        tempoPiscarInicio = tempoAtual;
        ledVerdeEstado = true;
        digitalWrite(LED_VERDE, ledVerdeEstado);
        digitalWrite(LED_AMARELO, LOW);
        
        // Bipe de confirmação
        tone(BUZZER, FREQ_BIPE, 200);
        
        Serial.println("BOMBA ATIVADA! 45 segundos até explosão!");
      }
      break;
      
    case ATIVA:
      // Bomba ativa - contando até explosão
      // Piscar LED verde
      if (tempoAtual - tempoPiscarInicio >= 500) {
        ledVerdeEstado = !ledVerdeEstado;
        digitalWrite(LED_VERDE, ledVerdeEstado);
        tempoPiscarInicio = tempoAtual;
      }
      
      // Bipes periódicos
      if (tempoAtual - tempoUltimoBipe >= INTERVALO_BIPE) {
        tone(BUZZER, FREQ_BIPE, 100);
        tempoUltimoBipe = tempoAtual;
      }
      
      // Verificar se tempo acabou
      if (tempoAtual - tempoBombaInicio >= TEMPO_EXPLOSAO) {
        estadoAtual = EXPLODIDA;
        Serial.println("BOOM! BOMBA EXPLODIU!");
      }
      
      // Verificar se botão desarmar foi pressionado
      if (digitalRead(BOTAO_REINICIAR) == LOW) {
        estadoAtual = DESARMANDO;
        tempoDesarmarInicio = tempoAtual;
        digitalWrite(LED_AMARELO, HIGH);
        digitalWrite(LED_VERDE, LOW);
        
        Serial.println("Desarme iniciado!");
        if (desarmeRapidoAtivo) {
          Serial.println("MODO RÁPIDO - Apenas 5 segundos necessários!");
        } else {
          Serial.println("Modo NORMAL - 10 segundos necessários");
        }
      }
      break;
      
    case DESARMANDO:
      // Desarmando a bomba - O TEMPO DE EXPLOSÃO CONTINUA CONTANDO!
      digitalWrite(LED_AMARELO, HIGH);
      digitalWrite(LED_VERDE, LOW);
      
      // VERIFICAR SE TEMPO DE EXPLOSÃO ACABOU DURANTE O DESARME
      if (tempoAtual - tempoBombaInicio >= TEMPO_EXPLOSAO) {
        estadoAtual = EXPLODIDA;
        Serial.println("BOOM! BOMBA EXPLODIU DURANTE O DESARME!");
        break;
      }
      
      // Determinar qual tempo de desarme usar
      unsigned long tempoDesarmeNecessario;
      if (desarmeRapidoAtivo) {
        tempoDesarmeNecessario = TEMPO_DESARMAR_RAPIDO; // 5 segundos
      } else {
        tempoDesarmeNecessario = TEMPO_DESARMAR; // 10 segundos
      }
      
      // Bipes para desarme - frequência depende do modo
      if (desarmeRapidoAtivo) {
        // Modo rápido - dois bipes rápidos
        if (tempoAtual - tempoUltimoBipe >= 600) {
          tone(BUZZER, FREQ_DESARMANDO, 300);
          tempoUltimoBipe = tempoAtual;
        }
      } else {
        // Modo normal - bipes simples
        if (tempoAtual - tempoUltimoBipe >= 1000) {
          tone(BUZZER, FREQ_BIPE, 200);
          tempoUltimoBipe = tempoAtual;
        }
      }
      
      if (digitalRead(BOTAO_REINICIAR) == HIGH) {
        // Botão desarmar solto - volta para ativa
        
        // VERIFICAR SE ATINGIU 5 SEGUNDOS ANTES DE SOLTAR (para ativar modo rápido)
        unsigned long tempoSegurou = tempoAtual - tempoDesarmarInicio;
        
        // Só ativa modo rápido se segurou 5+ segundos E ainda não estava ativo
        if (!desarmeRapidoAtivo && tempoSegurou >= 5000) {
          desarmeRapidoAtivo = true;
          Serial.println("MODO RÁPIDO ATIVADO! Próxima tentativa: apenas 5 segundos!");
        }
        
        estadoAtual = ATIVA;
        digitalWrite(LED_AMARELO, LOW);
        tempoPiscarInicio = tempoAtual;
        noTone(BUZZER);
        
        Serial.println("Desarme interrompido!");
        Serial.print("Tempo segurou: ");
        Serial.print(tempoSegurou / 1000);
        Serial.println(" segundos");
        
        if (desarmeRapidoAtivo) {
          Serial.println("Próxima tentativa: MODO RÁPIDO (5 segundos)");
        } else {
          Serial.println("Próxima tentativa: Modo NORMAL (10 segundos)");
        }
        
        Serial.print("Tempo restante para explosão: ");
        Serial.print((TEMPO_EXPLOSAO - (tempoAtual - tempoBombaInicio)) / 1000);
        Serial.println(" segundos");
        
      } else if (tempoAtual - tempoDesarmarInicio >= tempoDesarmeNecessario) {
        // Verificação adicional para garantir que está usando o tempo correto
        unsigned long tempoRealDesarme = tempoAtual - tempoDesarmarInicio;
        
        if ((desarmeRapidoAtivo && tempoRealDesarme >= TEMPO_DESARMAR_RAPIDO) ||
            (!desarmeRapidoAtivo && tempoRealDesarme >= TEMPO_DESARMAR)) {
          
          // Bomba desarmada com sucesso - DEFENSORES GANHAM
          placarDefensores++;
          estadoAtual = DESARMADA;
          digitalWrite(LED_AMARELO, LOW);
          digitalWrite(LED_VERDE, HIGH);
          noTone(BUZZER);
          
          // Bipe de sucesso
          tone(BUZZER, FREQ_BIPE, 1000);
          
          Serial.println("BOMBA DESARMADA! Vitória dos defensores!");
          Serial.print("Tempo total de desarme: ");
          Serial.print(tempoRealDesarme / 1000);
          Serial.println(" segundos");
          
          // Reset do desarme para próxima rodada
          desarmeRapidoAtivo = false;
          
          // Mostrar placar
          mostrandoPlacar = true;
          tempoMostrarPlacar = millis();
        }
      }
      break;
      
    case EXPLODIDA:
      // Bomba explodiu - ATACANTES GANHAM
      placarAtacantes++;
      digitalWrite(LED_VERMELHO, HIGH);
      digitalWrite(LED_VERDE, LOW);
      digitalWrite(LED_AMARELO, LOW);
      
      // Som contínuo da explosão
      tone(BUZZER, FREQ_EXPLOSAO);
      
      // Reset do desarme quando explode
      desarmeRapidoAtivo = false;
      
      // Mostrar placar após explosão
      mostrandoPlacar = true;
      tempoMostrarPlacar = millis();
      
      Serial.println("BOOM! BOMBA EXPLODIU! Atacantes ganham!");
      break;
      
    case DESARMADA:
      // Bomba desarmada - aguardando reinício
      digitalWrite(LED_VERDE, HIGH);
      
      // Mostrar placar após desarme
      mostrandoPlacar = true;
      tempoMostrarPlacar = millis();
      
      Serial.println("Bomba desarmada! Defensores ganham!");
      break;
  }
  
  delay(50);
}

void verificarBotaoReiniciar(unsigned long tempoAtual) {
  if (digitalRead(BOTAO_REINICIAR) == LOW) {
    if (!botaoReiniciarPressionado) {
      // Botão pressionado pela primeira vez
      botaoReiniciarPressionado = true;
      tempoBotaoReiniciar = tempoAtual;
      mensagemEncerramentoMostrada = false;
      Serial.println("Botão REINICIAR pressionado...");
    } else {
      // Botão continua pressionado
      unsigned long tempoPressionado = tempoAtual - tempoBotaoReiniciar;
      
      // Mostrar "Encerrando partida" após 3 segundos
      if (tempoPressionado >= 3000 && !mensagemEncerramentoMostrada) {
        lcd.clear();
        lcd.setCursor(0, 0);
        lcd.print("Encerrando");
        lcd.setCursor(0, 1);
        lcd.print("partida...");
        mensagemEncerramentoMostrada = true;
        Serial.println("Mostrando mensagem de encerramento...");
      }
      
      // Reset circuito após 3 segundos
      if (tempoPressionado >= TEMPO_REINICIAR_CURTO && tempoPressionado < TEMPO_REINICIAR_LONGO) {
        // Aguardando completar 10 segundos ou soltar
      }
      
      // Zerar placar após 10 segundos
      if (tempoPressionado >= TEMPO_REINICIAR_LONGO) {
        placarDefensores = 0;
        placarAtacantes = 0;
        estadoAtual = AGUARDANDO;
        mostrandoPlacar = false;
        digitalWrite(LED_VERMELHO, LOW);
        digitalWrite(LED_AMARELO, LOW);
        digitalWrite(LED_VERDE, LOW);
        noTone(BUZZER);
        limparLCD();
        botaoReiniciarPressionado = false;
        
        lcd.clear();
        lcd.setCursor(0, 0);
        lcd.print("Placar Zerado!");
        lcd.setCursor(0, 1);
        lcd.print("Partida Nova");
        Serial.println("PLACAR ZERADO! Nova partida iniciada.");
        
        delay(2000);
        limparLCD();
        return;
      }
    }
  } else {
    if (botaoReiniciarPressionado) {
      // Botão solto
      unsigned long tempoPressionado = tempoAtual - tempoBotaoReiniciar;
      
      // Reset circuito se pressionou entre 3 e 10 segundos
      if (tempoPressionado >= TEMPO_REINICIAR_CURTO && tempoPressionado < TEMPO_REINICIAR_LONGO) {
        estadoAtual = AGUARDANDO;
        mostrandoPlacar = false;
        digitalWrite(LED_VERMELHO, LOW);
        digitalWrite(LED_AMARELO, LOW);
        digitalWrite(LED_VERDE, LOW);
        noTone(BUZZER);
        limparLCD();
        Serial.println("Circuito reiniciado!");
      }
      
      botaoReiniciarPressionado = false;
    }
  }
}

void atualizarLCD() {
  lcd.clear();
  
  switch (estadoAtual) {
    case AGUARDANDO:
      lcd.setCursor(0, 0);
      lcd.print("Aguardando...");
      lcd.setCursor(0, 1);
      lcd.print("Pressione ARMAR");
      break;
      
    case ARMANDO:
      lcd.setCursor(0, 0);
      lcd.print("ARMANDO BOMBA");
      lcd.setCursor(0, 1);
      lcd.print("Segure: ");
      lcd.print(4 - (millis() - tempoArmarInicio) / 1000);
      lcd.print("s");
      break;
      
    case ATIVA:
      lcd.setCursor(0, 0);
      lcd.print("BOMBA ATIVA!");
      lcd.setCursor(0, 1);
      lcd.print("Explode em: ");
      lcd.print((TEMPO_EXPLOSAO - (millis() - tempoBombaInicio)) / 1000);
      lcd.print("s");
      break;
      
    case DESARMANDO:
      lcd.setCursor(0, 0);
      if (desarmeRapidoAtivo) {
        lcd.print("DESARMANDO [RAPIDO]");
      } else {
        lcd.print("DESARMANDO [NORMAL]");
      }
      lcd.setCursor(0, 1);
      unsigned long tempoRestanteDesarme;
      if (desarmeRapidoAtivo) {
        tempoRestanteDesarme = TEMPO_DESARMAR_RAPIDO - (millis() - tempoDesarmarInicio);
      } else {
        tempoRestanteDesarme = TEMPO_DESARMAR - (millis() - tempoDesarmarInicio);
      }
      lcd.print("Faltam: ");
      lcd.print(tempoRestanteDesarme / 1000);
      lcd.print("s");
      break;
      
    case EXPLODIDA:
      lcd.setCursor(0, 0);
      lcd.print("!!! EXPLODIU !!!");
      lcd.setCursor(0, 1);
      lcd.print("  ATTACK WIN");
      break;
      
    case DESARMADA:
      lcd.setCursor(0, 0);
      lcd.print("BOMBA DESARMADA!");
      lcd.setCursor(0, 1);
      lcd.print("  DEFENSE WIN  !");
      break;
  }
}

void mostrarPlacar() {
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("DEFENSE X ATTACK");
  lcd.setCursor(0, 1);
  lcd.print("    ");
  lcd.print(placarDefensores);
  lcd.print(" X ");
  lcd.print(placarAtacantes);
}

void limparLCD() {
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Aguardando...");
  lcd.setCursor(0, 1);
  lcd.print("Pressione ARMAR");
}
````

# Projeto no TinkerCAD

https://www.tinkercad.com/things/c0tn1UnBdFd-simulador-de-bomba

# Resultado Final❗

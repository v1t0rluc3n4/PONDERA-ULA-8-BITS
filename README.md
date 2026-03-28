#  (ULA) de 8 Bits

Este repositório documenta o projeto e a implementação em nível de portas lógicas (*Gate-Level*) dos principais módulos operacionais de uma Unidade Lógica e Aritmética (ULA). O projeto adota uma arquitetura rigorosa **Bottom-Up**, onde portas lógicas fundamentais abstraem componentes progressivamente mais complexos, culminando em matrizes combinacionais robustas para multiplicação e divisão.

---

##  Índice
- [(ULA) de 8 Bits](#ula-de-8-bits)
  - [Índice](#índice)
  - [1. Fundamentos Teóricos](#1-fundamentos-teóricos)
  - [2. Implementação Detalhada dos Módulos](#2-implementação-detalhada-dos-módulos)
    - [2.1. Somador Completo de 1 Bit (Full Adder)](#21-somador-completo-de-1-bit-full-adder)
    - [2.2. Somador de 8 Bits (Ripple Carry Adder)](#22-somador-de-8-bits-ripple-carry-adder)
    - [2.3. Módulo de Subtração (Lógica de Complemento de 2)](#23-módulo-de-subtração-lógica-de-complemento-de-2)
    - [2.4. Multiplicador Combinacional (Array Multiplier)](#24-multiplicador-combinacional-array-multiplier)
    - [2.5. Divisor Combinacional (Array Divider)](#25-divisor-combinacional-array-divider)
  - [3. Análise de Desempenho e Caminho Crítico](#3-análise-de-desempenho-e-caminho-crítico)
  - [4. Referências Bibliográficas](#4-referências-bibliográficas)

---

##  1. Fundamentos Teóricos

Para compreender a engenharia deste projeto, é essencial estabelecer os princípios da computação em baixo nível:

* **O Sistema Digital:** Diferente da eletrônica analógica, a eletrônica digital opera com valores discretos (binários). Utilizamos o estado **0 (Low/GND)** e o estado **1 (High/VCC)**. Essa abstração permite o processamento de informações com extrema confiabilidade através da Álgebra Booleana.
* **Lógica Aritmética:** Computadores não resolvem matemática de forma abstrata; eles roteiam sinais elétricos através de transistores organizados em portas lógicas (AND, OR, NOT, XOR). A soma de $1 + 1 = 10_2$, por exemplo, é resolvida fisicamente por uma porta XOR (que dita o bit $0$ da soma) e uma porta AND (que dita o bit $1$ do transporte/carry).


---

##  2. Implementação Detalhada dos Módulos

Nesta seção, dissecamos a engenharia por trás de cada circuito, abordando o fluxo de dados, as equações lógicas e as decisões de design arquitetural.

### 2.1. Somador Completo de 1 Bit (Full Adder)
O Somador Completo (`SOMA1BIT`) é a célula-tronco do processador. Diferente de um "Half Adder" (que soma apenas dois bits), o Full Adder possui três entradas: os operandos $A$ e $B$, e o transporte de entrada $C_{in}$ (*Carry-In*).

* **Análise Lógica:** A soma binária exige que o resultado seja $1$ caso haja um número ímpar de entradas em nível alto. O transporte ($C_{out}$) é acionado se duas ou mais entradas forem $1$.
* **Equações Booleanas Implementadas (Gate-Level):**
  * **Sinal de Saída (SUM):** Utiliza portas XOR em cascata.
    $$SUM = A \oplus B \oplus C_{in}$$
  * **Sinal de Transporte ($C_{out}$):** Utiliza portas AND e OR.
    $$C_{out} = (A \cdot B) + (C_{in} \cdot (A \oplus B))$$

![Circuito de Soma de 1 Bit](imagens/SOMA1BIT.png)

### 2.2. Somador de 8 Bits (Ripple Carry Adder)
Para realizar a soma de um byte completo ($A[7:0]$ e $B[7:0]$), 8 blocos de `SOMA1BIT` foram conectados em série, formando uma arquitetura de **Ripple Carry Adder (RCA)**.

* **Arquitetura de Propagação (Ripple):** O pino $C_{out}$ do Bit 0 (LSB) é conectado fisicamente ao pino $C_{in}$ do Bit 1, propagando o sinal como uma onda até o Bit 7 (MSB).
* **Mapeamento de Pesos Posicionais:** Os módulos refletem seus pesos na base 2 em notação decimal: `S0`, `S2`, `S4`, `S8`, `S16`, `S32`, `S64`, `S128`.
* **A Flag de OVER (Overflow):** O $C_{out}$ do último bloco (`S128`) é roteado para a saída `OVER`. Em somas sem sinal (*unsigned*), se este pino for a nível lógico $1$, indica que o resultado ultrapassou $255_{10}$ ($11111111_2$), excedendo a largura do barramento.

![Circuito de Soma de 8 Bits](imagens/SOMA8BIT.png)

### 2.3. Módulo de Subtração (Lógica de Complemento de 2)
Para otimizar o uso de transistores, o projeto reaproveita o hardware de soma para executar subtrações, aplicando o teorema matemático do **Complemento de 2**.

* **Fundamentação:** A operação $A - B$ é executada como $A + (-B)$. Em binário, a negação de um número consiste em inverter seus bits (Complemento de 1) e somar $1$.
* **Fluxo no Hardware:**
  1. A entrada $B$ passa por inversores (portas NOT), gerando o sinal interno `TBNOT`.
  2. O bit $+1$ necessário é injetado forçando o pino `CIN` do primeiro somador (`S0`) para o nível lógico $1$ (sinal `TCIN`).
* **Multiplexação:** Um MUX na saída, regido pelo sinal seletor `S`, define se a ULA fará bypass do dado `A` ($S=0$) ou se entregará o resultado da subtração `TSUB` ($S=1$).

![Circuito de Subtração](imagens/SUBTRAÇÃO.png)

### 2.4. Multiplicador Combinacional (Array Multiplier)
A multiplicação é realizada espacialmente, gerando produtos parciais paralelos, análogo ao método tradicional de multiplicação manual.

* **Produtos Parciais:** Como a multiplicação de bits únicos ($1 \cdot 1 = 1$, $1 \cdot 0 = 0$) é idêntica à tabela-verdade da porta AND, uma matriz de portas AND cruza cada bit do Multiplicador com cada bit do Multiplicando.
* **A Matriz Acumuladora:** Os sinais gerados alimentam uma matriz bidimensional de Somadores Completos. O design em "escada" realiza o deslocamento posicional implícito (*shift*). Somas propagam-se verticalmente e *Carrys* propagam-se horizontalmente até a extração do vetor final de produto.

![Circuito de Multiplicação](imagens/MULTIPLICADOR.png)

### 2.5. Divisor Combinacional (Array Divider)
O componente mais avançado do datapath. Utiliza a topologia de **Divisão Combinacional com Restauração (Restoring Array Divider)**.

* **Arquitetura Subtrai-e-Desloca:** Sem depender de laços de clock, a divisão longa é planificada no espaço físico. O circuito tenta subtrair parcelas do Divisor do Dividendo.
* **Resolução Lógica Condicional:**
  * **Sucesso:** Se a subtração parcial for não-negativa, o bit correspondente do Quociente (`Q`) recebe $1$, e o resto flui para a próxima linha.
  * **Falha e Restauração:** Se o resultado for negativo (indicando que "não cabe"), multiplexadores internos descartam a subtração, restauram o dividendo parcial para a próxima linha, e o Quociente recebe $0$.
* **Saída Dupla:** A malha converge gerando simultaneamente os barramentos de Quociente (`Q`) e Resto (`R`).

![Circuito de Divisão](imagens/DIVISOR.png)

---

##  3. Análise de Desempenho e Caminho Crítico

Como se trata de uma arquitetura estritamente combinacional, o desempenho máximo (frequência de operação teórica) é limitado pelo **Atraso de Propagação (Propagation Delay)**.
* **O Gargalo do Ripple Carry:** No Somador de 8 Bits, o MSB precisa aguardar o cálculo de todos os *carrys* anteriores. O caminho mais longo (*Critical Path*) vai da entrada $A[0]/B[0]$ até o pino `OVER`.
* **Escalabilidade:** Em implementações industriais modernas, problemas de propagação em matrizes de multiplicação/divisão são mitigados utilizando somadores do tipo *Carry Look-Ahead* (Antecipação de Transporte) e fragmentando os arrays com registradores de *Pipeline*.

---


##  4. Referências Bibliográficas
Materiais do projeto
Recursos utilizados diretamente na construção e documentação desta ALU:

Explicação ALU — Scribd Documento de referência sobre arquitetura e funcionamento de ALUs. https://pt.scribd.com/document/809087842/Explicacao-ALU

Binary Subtractor — Electronics Tutorials Teoria detalhada sobre subtratores binários e complemento de 2. https://www.electronics-tutorials.ws/combination/binary-subtractor.html

ALU — YouTube Tutorial 1 https://www.youtube.com/watch?v=-5N1FY8EC_8

ALU — YouTube Tutorial 2 https://www.youtube.com/watch?v=Wf_1mf6yCoc

ALU — YouTube Tutorial 3 https://www.youtube.com/watch?v=joHG5yaOW5I


# SoftPLC — Simulador de PLC em C

![C](https://img.shields.io/badge/C-00599C?style=for-the-badge&logo=c&logoColor=white)
![Platform](https://img.shields.io/badge/Platform-Windows-blue?style=for-the-badge&logo=windows)
![Status](https://img.shields.io/badge/Status-Em%20desenvolvimento-yellow?style=for-the-badge)
![License](https://img.shields.io/badge/License-MIT-yellow?style=for-the-badge)

**SoftPLC** é um núcleo de processamento lógico de baixo nível desenvolvido em **C puro**, projetado para emular o comportamento rigoroso de um **Software Programmable Logic Controller (SoftPLC)** — o "cérebro" de um sistema de automação industrial.

Este projeto não é apenas uma simulação de variáveis: é um motor que traduz a jornada física do dado — desde a leitura de corrente (**4-20mA**) de sensores simulados, passando por uma conversão Analógico-Digital (ADC), até a manipulação de registradores de 32 bits para o acionamento de atuadores via lógica de bits (*bitwise*).

<p align="center">
  <img src="assets/fluxograma.png" alt="Fluxograma do Sistema">
  <br>
  <em>O diagrama acima detalha a separação entre as threads de interface e a lógica de scan do PLC.</em>
</p>

---

## Índice

- [Tecnologias e Arquitetura](#tecnologias-e-arquitetura)
- [Instalação e Execução](#instalação-e-execução)
- [Análise Crítica e Limitações Arquiteturais](#análise-crítica-e-limitações-arquiteturais)
- [Processo de Desenvolvimento e Aprendizados](#processo-de-desenvolvimento-e-aprendizados)
- [Roadmap](#roadmap--futuras-melhorias)
- [Licença](#licença)

---

## Tecnologias e Arquitetura

| Recurso | Descrição |
|---|---|
| 📜 **Log de Eventos & Huffman** | Registro histórico de todas as decisões do controlador, comprimido automaticamente com o algoritmo de Huffman ao atingir 100kb. |
| 🔄 **Ciclo de Scan Industrial** | Execução ininterrupta da lógica de controle em malha fechada: leitura de entradas, processamento de interrupções e atualização de saídas. |
| 🌳 **Árvore AVL** | Estrutura balanceada usada para registro e busca de sensores/atuadores, garantindo tempos de resposta consistentes. |
| 🔢 **Lógica Bitwise Multivariável** | Decisões processadas via registradores de 32 bits, permitindo que múltiplos atuadores respondam simultaneamente a diferentes condições de sensores. |
| 🧵 **Multithreading** | Separação total entre a camada de controle (Kernel) e a camada de visualização (IHM) via Mutexes e POSIX Threads, evitando que interrupções de usuário bloqueiem o processamento industrial. |
| 🖥️ **Interface estática** | Atualiza os valores continuamente na mesma tela, facilitando a percepção de mudanças ao longo do tempo. |

---

## Instalação e Execução

### Pré-requisitos
- Compilador **GCC** (MinGW recomendado para Windows)
- Suporte a **pthreads** (nativo no MinGW)
- Testado no Windows 10

### Passo a passo

```bash
# 1. Clone o repositório
git clone https://github.com/DaviReder/softplc.git
cd softplc

# 2. Compile o projeto
gcc main.c src/*.c -I include -lpthread -o softplc

# 3. Execute o simulador
./softplc
```

---

## Análise Crítica e Limitações Arquiteturais

Como este projeto tem caráter fortemente acadêmico e foco no domínio prático de estruturas de dados, alguns *trade-offs* de design foram feitos deliberadamente — e seriam tratados de forma diferente em produção:

**1. Concorrência e gargalo de I/O de tela**
Na camada de interface (`main.c`), as chamadas de exibição (`printf`) ocorrem dentro da região crítica protegida pelo Mutex da árvore AVL. Em um RTOS real, I/O de tela é lento e bloqueante, o que introduziria *jitter* inaceitável no ciclo de scan. Uma abordagem comercial usaria *double-buffering*: copiar os dados da AVL para uma estrutura espelho em RAM e liberar o Mutex antes de renderizar.

**2. Árvore AVL vs. arrays estáticos**
Usar uma Árvore AVL para indexar sensores foi uma decisão didática, para demonstrar controle de algoritmos auto-balanceáveis em C puro. Num PLC real, o número de canais de I/O é fixo e conhecido em tempo de compilação — `malloc` é evitado em sistemas críticos pelo risco de fragmentação de memória e imprevisibilidade de tempo de execução. Em produção, a estrutura ideal seria um array estático ou tabela hash direta (O(1)).

**3. Acoplamento e portabilidade**
O projeto mistura a API nativa de Console do Windows com POSIX threads (via MinGW pthreads). Para portabilidade real (Linux/macOS), a camada gráfica e de threads deveria ser isolada numa Camada de Abstração de Sistema Operacional (OSAL) com diretivas condicionais (`#ifdef _WIN32`).

---

## Processo de Desenvolvimento e Aprendizados

### Por que essas decisões arquiteturais

Este projeto nasceu para validar meu domínio sobre estruturas de dados e programação de baixo nível — não para chegar à arquitetura mais eficiente possível. Prova disso é o uso de uma Árvore AVL para indexar apenas 4-5 sensores: seria overkill em qualquer cenário real, mas o objetivo era me desafiar a implementar e entender a lógica de balanceamento na prática, não escolher a ferramenta "certa" para o problema.

Reconheço cada decisão tomada. Nem todas fazem sentido para um PLC real, mas outras seguem sendo um bom ponto de partida:

- **Sistema de logs**, com compressão via Huffman já prevendo o volume de dados que seria gerado em produção
- **O ciclo de scan** que simula o comportamento de um PLC real
- **A manipulação bitwise**, que é uma das jogadas fundamentais para maximizar eficiência de equipamentos de campo

Onde o projeto realmente fica devendo é no tratamento de concorrência do loop de scan: falta um mecanismo mais robusto de sincronização para garantir segurança em acessos múltiplos e simultâneos às mesmas funções e variáveis.

### Dificuldades técnicas enfrentadas

Essa foi minha primeira implementação de uma Árvore AVL do zero, e o processo de tentativa e erro deixou marcas visíveis no código, que podem ter passado despercebido por mim:

- Erros de indexação
- Ponteiros nulos não tratados
- Acessos indevidos de memória
- Provavelmente alguns vazamentos que não identifiquei (o código nunca foi auditado com Valgrind ou ferramenta equivalente)

Ponteiros, de forma geral, foram o maior obstáculo do projeto inteiro.

### O que ficou de fato como ganho

Com refatoração, este projeto tem potencial real para virar a base de um simulador de PLC fiel — e, no limite, rodar dentro de hardware real. Mas o ganho mais concreto não foi o código em si: foi a capacidade de análise crítica de arquitetura que desenvolvi ao longo do processo, e entender na prática erros clássicos que eu já conhecia em teoria, mas nunca tinha cometido com as próprias mãos.

### Sobre uso de IA

IA foi usada apenas para ajustar a apresentação visual do terminal — nada na lógica, nas estruturas de dados ou nos algoritmos do projeto. Ainda não tenho uma solução elegante para essa parte da interface e sigo buscando melhorá-la.

---

## Roadmap & Futuras Melhorias

- [ ] **PID Controller** — algoritmo de controle Proporcional-Integral-Derivativo discreto, substituindo a lógica atual baseada em histerese.
- [ ] **Comunicação via Sockets** — evolução do motor para escutar/responder requisições de rede via TCP/UDP, emulando uma variante simplificada do Modbus TCP.
- [ ] **Refatoração & Sanity Checks** — varredura de gerenciamento de memória com Valgrind ou Dr. Memory, garantindo ausência de memory leaks.

---

## Licença

Distribuído sob a licença MIT.

---

> 🎓 **Sobre o autor:** Estudante de Engenharia de Controle e Automação na PUC Minas.
>
> O intuito deste projeto é puramente didático — servindo como base para fortificar o aprendizado em estruturas de dados complexas e otimização de software aplicada à automação industrial. Sinta-se à vontade para explorar e sugerir melhorias!

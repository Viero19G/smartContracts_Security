# ====================================
# 04-DIAGNOSTICO_TECNICO.md
# ====================================

# Diagnóstico Técnico Detalhado
## PredictTheFutureChallenge.sol

---

## 1. Arquitetura do Contrato

### Diagrama de Estados

```
┌─────────────────────────────────────────────────┐
│    CONTRATO VAZIO                               │
│  (1 ETH depositado no construtor)               │
│  guesser = 0x0                                  │
│  guess = 0                                      │
│  settlementBlockNumber = 0                      │
└─────────────────────────────────────────────────┘
              │
              │ lockInGuess(n)
              │ (1 ETH depositado)
              ▼
┌─────────────────────────────────────────────────┐
│  JOGO ATIVO                                      │
│  guesser = msg.sender                           │
│  guess = n                                      │
│  settlementBlockNumber = block.number + 1       │
│  saldo = 2 ETH                                  │
│  ⏱️ Espera por próximo bloco                     │
└─────────────────────────────────────────────────┘
              │
              │ settle()
              │ (após 1+ bloco)
              ▼
        ┌──────────────┐
        │              │
     SIM│              │NÃO
       guess==│       │
      answer  │       │
        │     │       │
        ▼     ▼       ▼
    ┌────┐┌────┐    ┌──────────┐
    │WIN ││LOSE│    │ LOSS     │
    │2ETH││  - │    │ bloqueado│
    └────┘└────┘    └──────────┘
```

### Fluxo de Transações

```
Ator               Função              Precondições         Efeito
─────────────────────────────────────────────────────────────────────
Qualquer um   constructor()       msg.value == 1 ETH    Inicializa contrato
              (via deploy)        (não chamável depois)  com 1 ETH

Jogador A     lockInGuess(5)      guesser == 0          Define guesser=A
              + 1 ETH            msg.value == 1 ETH    guess=5
                                                       settlementBlockNumber
                                                       = bloco atual + 1

Qualquer um   settle()            msg.sender == A       Calcula resposta
              (bloco futuro)      block.number > SBN    Se correto: A saca
                                                       2 ETH
                                                       Se errado: contrato
                                                       retém 2 ETH

Qualquer um   isComplete()        -                     Retorna True se
                                                       saldo == 0
```

---

## 2. Fluxo de Fundos

### Movimentação de ETH

```
ENTRADA:
├── Construtor: 1 ETH (criador do contrato)
└── lockInGuess(): 1 ETH por participante
    Total em Jogo: 2 ETH

SAÍDA (apenas em condições específicas):
└── settle() com vitória: 2 ETH → msg.sender (guesser)
    
PRISIONEIRO (sem condição de saque):
└── settle() com derrota: 2 ETH permanecem no contrato
    (nenhuma função permite resgate)
```

---

## 3. Matriz de Risco

### Tabela: Componente × Vulnerabilidade → Severidade

| Componente | RNG | Blockhash | Underflow | Validação | Compiler |
|---|---|---|---|---|---|
| `lockInGuess()` | - | - |  MÉDIA |  CRÍTICA |  ALTA |
| `settle()` |  CRÍTICA |  CRÍTICA |  ALTA | - |  ALTA |
| `isComplete()` | - | - | - | - | - |
| Estado |  CRÍTICA |  ALTA | - | - | - |

---

## 4. Cenários de Ataque

### Ataque 1: Previsão Determinística 
```
1. Hacker monitora mempool
2. Vê: lockInGuess(n) em bloco B
3. Calcula: answer = keccak256(blockhash(B), timestamp[B+1]) % 10
4. Faz: lockInGuess(answer) no mesmo bloco B
5. Resultado: Vitória 100% garantida
   Roubo: 1 ETH por exploração
```

### Ataque 2: Blockage Timeout 
```
1. Participante honesto chama lockInGuess() no bloco 1000
2. Participante aguarda resolução
3. Hacker não chama settle()
4. Após 256 blocos (bloco 1256+), blockhash expira
5. Se settle() é chamado: revert ou resultado errado
6. Resultado: 2 ETH presos indefinidamente
```

### Ataque 3: Front-Running 
```
1. Participante vê transação pendente: lockInGuess(5)
2. Mempool revela n=5
3. Hacker envia lockInGuess(previsto_5) com gas maior
4. Hacker's tx executa primeiro
5. Hacker garante vitória antes do participante honesto
```

---

## 5. Recomendações Operacionais

1. ** NÃO FAZER DEPLOY EM MAINNET**
2. Se já em testnet/produção: **DESTRUIR CONTRATO** e se ja estiver "deployado", abandonar e não utilizar mais.
3. Avisar comunidade e users

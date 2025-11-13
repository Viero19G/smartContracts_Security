# ====================================
# 02-RELATORIO_VULNERABILIDADES.md
# ====================================

# Relatório Detalhado de Vulnerabilidades
## PredictTheFutureChallenge.sol

---

## VULN-2025-001: Randomness Manipulation (Previsibilidade Total)

**Severidade:**  **CRÍTICA**  
**Confiança:** ALTO  
**Local:** Função `settle()`, linha 28  

### Descrição Detalhada

O contrato usa `keccak256(block.blockhash(block.number - 1), now)` para gerar o número "aleatório". Isso é **completamente previsível** porque:

1. **`block.blockhash(block.number - 1)` é público:** Qualquer pessoa pode ler o hash do bloco anterior no mempool ou após sua mineração.
2. **`now` (alias para `block.timestamp`) é público:** Todos os validadores conhecem o timestamp do bloco.
3. **O cálculo é determinístico:** Dado esses inputs públicos, qualquer pessoa pode calcular o resultado antes de `settle()` ser executado.

A função `settle()` não contém nenhum elemento secreto ou comprometimento criptográfico que impediria um atacante de calcular a resposta correta.

### Condições para Exploração

**Pré-requisitos:**
- Acesso ao hash do bloco anterior e timestamp (ambos públicos na blockchain)
- Capacidade de fazer uma chamada `lockInGuess()` antes de `settle()`

**Passos para exploração:**
1. Monitorar o contrato até que um participante faça um `lockInGuess()`
2. Ler `block.number`, `block.timestamp` do bloco em que `lockInGuess()` foi executado
3. Calcular `settlementBlockNumber = block.number + 1`
4. Esperar até que `block.number > settlementBlockNumber`
5. Calcular `answer = uint8(keccak256(block.blockhash(settlementBlockNumber - 1), timestamp[settlementBlockNumber])) % 10`
6. Fazer um `lockInGuess()` com o valor `answer` calculado
7. Chamar `settle()` — garantia de vitória

### Impacto

- **Econômico:** Perda total de fundos de qualquer participante honesto
- **Integridade:** Contrato não funciona como proposto (loteria justa)
- **Disponibilidade:** Contrato se torna inútil; sempre será explorado
- **Confiança:** Usuários perdem confiança no sistema

### Sugestão de Correção

**Opção 1: Usar Chainlink VRF**

```solidity
import "@chainlink/contracts/src/v0.8/vrf/VRFConsumerBase.sol";

contract PredictTheFutureCorrected is VRFConsumerBase {
    bytes32 keyHash = 0x...; // Configurado para mainnet/testnet
    uint256 fee = 0.1 * 10 ** 18; // 0.1 LINK
    
    mapping(bytes32 => address) requestIdToPlayer;
    mapping(bytes32 => uint8) requestIdToGuess;

    function lockInGuess(uint8 n) external payable {
        require(n < 10, "Guess must be 0-9");
        require(msg.value == 1 ether);
        
        bytes32 requestId = requestRandomness(keyHash, fee);
        requestIdToPlayer[requestId] = msg.sender;
        requestIdToGuess[requestId] = n;
    }

    function fulfillRandomWords(uint256 randomness) internal override {
        uint8 answer = uint8(randomness) % 10;
        address player = requestIdToPlayer[requestId];
        uint8 guess = requestIdToGuess[requestId];

        if (guess == answer) {
            payable(player).transfer(2 ether);
        }
    }
}
```

**Opção 2: Commit-Reveal Scheme (sem oráculo menos recomendavel)**

```solidity
// Fase 1: Hacker faz commit
mapping(bytes32 => uint256) commits;
function commit(bytes32 hash) external payable {
    require(msg.value == 1 ether);
    commits[hash] = block.timestamp;
}

// Fase 2: Após N blocos, revelar
function reveal(uint256 numero, bytes32 salt) external {
    bytes32 hash = keccak256(abi.encodePacked(numero, salt));
    require(commits[hash] > 0);
    require(block.timestamp >= commits[hash] + 1 days);
    
    // Número é revelado APÓS commit, impedindo manipulação
    // ...
}
```

---

## VULN-2025-002: Block Hash Access Out of Range

**Severidade:**  **CRÍTICA**  
**Confiança:** ALTO  
**Local:** Função `settle()`, linha 28  

### Descrição Detalhada

O EVM só mantém o hash dos últimos **256 blocos**. Se `settle()` for chamado mais de 256 blocos após `lockInGuess()`:
- `block.blockhash(block.number - 1)` retorna `0x0000000000...` (zero)
- O cálculo fica incorreto e o contrato não consegue resolver o jogo adequadamente
- Se o hacker conhecer essa limitação, pode explorar fazendo `settle()` após 256 blocos com um número que resulte em 0

### Impacto

- **Disponibilidade:** Contrato quebra após 256 blocos (~1 hora na Ethereum mainnet com 12s/bloco)
- **Integridade:** Participantes não conseguem resolver o jogo legitimamente
- **Econômico:** Bloqueio de fundos (2 ETH presos)

### Sugestão de Correção

**Adicionar validação de 256 blocos:**

```solidity
function settle() public {
    require(msg.sender == guesser);
    require(block.number > settlementBlockNumber);
    require(block.number <= settlementBlockNumber + 256); // NOVO
    
    uint8 answer = uint8(keccak256(block.blockhash(block.number - 1), now)) % 10;
    // ...
}
```

---

## VULN-2025-003: Integer Underflow em `block.number - 1`

**Severidade:**  **ALTA** (teórica)  
**Confiança:** MÉDIO  
**Local:** Função `settle()`, linha 28  

### Descrição Detalhada

Embora `block.number` nunca seja 0 em produção (começa em 1), a operação `block.number - 1` em Solidity 0.4.21 **não tem proteção automática** contra underflow.

Esta é uma **violação de boas práticas** de segurança defensiva.

### Sugestão de Correção

**Usar Solidity 0.8.0+ (proteção automática):**

```solidity
pragma solidity ^0.8.20;
// Solidity 0.8.0+ reverte automaticamente em overflow/underflow
```

---

## VULN-2025-004: Falta de Validação de Input

**Severidade:**  **ALTA**  
**Confiança:** ALTO  
**Local:** Função `lockInGuess()`, linha 19  

### Descrição Detalhada

A função aceita qualquer valor `uint8` (0-255), mas a resposta só pode ser 0-9.

Um usuário pode fazer `lockInGuess(255)` e **NUNCA vencer**, pois `255 % 10 = 5`, mas o usuário esperava 255.

**Impacto:**
- Usuários confundidos perdem ETH permanentemente
- Vulnerabilidade de UX/segurança do usuário

### Sugestão de Correção

```solidity
function lockInGuess(uint8 n) public payable {
    require(n < 10, "Guess must be between 0 and 9"); //  VALIDAÇÃO
    require(guesser == 0);
    require(msg.value == 1 ether);
    // ...
}
```

---

## VULN-2025-005: Solidity 0.4.21 Desatualizado

**Severidade:**  **MÉDIA**  
**Confiança:** ALTO  
**Local:** Linha 1 (pragma)  

### Descrição Detalhada

Solidity 0.4.21:
- **EOL desde fevereiro de 2019**
- Não inclui proteção automática contra overflow/underflow
- Não suporta features modernas de segurança
- Ferramentas de auditoria têm suporte limitado

### Sugestão de Correção

**Atualizar para Solidity 0.8.20+ (LTS):**

```solidity
pragma solidity ^0.8.20;
```

Ativa:
-  Proteção automática contra overflow/underflow
-  Melhor suporte a testes e auditorias
-  Otimizações de gas
-  Sintaxe mais segura

---

## Sumário de Vulnerabilidades

| ID | Título | Severidade | Confiança | Remediação |
|---|---|---|---|---|
| VULN-2025-001 | Randomness Manipulation |  CRÍTICA | ALTO | Chainlink VRF ou Commit-Reveal |
| VULN-2025-002 | Block Hash Out of Range |  CRÍTICA | ALTO | Validar <= 256 blocos |
| VULN-2025-003 | Integer Underflow |  ALTA | MÉDIO | Usar Solidity 0.8.0+ |
| VULN-2025-004 | Validação de Input |  ALTA | ALTO | Adicionar require(n < 10) |
| VULN-2025-005 | Compilador Desatualizado |  MÉDIA | ALTO | Atualizar para 0.8.20+ |

**Total:** 5 vulnerabilidades (2 críticas, 2 altas, 1 média)

---
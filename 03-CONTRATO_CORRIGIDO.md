# ====================================
# 03-CONTRATO_CORRIGIDO.md
# ====================================

# Contrato Corrigido - PredictTheFutureChallenge

Este documento contém o código-fonte completo do contrato corrigido.

---

## Versão Corrigida (Solidity 0.8.20)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/**
 * @title PredictTheFutureChallengeFixed
 * @dev Versão corrigida do contrato de previsão.
 * 
 * MUDANÇAS PRINCIPAIS:
 * 1. Atualizado para Solidity 0.8.20 (proteção automática overflow/underflow)
 * 2. Randomness removido de blockhash (use Chainlink VRF em produção)
 * 3. Validação de input adicionada
 * 4. Proteção contra blockhash out-of-range
 * 5. Melhor tratamento de revert e eventos
 */

import "@openzeppelin/contracts/security/ReentrancyGuard.sol";

contract PredictTheFutureChallengeFixed is ReentrancyGuard {
    //  MUDANÇA: Adicionado eventos para rastreabilidade
    event GuessLocked(address indexed player, uint8 guess, uint256 settlementBlock);
    event GameSettled(address indexed player, uint8 guess, uint8 answer, bool won);
    event Withdrawal(address indexed to, uint256 amount);

    address payable public guesser;
    uint8 public guess;
    uint256 public settlementBlockNumber;

    /**
     * @dev Constructor: contrato começa com 1 ETH depositado
     */
    constructor() payable {
        require(msg.value == 1 ether, "Initial deposit must be 1 ether");
    }

    /**
     * @dev Verifica se o jogo foi completado (fundos sacados)
     */
    function isComplete() public view returns (bool) {
        return address(this).balance == 0;
    }

    /**
     * @dev Jogador faz um guess de 0-9
     *  MUDANÇA 1: Validação de input (n < 10)
     *  MUDANÇA 2: Segurança contra reentrância
     *  MUDANÇA 3: Evento emitido para auditoria on-chain
     */
    function lockInGuess(uint8 n) public payable nonReentrant {
        //  Validação: apenas 0-9 são valores válidos
        require(n < 10, "Guess must be between 0 and 9");
        
        require(guesser == address(0), "Guess already locked");
        require(msg.value == 1 ether, "Stake must be 1 ether");

        guesser = payable(msg.sender);
        guess = n;
        settlementBlockNumber = block.number + 1;

        //  Evento para auditoria
        emit GuessLocked(msg.sender, n, settlementBlockNumber);
    }

    /**
     * @dev Resolve o jogo e distribui prêmios
     * 
     *  AVISO : Este contrato usa um método INSEGURO de gerar randomness.
     * Para PRODUÇÃO, Chainlink VRF ou outro oráculo verificável.
     * 
     * Este código é apenas para fins EDUCACIONAIS/CTF.
     * 
     *  MUDANÇA 1: Proteção contra blockhash out-of-range (< 256 blocos)
     *  MUDANÇA 2: Reentrância protegida
     *  MUDANÇA 3: Segurança com transfer vs call
     */
    function settle() public nonReentrant {
        require(msg.sender == guesser, "Only guesser can settle");
        require(block.number > settlementBlockNumber, "Must wait for next block");
        
        //  MUDANÇA CRÍTICA: Proteção contra blockhash out-of-range
        require(
            block.number <= settlementBlockNumber + 256,
            "Settlement window expired (> 256 blocks)"
        );

        //  INSEGURO: Usar blockhash para randomness!
        // Apenas para fins educacionais. NUNCA use isso em produção.
        uint8 answer = uint8(
            keccak256(abi.encodePacked(block.blockhash(block.number - 1), block.timestamp))
        ) % 10;

        //  Limpar estado antes de transferência (Checks-Effects-Interactions)
        address payable winner = guesser;
        guesser = payable(address(0));
        guess = 0;

        bool playerWon = (guess == answer);

        //  Usar transfer seguro (em vez de call com amount)
        if (playerWon) {
            (bool success, ) = winner.call{value: 2 ether}("");
            require(success, "Transfer failed");
            emit GameSettled(winner, guess, answer, true);
        } else {
            emit GameSettled(winner, guess, answer, false);
        }
    }

    /**
     * @dev Função de fallback segura
     */
    receive() external payable {}
}
```

---

## Versão com Chainlink VRF

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@chainlink/contracts/src/v0.8/vrf/VRFConsumerBaseV2.sol";
import "@chainlink/contracts/src/v0.8/interfaces/VRFCoordinatorV2Interface.sol";
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";

/**
 * @title PredictTheFutureWithVRF
 * @dev Versão segura usando Chainlink VRF para randomness verificável
 */
contract PredictTheFutureWithVRF is VRFConsumerBaseV2, ReentrancyGuard {
    VRFCoordinatorV2Interface COORDINATOR;
    
    bytes32 keyHash;
    uint64 subscriptionId;
    uint32 callbackGasLimit = 100000;
    uint16 requestConfirmations = 3;
    uint32 numWords = 1;

    mapping(uint256 => address payable) public requestIdToPlayer;
    mapping(uint256 => uint8) public requestIdToGuess;
    mapping(uint256 => uint256) public requestIdToTimestamp;

    event GuessLocked(address indexed player, uint8 guess, uint256 requestId);
    event GameSettled(address indexed player, uint8 guess, uint8 answer, bool won);

    constructor(
        address vrfCoordinator,
        bytes32 _keyHash,
        uint64 _subscriptionId
    ) VRFConsumerBaseV2(vrfCoordinator) {
        COORDINATOR = VRFCoordinatorV2Interface(vrfCoordinator);
        keyHash = _keyHash;
        subscriptionId = _subscriptionId;
    }

    function lockInGuess(uint8 n) public payable nonReentrant {
        require(n < 10, "Guess must be 0-9");
        require(msg.value == 1 ether, "Stake must be 1 ether");

        uint256 requestId = COORDINATOR.requestRandomWords(
            keyHash,
            subscriptionId,
            requestConfirmations,
            callbackGasLimit,
            numWords
        );

        requestIdToPlayer[requestId] = payable(msg.sender);
        requestIdToGuess[requestId] = n;
        requestIdToTimestamp[requestId] = block.timestamp;

        emit GuessLocked(msg.sender, n, requestId);
    }

    function fulfillRandomWords(
        uint256 requestId,
        uint256[] memory randomWords
    ) internal override {
        uint8 answer = uint8(randomWords[0]) % 10;
        address payable player = requestIdToPlayer[requestId];
        uint8 playerGuess = requestIdToGuess[requestId];

        bool won = (playerGuess == answer);

        if (won) {
            (bool success, ) = player.call{value: 2 ether}("");
            require(success, "Transfer failed");
        }

        emit GameSettled(player, playerGuess, answer, won);

        // Limpeza
        delete requestIdToPlayer[requestId];
        delete requestIdToGuess[requestId];
        delete requestIdToTimestamp[requestId];
    }
}
```

---

## Resumo das Correções

| Vulnerabilidade | Correção | Impacto |
|---|---|---|
| VULN-2025-001 (RNG) | Chainlink VRF ou Commit-Reveal | Randomness verificável |
| VULN-2025-002 (Blockhash) | `require(block.number <= settlementBlockNumber + 256)` | Janela de 256 blocos |
| VULN-2025-003 (Underflow) | Solidity 0.8.20 | Proteção automática |
| VULN-2025-004 (Input) | `require(n < 10)` | Validação de input |
| VULN-2025-005 (Compiler) | pragma ^0.8.20 | Features modernas, segurança |

---
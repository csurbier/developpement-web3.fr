---
layout: '../../layouts/BLogWeb3.astro'

title: "Angular et web3.js"
description: "Tutoriel sur comment utiliser la librairie web3.js avec Angular."
blogPost: "Initiation utilisation de web3.js avec Angular"
image: true
canonical: "https://www.developpement-web3.fr/web3/angular-et-web3js/"
---

## Installation

Dans ce simple tutoriel, nous allons voir comment intégrer et utiliser **Web3.js** dans un projet **Angular (version 14)**. 

Vous pouvez retrouver le code source de ce tutoriel sur mon [Github](https://github.com/csurbier/tutoriels-developpement-web3/tree/main/angular-web3)

Tout d'abord créons notre projet angular avec la commande:

```bash
 ng new web3angularfrontend
? Would you like to add Angular routing? Yes
? Which stylesheet format would you like to use? CSS
```

Puis dans le nouveau répertoire nouvellement crée, on installe **web3.js**:

```bash
npm install -s web3
```

Et enfin on peut lancer notre site afin de vérifier que tout fonctionne

```bash
ng serve
```

## Détection et connexion à notre wallet Metamask

Dans le fichier **app.component.html**, nous allons remplacer tout le contenu de la balise *content* par :
```html
<div class="content" role="main">
  <!-- Footer -->
  <footer>
    <a  (click)="connectWallet()" > Connect your wallet to register </a>
  </footer>
</div>
 ```

Il s'agit d'un simple lien qui va appeler une méthode **connectWallet**. 

Pour utiliser la librairie **web3**, il nous faut d'abord l'importer. Modifions le fichier **app.component.ts** ainsi:

```typescript
import { Component } from '@angular/core';
import Web3 from 'web3';
declare const window: any; 

@Component({
  selector: 'app-root',
  templateUrl: './app.component.html',
  styleUrls: ['./app.component.css']
})
export class AppComponent {
  title = 'web3angularfrontend';
  connectedAddress : string = ""
  walletConnected : boolean = false ;
  hasWallet = false 
  web3 : any 

  constructor(){
    if (window.ethereum){
      this.hasWallet = true
    }
    else {
        this.hasWallet = false 
    }
   console.log(this.hasWallet)
  }
}
```

Pour savoir si l'utilisateur est connecté avec un navigateur contenant une extension web3 (Metamask ou autre), il suffit de regarder si l'objet **windows** de ce dernier contient une propriété **ethereum**. 

*Note: Si ce n'est pas le cas, il est judicieux d'inviter l'utilisateur a installer l'extension Metamask.*

Partons du postulat que notre navigateur contient bien l'extension Metasmask, pour lancer la connexion à notre wallet et récupérer l'adresse du wallet, il suffit d'écrire:

```typescript
 async connectWallet(){
     
    let address = await window.ethereum.request({ method: 'eth_requestAccounts' });
    if (address.length>0){
          this.connectedAddress = address[0];
          this.walletConnected = true 
          this.web3 = new Web3(Web3.givenProvider)
          console.log('address', address);
    }
    else{
      console.log("===Pas d'adresse ")
    }
  }
```
Soit l'utilisateur s'est déjà connecté à son wallet Metamask récemment et alors vous obtiendrez directement l'addresse du compte dans la console javascript, soit la popup de connexion Metamask va apparaître et vous demander de saisir votre mot de passe puis, l'adresse du compte sera affichée dans la console javascript.

## Création d'un service pour gérer les interactions Web3

Améliorons maintenant notre code en créant un service qui gérera toutes les interactions **web3**:

```bash
ng generate service Web3Service 
mkdir src/app/services
mv src/app/web3-service.service.* src/app/services 
```

Puis éditons le fichier **web3-service.service.ts** afin de déplacer tout notre code précedent dans ce service afin de l'adapter légèrement:
```typescript
import { Injectable } from '@angular/core';
import Web3 from 'web3';
declare const window: any;
@Injectable({
  providedIn: 'root'
})
export class Web3ServiceService {
  connectedAddress: string = ""
  walletConnected: boolean = false;
  hasWallet = false
  web3: any

  constructor() {
    if (window.ethereum) {
      this.hasWallet = true
    }
    else {
      this.hasWallet = false
    }
    console.log(this.hasWallet)
  }

  async getAccount() {
    return new Promise(async resolve => {
      let address = await window.ethereum.request({ method: 'eth_requestAccounts' });
      if (address.length > 0) {
        this.connectedAddress = address[0];
        this.walletConnected = true
        // Connexion Web3 avec le provider fournit automatiquement
        this.web3 = new Web3(Web3.givenProvider)
        console.log('address', address);
        resolve(true)
      }
      else {
        console.log("===Pas d'adresse ")
        resolve(false)
      }
    })
  }
}
```
et maintenant modifions notre fichier **app.component.ts** pour utiliser ce service:

```typescript
import { Component } from '@angular/core';
import { Web3ServiceService } from './services/web3-service.service';

@Component({
  selector: 'app-root',
  templateUrl: './app.component.html',
  styleUrls: ['./app.component.css']
})
export class AppComponent {
  title = 'web3angularfrontend';
 

  constructor(private web3Service:Web3ServiceService){
  
  }
 
  async connectWallet(){
    this.web3Service.getAccount().then((connecte)=>{
      if (connecte){
        console.log("Connecte avec adresse ",this.web3Service.connectedAddress)
      }
    })
  }
  
}
````

## Connaître la blockchain connectée

Pour déterminer sur quel blockchain (Ethereum, Polygon, Goerli,...) le wallet de l'utilisateur est connecté, on peut utiliser la méthode **getId()** de la librairie **Web3.js**. 

Écrivons:

```typescript
 public  checkCurrentChainId = async () => {
    return new Promise(async resolve => {
      let networkId = await this.web3.eth.net.getId()
      resolve(networkId)
    })
  }
```
et 
```typescript
 async connectWallet(){
    this.web3Service.getAccount().then((connecte)=>{
      if (connecte){
        console.log("Connecte avec adresse ",this.web3Service.connectedAddress)
        this.web3Service.checkCurrentChainId().then((chainId)=>{
          console.log("Chainid ",chainId)
        })
      }
    })
  }
````
En fonction de la blockchain(réseau) sur laquelle votre Metamask est connecté, vous verrez apparaître dans la console javascript un identifiant. Dans mon cas:
```javascript
Chainid  1
```
ce qui correspond à la blockchain Ethereum. Vous pouvez retrouver les identifiants en fonction des blockchains sur ce site [ChainList](https://chainlist.org/).

Voilà pour notre courte introduction au développement web3 avec Angular. 

Dans notre prochain tutoriel, nous continuerons à explorer l'utilisation de la librairie **web3.js** et verrons comment détecter le changement d'adresse de compte Metamask, de blockchain et aussi comment demander l'ajout d'une blockchain dans la configuration Metamask.

Vous pouvez aussi soutenir ce site en envoyant de l'ETH ou du MATIC:

<img style="width: 100px;" src="/img/qr-code.png">

<style>
    p { color: white;!important }
    a { color:darkorange!important}
  </style>
# Smart contract

Précédemment nous avons vu tous les éléments que pouvait posséder un fichier Solidity.
Les éléments les plus intéressants sont évidement les smart contracts qui composent ce fichier.
Ils sont définit avec le keyword `contract`.

## Layout order in a smart contract

Comme précisé dans le [style guide](https://docs.soliditylang.org/en/latest/style-guide.html#order-of-layout) l'ordre des éléments dans un smart contract suit une convention.

Il faudra déclarer les différents éléments d'un smart contract avec l'ordre suivant:

1. Library usage
2. Type declarations
3. State variables
4. Events
5. constructor
6. Function modifiers
7. Functions

## SmartWallet

Les smart contracts ci dessous utilisent tous les éléments que vous pouvez utiliser dans un smart contract.  
Néanmoins ces smart contracts n'utilisent pas d'`enum` et de `struct`.

_Ownable.sol_:

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.0;

contract Ownable {
    address private _owner;
    constructor(address owner_) {
        _owner = owner_;
    }

    modifier onlyOwner() {
        require(msg.sender == _owner, "Ownable: Only owner can call this function");
        _;
    }

    function owner() public view returns(address) {
        return _owner;
    }
}
```

_SmartWallet.sol_:

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.0;

// Pour remix il faut importer une url depuis un repository github
// Depuis un project Hardhat ou Truffle on utiliserait: import "@openzeppelin/ccontracts/utils/Address.sol";
import "https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/utils/Address.sol";
import "./Ownable.sol";


contract SmartWallet is Ownable {
    // library usage
    using Address for address payable;

    // State variables
    mapping(address => uint256) private _balances;
    uint256 private _tax;
    uint256 private _profit;
    uint256 private _totalProfit;

    // Events
    event Deposited(address indexed sender, uint256 amount);
    event Withdrew(address indexed recipient, uint256);
    event Transfered(address indexed sender, address indexed recipient, uint256 amount);

    // constructor
    constructor(address owner_, uint256 tax_) Ownable(owner_) {
        require(tax_ >= 0 && tax_ <= 100, "SmartWallet: Invalid percentage");
        _tax = tax_;
    }

    // modifiers
    // Le modifier onlyOwner a été défini dans le smart contract Ownable


    // Function declarations below
    receive() external payable {
        _deposit(msg.sender, msg.value);
    }

    fallback() external {

    }

    function deposit() external payable {
        _deposit(msg.sender, msg.value);
    }


    function withdraw() public {
        uint256 amount = _balances[msg.sender];
        _withdraw(msg.sender, amount);
    }

    function withdrawAmount(uint256 amount) public {
        _withdraw(msg.sender, amount);
    }

    function transfer(address recipient, uint256 amount) public {
        require(_balances[msg.sender] > 0, "SmartWallet: can not transfer 0 ether");
        require(_balances[msg.sender] >= amount, "SmartWallet: Not enough Ether to transfer");
        require(recipient != address(0), "SmartWallet: transfer to the zero address");
        _balances[msg.sender] -= amount;
        _balances[recipient] += amount;
        emit Transfered(msg.sender, recipient, amount);
    }

    function withdrawProfit() public onlyOwner {
        require(_profit > 0, "SmartWallet: can not withdraw 0 ether");
        uint256 amount = _profit;
        _profit = 0;
        payable(msg.sender).sendValue(amount);
    }

    function setTax(uint256 tax_) public onlyOwner {
        require(tax_ >= 0 && tax_ <= 100, "SmartWallet: Invalid percentage");
        _tax = tax_;
    }

    function balanceOf(address account) public view returns (uint256) {
        return _balances[account];
    }


    function total() public view returns (uint256) {
        return address(this).balance;
    }

    function tax() public view returns (uint256) {
        return _tax;
    }

    function profit() public view returns(uint256) {
        return _profit;
    }

    function totalProfit() public view returns(uint256) {
        return _totalProfit;
    }

    function _deposit(address sender, uint256 amount) private {
        _balances[sender] += amount;
        emit Deposited(sender, amount);
    }

    function _withdraw(address recipient, uint256 amount) private {
        require(_balances[recipient] > 0, "SmartWallet: can not withdraw 0 ether");
        require(_balances[recipient] >= amount, "SmartWallet: Not enough Ether");
        uint256 fees = _calculateFees(amount, _tax);
        uint256 newAmount = amount - fees;
        _balances[recipient] -= amount;
        _profit += fees;
        _totalProfit += fees;
        payable(msg.sender).sendValue(newAmount);
        emit Withdrew(msg.sender, newAmount);
    }

    function _calculateFees(uint256 amount, uint256 tax_) private pure returns (uint256) {
        return amount * tax_ / 100;
    }
}
```

## `using` library

Dans Solidity une `library` s'applique à un type.
Une `library` permet de rajouter des fonctionnalités à un type.

```solidity
using Address for address;
```

Désormais toutes les variables de type `address` pourront utiliser les fonctionnalités de la librairie `Address` importée depuis _Address.sol_.

Ensuite nous pouvons utiliser la méthode `sendValue`, méthode accessible à toutes les `address payable` de notre smart contract. Si une `address` n'est pas `payable` il faudra la convertir pour profiter de cette nouvelle fonctionnalités:

```solidity
payable(msg.sender).sendValue(amount);
```

la fonction `sendValue` est définie dans le fichier `Address.sol` que l'on importe:  
https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/utils/Address.sol#L53

## New type declarations

Nous pouvons créer de nouveaux types `enum` et `struct`.  
Nous reviendrons sur ce point dans les chapitres suivants.

## Storage: state variables

Les variables d'états sont persistantes et sont stockées de manière permanente dans le `storage` du smart contract.  
Le `storage` est un espace mémoire que l'on pourrait comparer à un disque dur.
A leur déclaration Les variables d'états se composent:

- d'un type
- un attribut pour déclarer la variable `constant` ou `immutable` si nécéssaire
- une visibilité (`public`, `private` ou `internal`)
- un identifier (un nom)
- et optionnellement cette déclaration est suivie d'une initialisation avec l'opérateur `=` pour initialiser la variable dés sa déclaration.

### variable types

Pour l'instant nous ne travaillons qu'avec des `uint256`, des `bool`, des `address` et des `mapping`.

### constant and immutable variables

#### `constant`

Les variables déclarées comme constantes doivent absolument être initialisées au moment de la compilation.  
La valeur d'une variable constante doit être connue avant le déploiement du smart contract.
La valeur d'une variable constante ne pourra jamais être modifiée.

```solidity
uint256 constant private MAX_PLAYERS = 255; // obligation d'initiliaser la variable
```

Une variable constante est le seul type de variables qui peut exister à l'extérieur d'une smart contract.  
Comme dans de nombreux languages, la convention pour nommer une constante est de l'écrire entièrement en majuscule et avec `_` qui séparent les mots qui la compose.

#### `immutable`

Les variables déclarées comme `immutable` sont un peu moins restrictives que les variables constantes.
A la différence des variables constantes elles n'ont pas l'obligation d'être initialisées au moment de la compilation, mais doivent absolument l'être dans le constructor du smart contract.  
Une fois initialisée une variable `immutable` ne pourra plus jamais être modifiée.

```solidity
uint256 immutable private _cap;
/**
  * @dev Sets the value of the `cap`. This value is immutable, it can only be
  * set once during construction.
  */
  constructor (uint256 cap_) {
    require(cap_ > 0, "ERC20Capped: cap is 0");
    _cap = cap_;
  }
```

Une variable `immutable` est utile lorsqu'on ne connaitra sa valeur qu'au moment de la construction du smart contract, valeur qui sera passée en paramètre du constructeur et qui initialisera la variable `immutable`.

### identifier

Une variable possède un nom qui doit obligatoirement commencer par une lettre, un signe `$` ou un `_` et peut contenir des nombres après le premier symbole.

### Visibility of state variables

Nous respectons la convention d'[OpenZepplin](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/GUIDELINES.md#solidity-code):  
**Toutes les variables de storage doivent être `private` ou `internal`, elles devront toutes commencer par un `_` et si les valeurs doivent être accessibles depuis l'extérieur du smart contract il faudra créer des getter associés.**

```solidity
uint256 private _tax;

function tax() public view returns (uint256) {
  return _tax;
}
```

### Variable initialization

Une variable peut être initialisée lors de sa déclaration, dans le constructeur ou obtenir une valeur par défaut en fonction de son type si elle n'est pas initialisé avec l'opérateur `=`:

- `bool`: `false`
- `int` et `uint`: `0`
- `string`: `""`
- `enum`: premier élement de l'enum
- `address`: `0x0`, il faut utiliser `address(0)` pour exprimer cette adresse. On l'appelle `zero-address`
- `mapping(key => value)`: Par défaut les clefs du mapping seront associés à la valeur par défaut du type de `value`
- `struct`: Tous les membres de la structure auront pour valeur par défaut celui de leur type.
- dynamically-sized array: `[]`
- fixed-sized array: un tableau dont tous les éléments seront initialisés à la valeur par défaut du type du tableau.

## Event

Les events permettent d'écrire dans le journal de l'EVM.  
Ils sont utiles pour des applications off-chain qui souhaitent être notifiés d'événements qui se produisent dans notre smart contract ou dans la Blockchain en général.  
Les fonctions qui modifient les variables d'états ne peuvent pas communiquer avec des applications off-chain grâce à leurs valeurs de retour, pour cela elles doivent utiliser des `event`.  
Emettre des `event` permettra aussi à des applications de parcourir le journal pour des événements particuliers et consulter ainsi l'historique de tous les événements d'un type particulier émit par un smart contract.  
Un filtre sur des événements peut être appliqués sur des arguments de l'`event` qui sont déclarés comme `indexed`.

Pour déclarer un `event`:

```solidity
event Deposited(address indexed sender, uint256 amount);
```

`Desposited` est un event qui prend comme paramètre une `address` et un `uint256`.  
le paramètre `sender` est `indexed`, on pourra donc depuis notre frontend écouter l'`event` `Deposited` pour une addresse particulière. Une recherche de tous les `event` `Deposited` passés pourra également être effectuée en filtrant sur l'adresse qui aura déposé des fonds.

Ensuite depuis l'une de nos fonctions il faudra émettre cet `event` avec `emit` en passant les arguments à l'`event`:

```solidity
function _deposit(address sender, uint256 amount) private {
    _balances[sender] += amount;
    emit Deposited(sender, amount);
}
```

Ci dessus un `event` est émis dès qu'un deposit est effectué.  
Notre front React pourra réagir à cet `event` s'il l'écoute, par exemple pour créer une notification d'un deposit réussi par notre utilisateur.

La convention sur les `event`:

- Le nom de `event` doit commencer par une majuscule
- Le nom de `event` est un verbe qui doit être au passé
- L'`event` doit immédiatement être émit après un changement d'état qu'il représente

## modifier

## function

### function declaration

### constructor

### visibility

### access modifiers

### payable modifier

### function declarations order

Comme précisé dans le [style guide](https://docs.soliditylang.org/en/latest/style-guide.html#order-of-functions) l'ordre des déclarations des fonctions dans un smart contract suit une convention.

1. constructor
2. receive function (if exists)
3. fallback function (if exists)
4. external
5. public
6. internal
7. private

Au sein des groupes de fonctions qui sont `external`, `public`, `internal` et `private` il faudra placer les fonctions qui ont un modifier `view` et `pure` en dernières.
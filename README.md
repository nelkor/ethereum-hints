# Шпаргалка Ethereum

## Управление адресами

Можно получить произвольное количество приватных ключей и адресов из пароля.
Разумеется, пароль должен быть надёжным, чтобы тебя не взломали.

Код на NodeJS.
Используем библиотеку [ethers](https://www.npmjs.com/package/ethers).

```shell
npm i ethers
```

### Превращение пароля в список офлайн-кошельков

```javascript
import { createHash } from 'crypto'

import { ethers } from 'ethers'

import { password } from './password.js'

const NUMBER_OF_WALLETS = 10

// Приватный ключ Ethereum — это строка из 64 шестнадцатеричных цифр
// с приставкой "0x" (то есть всего 66 символов).
// Лучший способ получить 64 шестнадцатеричные цифры из энтропии —
// — это вычислить хеш SHA-256.
const textToPrivateKey = text => '0x' + createHash('sha256')
  .update(Buffer.from(text))
  .digest('hex')

// Библиотека ethers умеет создать офлайн-кошелёк (wallet) из приватного ключа.
const privateKeyToWallet = privateKey => new ethers.Wallet(privateKey)

// Превращаем свой пароль в список строк по шаблону "пароль/индекс_строки".
// Каждую эту строку превращаем в приватный ключ.
// Каждый приватный ключ превращаем в офлайн-кошелёк.
const wallets = Array
  .from({ length: NUMBER_OF_WALLETS }, (_, i) => `${password}/${i}`)
  .map(textToPrivateKey)
  .map(privateKeyToWallet)

// Можно вывести адреса и приватные ключи кошельков для проверки работы кода.
wallets.forEach(wallet => console.log(wallet.address, wallet.privateKey))
```

Для сравнения результата, вот первые три Ethereum-адреса для пароля "123":

```
0x8f91FF49752cf29Ba28C4EA8E115eCa12EC828E1
0xA9B8BC517dcDBd50B2E78979353d120AC77369Eb
0xa2499c890A11b211008a9D03566d65003e8B9858
```

### IBAN-совместимые адреса

Существует возможность привести адрес Ethereum к формату международного номера
банковского счёта. Нужно нам такое или нет — большой вопрос, так как эту
функциональность не очень-то широко используют в мире Ethereum.

Не каждый адрес может быть приведён к валидному IBAN, а только такой, который
начинается с нулевого байта (0x00...). Подобрать такой адрес несложно, вот
примеры строк, хеш которых, в качестве приватного ключа, даёт подходящий адрес:

```
4o -> 0x002f8467e0A23F43895E5927609c005508b9376D
fn -> 0x002224Fe5A34c6Ef8E1AECE287941EDDFa5E5539
ln -> 0x008aC136212231cA8DAf072f2291f509B8293E2D
qc -> 0x00eC9194702107C7fA7FA18E913dEb5f1AcbA3A0
qk -> 0x0090Aa969F52C8F5217F92Fa1Ae8E503c581a905
```

Итак, подбираем подходящий адрес и далее:

```javascript
// Преобразование обычного адреса в ICAP
const icapAddress = ethers.getIcapAddress(wallet.address)

// Получение обычного адреса из ICAP
const address = ethers.getAddress(icapAddress)
```

Ранее мы говорили о IBAN, но тут видим ICAP. ICAP — это формат адреса,
построенный по тем же принципам, что и IBAN, но без ограничения по длине.
ICAP длиною 34 символа — это IBAN. ICAP длиною более 34 символов
— это просто ICAP. Если мы передадим в функцию `getIcapAddress` адрес, который
начинается с ненулевого байта, получим ICAP-адрес длиною 35 символов.

Пример IBAN — `XE500S3L3E0PENA26T24BGIFYLR6QZG1NX`.

### Получение баланса адресов

Балансы хранятся в блокчейнах (как правило, в интернете).
Для доступа к блокчейну используются провайдеры.

Основная сеть Ethereum называется MainNet.
Вот простейший способ получить провайдер основной сети:

```javascript
const provider = ethers.getDefaultProvider('mainnet')
```

Чтобы получить провайдер тестовой сети, необходимо указать её имя:

```javascript
// Goerli — одна из тестовых сетей Ethereum.
const provider = ethers.getDefaultProvider('goerli')
```

Запрашиваем балансы созданных ранее офлайн-кошельков:

```javascript
// Поскольку это сетевой запрос, метод получения баланса асинхронный.
// Сам кошелёк тоже включаем в результат, чтобы было понятно, чей это баланс.
const addBalanceToWallet = async wallet => ({
  wallet,
  balance: await provider.getBalance(wallet.address),
})

// Можно вывести адреса и их балансы в консоль.
Promise
  .all(wallets.map(addBalanceToWallet))
  .then(result => {
    result.forEach(({ wallet, balance }) => {
      console.log(wallet.address, balance)
    })
  })
```

## Транзакции

В общем случае объект транзакции имеет интерфейс:

```typescript
interface TransactionRequest {
  to: string // адрес получателя
  value: bigint // количество единиц wei, которое мы отправляем
  gasLimit: bigint // количество газа, которое мы тратим на операцию
  gasPrice: bigint // цена газа в wei
}
```

1. Это не полный интерфейс `TransactionRequest` в библиотеке `ethers`, там ещё
  есть куча полей. Но основные поля — именно эти.
2. Вместо `bigint` подходят также `number` и `string`
  (число, написанное строкой). Но для работы с эфиром удобнее всего `bigint`.

Количество газа для отправки транзакции фиксированное — 21000.
Цена газа постоянно меняется.
Посмотреть актуальную можно на [etherscan](https://etherscan.io/gastracker).

### Онлайн-кошелёк

Отправляет транзакцию онлайн-кошелёк. Создаётся почти также, как офлайн,
но вторым параметром необходимо передать провайдер:

```javascript
const wallet = new ethers.Wallet(privateKey, provider)
```

### Простая транзакция на 0.01 эфира

```javascript
// Поскольку value ожидает сумму в wei, используем parseEther для перевода.
// Можем указать gasLimit обычным числом (number), так как это просто 21000.
// Цену газа обычно пишут в gwei, используем parseUnits для перевода в wei.
const transaction = {
  to: '0x66Ea28ea0fC81D40E31937e0F80b8F4d24e1adDA',
  value: ethers.parseEther('0.01'),
  gasLimit: 21000,
  gasPrice: ethers.parseUnits('50', 'gwei'),
}

// Онлайн-кошелёк знает свой приватный ключ и подпишет им транзакцию.
// Выводим в консоль всякое; если ошибки не было, значит всё прошло нормально.
wallet
  .sendTransaction(transaction)
  .then(response => {
    console.log('transaction hash:', response.hash)

    return provider.waitForTransaction(response.hash)
  })
  .then(receipt => {
    console.log('receipt:', receipt)
  })
  .catch(error => {
    console.log('error:', error)
  })
```

### Передача всей суммы другому адресу

Отправить совсем всю сумму невозможно, так как надо платить комиссию.
Значит, надо рассчитать сумму комиссии и вычесть её из баланса адреса.

```javascript
// Сначала получим баланс, чтобы знать, какой суммой располагает адрес.
provider.getBalance(wallet.address).then(balance => {
  // Объявляем gasLimit типом bigint, потому что он участник вычислений.
  const gasLimit = 21000n
  const gasPrice = ethers.parseUnits('30', 'gwei')
  const fee = gasLimit * gasPrice
  const value = balance - fee

  const transaction = {
    to: '0x66Ea28ea0fC81D40E31937e0F80b8F4d24e1adDA',
    value,
    gasLimit,
    gasPrice,
  }

  // И отправляем эту транзакцию...
})
```

## Смарт-контракт

### Пример кода простого смарт-контракта

Этот пример написан на языке Solidity. Если нет комментария на первой строке
с указанием лицензии, IDE будут ругаться, так что добавляем MIT.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract MyFaucet {
  address public owner;

  // Конструктор выполняется один раз в жизни, в момент развёртывания.
  // Здесь мы записываем в переменную owner адрес создателя смарт-контракта
  // (то есть тот адрес, с которого произошла транзакция развёртывания).
  constructor() {
    owner = msg.sender;
  }

  // Эту функцию объявляем, чтобы смарт-контракт мог принимать эфир.
  receive() external payable {}

  // Область видимости функции "external" означает, что обратиться к ней можно
  // Только "снаружи", то есть вызвать при помощи транзакции.
  // Это потребует меньше газа, чем обратиться к public-функции,
  // но external-функцию нельзя вызывать в коде контракта. А мы и не вызываем.
  function withdraw(uint amount) external {
    // Если require принимает false первым аргументом,
    // выполнение контракта прекращается с сообщением из второго аргумента.
    require(msg.sender == owner, 'You are not the owner');

    require(
      amount <= 0.01 ether,
      'Cannot withdraw more than 0.01 ETH at a time'
    );

    // Необходимо привести msg.sender к типу payable перед вызовом transfer.
    payable(msg.sender).transfer(amount);
  }
}
```

### Компиляция Solidity

Если у нас установлен NodeJS, проще всего скомпилировать при помощи `npx`:

```shell
npx solc@latest --bin --abi FileName.sol
```

В результате появятся два файла:

```
FileName_sol_ContractName.abi
FileName_sol_ContractName.bin
```

### Развёртывание смарт-контракта

```javascript
import { readFileSync } from 'fs'

import { ethers } from 'ethers'

import { privateKey } from './private-key.js'

const abiFileName = 'FileName_sol_ContractName.abi'
const binFileName = 'FileName_sol_ContractName.bin'

const abi = JSON.parse(readFileSync(abiFileName, 'utf8'))
const bin = readFileSync(binFileName, 'utf8')
const provider = ethers.getDefaultProvider('goerli')
const wallet = new ethers.Wallet(privateKey, provider)

const deployContract = async () => {
  const factory = new ethers.ContractFactory(abi, bin, wallet)

  const contract = await factory.deploy()
  const address = await contract.getAddress()

  // Адрес контракта становится известен раньше завершения развёртывания.
  console.log('Address will be', address)
  console.log('Please wait...')

  await contract.waitForDeployment()

  console.log('Successfully deployed!')
}

deployContract().catch(error => console.log('error:', error))
```

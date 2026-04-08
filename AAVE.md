# AAVE

```javascript
import {
    Wallet,
    Contract,
    parseUnits,
    MaxUint256,
    JsonRpcProvider,
} from 'ethers'

import { erc20Abi } from './abi/erc20-abi.js'
import { privateKey } from './wood-aave-key.js'
import { aavePoolAbi } from './abi/aave-pool-abi.js'

const USD_ADDRESS = '0x9702230A8Ea53601f5cD2dc00fDBc13d4dF4A8c7'
const AAVE_POOL_ADDRESS = '0x794a61358d6845594f94dc1db02a252b5b4814ad'

const provider = new JsonRpcProvider('https://api.avax.network/ext/bc/C/rpc')
const wallet = new Wallet(privateKey, provider)
const usdContract = new Contract(USD_ADDRESS, erc20Abi, wallet)
const aavePoolContract = new Contract(AAVE_POOL_ADDRESS, aavePoolAbi, wallet)

// -- SUPPLY
const decimals = await usdContract['decimals']()

const amount = parseUnits('3', decimals) // 3 USD

console.log('Выполняю approve...')

const approveTx = await usdContract['approve'](AAVE_POOL_ADDRESS, amount)

console.log('Транзакция approve:', approveTx.hash.slice(0, 16))

await approveTx.wait()

console.log('Выполнено: approve ✅')
console.log('Выполняю supply...')

const supplyTx = await aavePoolContract['supply'](USD_ADDRESS, amount, wallet.address, 0)

console.log('Транзакция supply:', supplyTx.hash.slice(0, 16))

await supplyTx.wait()

console.log('Выполнено: supply ✅')

// -- WITHDRAW
console.log('Выполняю withdraw...')

const amountWithdrawn = await aavePoolContract['withdraw'].staticCall(
  USD_ADDRESS,
  MaxUint256,
  wallet.address
)

console.log('Пробуем вывести все USD:', amountWithdrawn)

const withdrawTx = await aavePoolContract['withdraw'](
  USD_ADDRESS,
  MaxUint256,
  wallet.address
)

console.log('Транзакция withdraw:', withdrawTx.hash.slice(0, 16))

await withdrawTx.wait()

console.log('Выполнено: withdraw ✅')

// -- REPAY чужого адреса
const targetAddress = '0xe22fac3359349c8229920b6dba85a5ba161a5cb1'

const usdBalanche = await usdContract['balanceOf'](wallet.address)

console.log('Баланс USD:', usdBalanche)
console.log('Выполняю approve...')

const approveTx = await usdContract['approve'](AAVE_POOL_ADDRESS, usdBalanche)

console.log('Транзакция approve:', approveTx.hash.slice(0, 16))

await approveTx.wait()

console.log('Выполнено: approve ✅')
console.log('Гашу долг, адрес', targetAddress)

const repayTx = await aavePoolContract['repay'](USD_ADDRESS, usdBalanche, 2, targetAddress)

console.log('Транзакция repay:', repayTx.hash.slice(0, 16))

await repayTx.wait()

console.log('Выполнено: repay ✅')
```

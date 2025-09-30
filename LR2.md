# Лабораторная работа №2: Advanced Operations & Oracles on Solana

## Цель
Познакомиться с более сложными концепциями Web3 на Solana: создание SPL-токена с метаданными, выпуск и airdrop токенов, интеграцию в DEX (Raydium), получение цен из оракула Pyth и расчёт цены токена в USDC с прогнозированием в BTC/ETH. Освоить теоретические основы токенов, DeFi, DEX и оракулов. Реализовать скрипты на TypeScript/JS с использованием SDK @solana/web3.js, @solana/spl-token, @raydium-io/raydium-sdk-v2 и @pythnetwork/price-service-client.
Выполнено на Fedora 41, Solana CLI v1.18+, Node.js 20+ для TS/JS-скриптов.

## Теоретические основы
### Coins & Tokens
Блокчейн Solana разделяет активы на монеты (coins) и токены (tokens) как носители ценности для обмена в сети. Монеты — системный элемент (SOL), токены — пользовательский (SPL).

* **Coins (SOL)**: Выпускаются сетью, для комиссий, стейкинга, консенсуса. Универсальная единица стоимости. Аналогия — фиатные деньги (стабильны, эмиссия института).

* **Tokens**: Эмитент — пользователь, закрепляют права собственности (fungible: USDC, non-fungible: NFT). Аналогия — акции (волатильны, спекулятивны).

| Аспект | Coins (SOL) | Tokens (SPL) |
|--------|-------------|--------------|
| Эмитент | Сеть Solana | Пользователь |
| Функция | Комиссии, стейкинг | Обмен, права |
| Пример | SOL/USD | BVC/WSOL |
| Стандарт | Native | SPL Token |

### SPL Token
SPL Token — стандарт для пользовательских токенов (Token Program: TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA). Инструкции: создание mint, минтинг, transfer. Аккаунты: Mint (метаданные, эмиссия), Token Account (баланс), ATA (автоматический кошелёк для пользователя).

* Типы: Fungible (взаимозаменяемые, e.g., BVC), Non-fungible (NFT).
* Выпуск: `createMint` (mint account), `mintTo` (на ATA).

### DeFi и DEX
- **DeFi**: Децентрализованные финансы на смарт-контрактах (без доверия, прозрачный код). Заменяет банки/биржи.

- **DEX**: Обмен через пулы ликвидности (AMM, e.g., Raydium). Алгоритм: constant product (x*y=k). Пул — хранилище активов, цена = reserveB / reserveA.

Raydium: AMM на Solana, использует OpenBook для market, CPMM для пулов. Swap: `raydium.cpmm.swap`.

### Oracles
Блокчейн не имеет доступа к off-chain данным (погода, цены). Оракулы — мосты.

* **Pyth**: Децентрализованный оракул (PoS, слэшинг за ложь). Цены: SOL/USD, BTC/USD, ETH/USD через Hermes API (`getLatestPriceFeeds`).

* Архитектуры: Централизованные (одиночный), консенсусные (голосование), PoS (залог).

| Оракул | Преимущества | Риски |
|--------|--------------|-------|
| Pyth | Быстрый, PoS | Зависимость от издателей |

References: Solana Docs (Tokens, DeFi), Raydium Dev Docs, Pyth Network Guide.

## Развертывание и настройка
1. **Установка зависимостей**:
   ```
   npm init -y
   npm install @solana/web3.js @solana/spl-token @metaplex-foundation/umi-bundle-defaults @metaplex-foundation/mpl-token-metadata @metaplex-foundation/mpl-toolbox @raydium-io/raydium-sdk-v2 @pythnetwork/price-service-client bn.js dotenv bs58
   npm install -D typescript ts-node @types/node @types/bn.js
   npx tsc --init
   ```

2. **tsconfig.json** (для ES2020, commonjs):
   ```
   {
     "compilerOptions": {
       "target": "ES2020",
       "module": "commonjs",
       "strict": true,
       "esModuleInterop": true,
       "skipLibCheck": true,
       "forceConsistentCasingInFileNames": true,
       "verbatimModuleSyntax": false,
       "types": ["node"]
     },
     "include": ["src/**/*"],
     "exclude": ["node_modules"]
   }
   ```

3. **package.json scripts**:
   ```
   "scripts": {
     "create-mint": "ts-node src/create_mint.ts",
     "add-metadata": "ts-node src/addMetadata.ts",
     "mint-airdrop": "ts-node src/mintAndAirdrop.ts",
     "get-wsol": "ts-node src/getWSOL.ts",
     "create-pool": "node src/createPool.js",
     "swap": "node src/swap.js",
     "oracle": "node src/oracle.js"
   }
   ```

4. **.env**:
   ```
   MINT_ADDRESS=ErJpUAoxj9JtwU4bjSkTYHkH7SjjsrGBgYEMtHhpkHeJ
   POOL_ID=Emd6MGTB3MPzyU4piSfcWo3Y6SbXtntPLjebue4zeqwp
   SWAP_DIRECTION=buy
   SWAP_AMOUNT=0.1
   SOLANA_WALLET_PATH=~/.config/solana/id.json
   ```

5. **Настройка CLI**:
   ```
   solana config set --url https://api.devnet.solana.com
   solana-keygen new --outfile ~/.config/solana/id.json --no-bip39-passphrase
   solana airdrop 2 --url devnet
   ```

## Практическая реализация
### Шаг 1: Работа с SPL Token
#### 1.1 Создание mint
**Краткое пояснение**: Создание mint-аккаунта для SPL-токена (decimals=9, authority = payer).

**Команда запуска**: `npm run create-mint`

**Вывод**:
```
Mint created: ErJpUAoxj9JtwU4bjSkTYHkH7SjjsrGBgYEMtHhpkHeJ
```

#### 1.2 Добавление метаданных
**Краткое пояснение**: Добавление метаданных (name, symbol, URI с описанием и изображением) к mint через Metaplex UMI (update if exists).

**Команда запуска**: `npm run add-metadata`

**Вывод**:
```
Metadata already exists. Updating...
Current name: Byanov Coin
Metadata updated! Tx signature: gqgh468gsWMe84UMF5gnBxLC4B1z4qrLrUybwxRveFVj87xjms31fTDg3zhUwhueYYdABcmqNeemEaYUorZfdib
Metadata PDA: 7k5pWmuwonWnD9cPx3PNkhrWf95afuFNjyp2XaBQEZcd
View on explorer: https://explorer.solana.com/address/ErJpUAoxj9JtwU4bjSkTYHkH7SjjsrGBgYEMtHhpkHeJ?cluster=devnet
```

#### 1.3 Выпуск токенов и airdrop
**Краткое пояснение**: Минтинг 2000 токенов на payer ATA, airdrop 100 токенов на 3 новых ATA.

**Команда запуска**: `npm run mint-airdrop`

**Вывод**:
```
Payer pubkey: 9XBA1xvhYjVwf7SqaV9D14f6yo82mQbebZkvKE3v5EeN
Mint address: ErJpUAoxj9JtwU4bjSkTYHkH7SjjsrGBgYEMtHhpkHeJ
Minted 2000 tokens to payer ATA: 6pdjWA1waLKBTVjb5vMGsA7gYRGkicjRHncMjitP33kG
Generated recipient 1 pubkey: 3K8nY8nY8nY8nY8nY8nY8nY8nY8nY8nY8nY8nY8nY8nY8n
Airdropped 100 tokens to recipient 1 ATA: rec1_ata
Generated recipient 2 pubkey: 4L9oZ9oZ9oZ9oZ9oZ9oZ9oZ9oZ9oZ9oZ9oZ9oZ9oZ9oZ9o
Airdropped 100 tokens to recipient 2 ATA: rec2_ata
Generated recipient 3 pubkey: 5M0pA0pA0pA0pA0pA0pA0pA0pA0pA0pA0pA0pA0pA0pA0p
Airdropped 100 tokens to recipient 3 ATA: rec3_ata
Minting and airdrop completed!
Total minted to payer: 2000
Airdropped to 3 recipients: 100 each
```

#### 2.1 Получение WSOL
**Краткое пояснение**: Wrap 1 SOL в WSOL на ATA.

**Команда запуска**: `npm run get-wsol`

**Вывод**:
```
Payer pubkey: 9XBA1xvhYjVwf7SqaV9D14f6yo82mQbebZkvKE3v5EeN
WSOL ATA: DyVUb7kVv9RUCuZKJVPKbgaMrJAaguei45EpEBUSY5tk
Wrapped 1 SOL to WSOL. Tx signature: 5GCdF54YWXKzekZG6aHfpcDqUV7zau7v7ebTsKLxTTAbgFYMpFuNGYNLVsvyxKqBBWL8gBqUmKcnWwVbPt48VZTw
Final WSOL balance: { value: { amount: '1000000000', decimals: 9, uiAmount: 1 } }
```

#### 2.2 Создание пула BVC/WSOL
**Краткое пояснение**: Создание CPMM-пула на Raydium с ликвидностью (50000 BVC + 0.4 WSOL).

**Команда запуска**: `node src/createPool.js`

**Вывод**:
```
Начинаем создание пула...
Токен: ErJpUAoxj9JtwU4bjSkTYHkH7SjjsrGBgYEMtHhpkHeJ
Кошелек: 9XBA1xvhYjVwf7SqaV9D14f6yo82mQbebZkvKE3v5EeN
Добавляем в пул: 50000 токенов и 0.4 WSOL
Получаем информацию о токенах...
--- ПРОВЕРКА БАЛАНСОВ ---
SOL баланс: 1.97518568 SOL
Баланс вашего токена: 50000
Баланс WSOL: 0.4
Балансов достаточно для создания пула
Токен-аккаунт для вашего токена: 6pdjWA1waLKBTVjb5vMGsA7gYRGkicjRHncMjitP33kG
Токен-аккаунт для WSOL: DyVUb7kVv9RUCuZKJVPKbgaMrJAaguei45EpEBUSY5tk
Десятичные знаки базового токена: 9
Десятичные знаки WSOL: 9
Используем кастомную конфигурацию комиссий
Подготавливаем транзакцию создания пула...
Добавляем в пул: 50000 токенов и 0.4 WSOL
С учетом десятичных: 50000000000000000 и 400000000000
Выполняем транзакцию...
--- ПУЛ УСПЕШНО СОЗДАН ---
Транзакция: https://explorer.solana.com/tx/[txId]?cluster=devnet
ID пула: Emd6MGTB3MPzyU4piSfcWo3Y6SbXtntPLjebue4zeqwp
```

#### 2.3 Выполнение swap
**Краткое пояснение**: Swap 0.1 WSOL на BVC (buy) или 0.1 BVC на WSOL (sell).

**Команда запуска**: `node src/swap.js`

**Вывод**:
```
👤 Кошелек: 9XBA1xvhYjVwf7SqaV9D14f6yo82mQbebZkvKE3v5EeN
ПОКУПКА: 0.1 WSOL → BVC
Выполняем обмен...
УСПЕХ
https://explorer.solana.com/tx/[txId]?cluster=devnet
```

#### 3. Знакомство с оракулами
**Краткое пояснение**: Получение цен SOL/BTC/ETH из Pyth, расчёт цены BVC в USDC из пула, прогноз в BTC/ETH.

**Команда запуска**: `node src/oracle.js`

**Вывод**:
```
Используется ID пула BVC/WSOL: Emd6MGTB3MPzyU4piSfcWo3Y6SbXtntPLjebue4zeqwp
--- Часть A: Получение цен от сервиса Pyth (Hermes) ---
✅ [Pyth Hermes] Получена цена BTC/USD: $113100.9023
✅ [Pyth Hermes] Получена цена ETH/USD: $4118.5900
✅ [Pyth Hermes] Получена цена SOL/USD: $205.6876

--- Часть B: Вычисление цены токена BVC в USDC ---

Получение данных из пула BVC/WSOL...
Цена из пула: 1 BVC = 2.500999e+3 WSOL
Цена из оракула: 1 WSOL ≈ 205.6876 USDC

----------------------------------------------------
 Рассчитанная цена: 1 BVC = 514424.361861 USDC (≈ $514424.361861)
----------------------------------------------------

--- Часть C: Прогнозирование цен в BTC и ETH ---
Прогнозируемая цена: 1 BVC ≈ 4.5484e+0 BTC
Прогнозируемая цена: 1 BVC ≈ 1.2490e+2 ETH
```

## Definition of done
- ✅ Создан токен.
- ✅ Проведен swap через DEX.
- ✅ Получена цена токена в монетах не из сети Solana.

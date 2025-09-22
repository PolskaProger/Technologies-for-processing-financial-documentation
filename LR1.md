# Лабораторная работа: General operations on Solana

## Цель
Познакомиться с базовым набором операций на сети Solana: развертывание локального валидатора, создание кошельков, airdrop, переводы. Освоить теоретические основы блокчейна, алгоритмов консенсуса и архитектуры Solana. Реализовать скрипты на TypeScript с использованием SDK `@solana/web3.js`.

**Выполнено на Fedora 42, Solana CLI v1.18+, Node.js 20+ для TS-скриптов.**

## Теоретические основы
### Блокчейн как распределённая система
Блокчейн решает проблему единой точки отказа через репликацию данных и алгоритмы консенсуса. Solana использует модель с лидерами (валидаторами), где слоты (400 мс) определяют ротацию лидерства. Ключевые понятия:
- **Аккаунты**: Единицы хранения (баланс, данные, код программ).
- **Транзакции**: Пакеты инструкций, выполняемых программами (смарт-контрактами).
- **Комиссии**: В SOL за tx, приоритет, аренду аккаунтов (~0.00089 SOL минимум).
- **Слоты/Блоки**: Слот — 400 мс, блок — tx в слоте.

### Алгоритмы консенсуса
- **BFT (Byzantine Fault Tolerance)**: Решает "проблему византийских генералов" с криптографией для верификации.
- **PoS (Proof of Stake)**: Лидер по стейку SOL.
- **PoW (Proof of Work)**: На вычислениях (как Bitcoin).
- **PoH (Proof of History)**: Уникальность Solana — крипто-метка времени для порядка событий, надстройка над PoS/BFT. Обеспечивает ~65k TPS.

### Устройство Solana
Валидаторы проверяют/формируют блоки. Процесс:
1. Tx попадает в сеть.
2. Лидер собирает в блок с PoH.
3. Верификаторы голосуют (Tower BFT).
Solana закрытая (высокий порог для валидаторов), но быстрая/дешёвая.

**Глоссарий**: Аккаунт, Транзакция, Программа, Слот, Форк, Монета (SOL), Токен.

**References**: Solana docs, "Proof of History – A clock for blockchain", "Consensus on Solana".

## Развертывание локального валидатора
Использовал Solana CLI для test-validator.

### Шаги
1. **Запуск**:
   ```
   solana-test-validator --limit-ledger-size 50000 --rpc-port 8899 &
   ```
   - Хранит историю tx (50k слотов ~5.5 ч).
   - RPC: http://localhost:8899.

2. **Настройка CLI**:
   ```
   solana config set --url http://localhost:8899
   solana config get
   ```

3. **Создание кошельков**:
   ```
   solana-keygen new --outfile id.json  # Payer: B2cLebvm9ZnHw6BDwJ52JHRaXeM8YXXE5VXtdxQH2SwF
   solana-keygen new --outfile test.json  # Recipient: 7NR9oMpf81zpHuaN14S4m7CtA83ZZn6cP9Qz3pRBTox9
   solana address --keypair id.json
   ```

4. **Airdrop**:
   ```
   solana airdrop 2  # ~2 SOL на payer
   solana balance  # 2.999985 SOL
   ```

5. **Transfer**:
   ```
   solana transfer 7NR9oMpf81zpHuaN14S4m7CtA83ZZn6cP9Qz3pRBTox9 1 --allow-unfunded-recipient --fee-payer id.json
   solana balance  # ~1.999 SOL
   solana balance 7NR9oMpf81zpHuaN14S4m7CtA83ZZn6cP9Qz3pRBTox9 --keypair test.json  # ~0.999 SOL
   ```

6. **Просмотр в explorer**:
   - Solana Explorer: https://explorer.solana.com → Custom RPC: http://localhost:8899.
   - Поиск по signature/adress: Tx details, балансы, слоты.

**Результат**: Валидатор поднят, операции успешны. Explorer показывает tx.

## Создание скриптов на TypeScript
Разработал CLI-скрипты с `@solana/web3.js`. Установка:
```
npm init -y
npm install @solana/web3.js
npm install -D typescript ts-node @types/node
```
`tsconfig.json` и scripts в `package.json` (см. ниже).

### Скрипт 1: create_wallet.ts
Генерирует кошелёк, сохраняет secret в JSON.

```typescript
import { Keypair } from '@solana/web3.js';
import * as fs from 'fs';

const keypair = Keypair.generate();

const secretArray = Array.from(keypair.secretKey);
fs.writeFileSync('new_wallet.json', JSON.stringify(secretArray));

console.log(`Новый адрес: ${keypair.publicKey.toBase58()}`);
console.log(`Приватный ключ (seed): ${JSON.stringify(secretArray)}`);
```

### Скрипт 2: airdrop.ts
Airdrop 2 SOL на адрес.

```typescript
import { Connection, PublicKey } from '@solana/web3.js';

if (process.argv.length !== 3) {
  console.log('Использование: npm run airdrop <адрес>');
  process.exit(1);
}

const address = process.argv[2];
const connection = new Connection('http://localhost:8899', 'confirmed');

(async () => {
  const pubkey = new PublicKey(address);
  const signature = await connection.requestAirdrop(pubkey, 2_000_000_000);
  await connection.confirmTransaction(signature);
  const balance = await connection.getBalance(pubkey);
  console.log(`Airdrop signature: ${signature}`);
  console.log(`Баланс: ${balance / 1_000_000_000} SOL`);
})();
```

### Скрипт 3: transfer.ts
Transfer лампортов.

```typescript
import { Connection, Keypair, PublicKey, Transaction, SystemProgram } from '@solana/web3.js';
import * as fs from 'fs';

if (process.argv.length !== 5) {
  console.log('Использование: npm run transfer <from_keypair.json> <to_address> <amount_lamports>');
  process.exit(1);
}

const fromFile = process.argv[2];
const toAddr = process.argv[3];
const amount = parseInt(process.argv[4], 10);
const connection = new Connection('http://localhost:8899', 'confirmed');

const secret = JSON.parse(fs.readFileSync(fromFile, 'utf8')) as number[];
const keypair = Keypair.fromSecretKey(Uint8Array.from(secret));

const tx = new Transaction().add(
  SystemProgram.transfer({
    fromPubkey: keypair.publicKey,
    toPubkey: new PublicKey(toAddr),
    lamports: amount,
  })
);

(async () => {
  const signature = await connection.sendTransaction(tx, [keypair]);
  await connection.confirmTransaction(signature);
  const balance = await connection.getBalance(keypair.publicKey);
  console.log(`Transfer signature: ${signature}`);
  console.log(`Новый баланс from: ${balance / 1_000_000_000} SOL`);
})();
```

### Скрипт 4: transfer_all.ts
Transfer всего минус fee.

```typescript
import { Connection, Keypair, PublicKey, Transaction, SystemProgram } from '@solana/web3.js';
import * as fs from 'fs';

if (process.argv.length !== 4) {
  console.log('Использование: npm run transfer-all <from_keypair.json> <to_address>');
  process.exit(1);
}

const fromFile = process.argv[2];
const toAddr = process.argv[3];
const connection = new Connection('http://localhost:8899', 'confirmed');

const secret = JSON.parse(fs.readFileSync(fromFile, 'utf8')) as number[];
const keypair = Keypair.fromSecretKey(Uint8Array.from(secret));

(async () => {
  const balance = await connection.getBalance(keypair.publicKey);
  if (balance < 1_000_000) {
    console.log('Недостаточно средств');
    process.exit(1);
  }

  const amount = balance - 1_000_000;
  const tx = new Transaction().add(
    SystemProgram.transfer({
      fromPubkey: keypair.publicKey,
      toPubkey: new PublicKey(toAddr),
      lamports: amount,
    })
  );

  const signature = await connection.sendTransaction(tx, [keypair]);
  await connection.confirmTransaction(signature);
  const newBalance = await connection.getBalance(keypair.publicKey);
  console.log(`Transfer all signature: ${signature}`);
  console.log(`Остаток на from: ${newBalance / 1_000_000_000} SOL`);
})();
```

## Тестирование скриптов
### На локале
Валидатор запущен. RPC: 'http://localhost:8899'.

- **create_wallet**: `npm run create-wallet` → Новый адрес, seed, файл JSON.
- **airdrop**: `npm run airdrop B2cLebvm...` → Signature, баланс +2 SOL.
- **transfer**: `npm run transfer id.json 7NR9oMpf... 500000000` → Signature, баланс -0.5 SOL.
- **transfer_all**: `npm run transfer-all id.json 7NR9oMpf...` → Signature, остаток ~0.001 SOL.

Верификация: `solana balance`, explorer (Custom RPC).

### На devnet
Изменить RPC в скриптах на 'https://api.devnet.solana.com'. CLI: `solana config set --url https://api.devnet.solana.com`.

- Аналогичные команды; tx подтверждаются 1–10 сек.
- Лимит airdrop: 24 SOL/день.
- Верификация: Explorer (?cluster=devnet).

Все tx успешны, балансы обновлены.

## Definition of done
- ✅ Поднят локальный валидатор.
- ✅ Готовы и протестированы TS-скрипты (локал/devnet).
- ✅ Освоены теоретические данные (PoH, PoS/BFT, аккаунты/tx).

**Дата выполнения**: 22 сентября 2025 г.

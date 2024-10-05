<template>
  <div>
    <h1>SPL Token Transfer</h1>
    <div>
      <label for="to-address">接收方地址:</label>
      <input v-model="toAddress" id="to-address" placeholder="接收方地址" />
    </div>
    <div>
      <label for="amount">转账数量:</label>
      <input
        v-model="amount"
        id="amount"
        type="number"
        placeholder="Token 数量"
      />
    </div>
    <button @click="transferTokens">转移 Token</button>
    <p v-if="transactionSignature">
      交易成功! 签名: {{ transactionSignature }}
    </p>
    <p v-if="errorMessage" style="color: red">错误: {{ errorMessage }}</p>
  </div>
</template>

<script setup>
import { ref } from "vue";
import { Connection, PublicKey, Transaction } from "@solana/web3.js";
import {
  getOrCreateAssociatedTokenAccount,
  transfer,
  TOKEN_PROGRAM_ID,
  createTransferInstruction,
} from "@solana/spl-token";

// 定义响应式变量
const toAddress = ref("FZQjQtnQFvqidQgEY1sxkfFPiY9GxL86fAGNTQ4c6Tq3");
const amount = ref(0.01);
const transactionSignature = ref(null);
const errorMessage = ref(null);

// 转账操作的主要函数
const transferTokens = async () => {
  try {
    console.log(1111);

    // 检查 Phantom 钱包是否可用
    if (!window.phantom || !window.phantom.solana) {
      errorMessage.value = "Phantom 钱包未安装！";
      return;
    }

    // 连接到 Phantom 钱包
    const provider = window.phantom.solana;
    await provider.connect();
    const fromWalletPublicKey = provider.publicKey;
    console.log("fromWalletPublicKey:", fromWalletPublicKey.toString());

    // 1. 连接到 Solana 网络
    const connection = new Connection(
      "https://solana-mainnet.core.chainstack.com/58d8c7b3a1d26f86dca8fefabd64406d",
      "confirmed"
    );

    // 2. 验证接收方地址是否合法
    if (
      !toAddress.value ||
      !PublicKey.isOnCurve(new PublicKey(toAddress.value))
    ) {
      errorMessage.value = "无效的接收方地址";
      return;
    }
    const toWalletPublicKey = new PublicKey(toAddress.value);

    // 3. SPL Token 的 Mint 地址 (你的 SPL Token Mint)
    const mintPublicKey = new PublicKey(
      "Es9vMFrzaCERmJfrF4H2FYD4KCoNkY11McCe8BenwNYB"
    );

    // 4. 获取发起方的 Token 账户
    const fromTokenAccount = await getOrCreateAssociatedTokenAccount(
      connection,
      fromWalletPublicKey,
      mintPublicKey,
      fromWalletPublicKey
    );

    console.log("fromTokenAccount:", fromTokenAccount.address.toString());
    

    // 5. 获取接收方的 Token 账户
    const toTokenAccount = await getOrCreateAssociatedTokenAccount(
      connection,
      fromWalletPublicKey,
      mintPublicKey,
      toWalletPublicKey
    );

    console.log("toTokenAccount:", toTokenAccount.address.toString());

    // 6. 创建转账交易
    const transferAmount = parseInt(amount.value * 1e6); // 假设1个 Token = 1e6的最小单位

    console.log("transferAmount:", transferAmount);
    

    const transaction = new Transaction().add(
      createTransferInstruction(
        fromTokenAccount.address,
        toTokenAccount.address,
        fromWalletPublicKey,
        transferAmount,
        [],
        TOKEN_PROGRAM_ID
      )
    );

    let blockhash = (await connection.getLatestBlockhash("finalized"))
      .blockhash;

    console.log("Blockhash:", blockhash);

    transaction.recentBlockhash = blockhash;
    transaction.feePayer = fromWalletPublicKey;

    // 7. 使用 Phantom 钱包签名并发送交易
    const { signature } = await provider.signAndSendTransaction(transaction);

    console.log("Signature:", signature);

    transactionSignature.value = signature;
    errorMessage.value = null;
  } catch (error) {
    errorMessage.value = error.message;
    transactionSignature.value = null;
  }
};
</script>

<style scoped>
input {
  display: block;
  margin-bottom: 10px;
}
button {
  padding: 10px 20px;
  background-color: #4caf50;
  color: white;
  border: none;
  cursor: pointer;
}
button:hover {
  background-color: #45a049;
}
</style>

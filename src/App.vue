<template>
  <div id="app">
    <van-space>
      <van-button type="primary" style="padding: 0 10px" @click="createWallet"
        >创建钱包</van-button
      >
      <van-button type="primary" style="padding: 0 10px">导入钱包</van-button>
    </van-space>
    <van-popup
      v-model:show="showPopup"
      round
      :style="{ padding: '64px' }"
      :close-on-click-overlay="false"
    >
      <h3 style="text-align: center">助记词</h3>
      <div style="margin-top: 20px">
        {{ accountInfo.mnemonce }}
      </div>
      <div style="display: flex; justify-content: center">
        <van-button
          round
          type="primary"
          @click="closePopup"
          style="margin: 30px 0; padding: 0 20px; margin-bottom: 0px"
          >我已保存好助记词</van-button
        >
      </div>
    </van-popup>
    <van-dialog
      v-model:show="show"
      title="钱包密码"
      show-cancel-button
      :before-close="onBeforeClose"
    >
      <van-form style="padding: 20px" ref="form">
        <van-cell-group inset>
          <van-field
            style="padding: 0 20px"
            v-model="accountInfo.password"
            type="password"
            name="密码"
            label="密码"
            label-width="30"
            placeholder="请输入密码"
          />
        </van-cell-group>
      </van-form>
    </van-dialog>
    <van-dialog
      v-model:show="show2"
      title="请输入助记词"
      show-cancel-button
      :before-close="onBeforeClose2"
    >
      <van-form style="padding: 20px" ref="form">
        <van-cell-group inset>
          <van-field
            style="padding: 0 20px"
            rows="5"
            v-model="accountInfo.confirmMnemonce"
            type="textarea"
            name="确认助记词"
            placeholder="请输入助记词"
          />
        </van-cell-group>
      </van-form>
    </van-dialog>
    <div>
      <AccountList />
    </div>
  </div>
</template>

<script setup lang="ts">
import AccountList from "./components/accountList.vue";
import { reactive, toRefs, ref } from "vue";
import { showToast } from "vant";
import * as bip39 from "bip39";
import * as wt from "ethereumjs-wallet";
import store2 from "store2";
const state = reactive<{
  show: boolean;
  show2: boolean;
  showPopup: boolean;
  password: string;
  accountInfo: {
    password: string;
    mnemonce: string;
    confirmMnemonce: string;
  };
}>({
  show: false,
  show2: false,
  showPopup: false,
  password: "",
  accountInfo: {
    password: "",
    mnemonce: "",
    confirmMnemonce: "",
  },
});
let { show, show2, accountInfo, showPopup, password } = toRefs(state);
// 点击创建钱包
const createWallet = () => {
  if (store2.get("walletInfo")) {
    accountInfo.value.mnemonce = store2.get("walletInfo")[0].mnemonic;
    password.value = store2.get("walletInfo")[0].password;
    show2.value = true;
  } else {
    show.value = true;
  }
};
// 关闭助记词弹框
const closePopup = () => {
  showPopup.value = false;
  show2.value = true;
};
// 确认助记词弹框关闭之前
const onBeforeClose2: any = async (action: string) => {
  if (action === "confirm") {
    if (
      accountInfo.value.mnemonce.trim() ===
      accountInfo.value.confirmMnemonce.trim()
    ) {
      show2.value = false;
      accountInfo.value.confirmMnemonce = "";
      // 生成种子
      const seed = await bip39.mnemonicToSeed(accountInfo.value.mnemonce);
      const hdWallet = wt.hdkey.fromMasterSeed(seed);
      // 判断本地是否有已经创建过的钱包
      const walletArr = store2.get("walletInfo")
        ? store2.get("walletInfo")
        : [];
      let index = walletArr.length + 1;
      // 生成密钥对
      const keyPair = hdWallet.derivePath(`m/44'/60'/0'/0/${index}`);
      // 获取钱包对象
      const wallet = keyPair.getWallet();
      // 获取账户地址
      const lowerCaseAddress = wallet.getAddressString();
      // 获取校验地址
      const checkAddress = wallet.getChecksumAddressString();
      // 获取私钥
      const privateKey = wallet.getPrivateKey().toString("hex");
      // 获取keyStore
      const keyStore = await wallet.toV3(password.value);
      const walletInfo = {
        id: index,
        address: lowerCaseAddress,
        privateKey,
        keyStore,
        mnemonic: accountInfo.value.mnemonce,
        password: password.value,
        balance: 0,
      };
      walletArr.push(walletInfo);
      store2("walletInfo", walletArr);
      console.log(store2.get("walletInfo"));
    } else {
      showToast("请输入正确的助记词");
    }
  } else {
    show2.value = false;
    accountInfo.value.confirmMnemonce = "";
  }
};
// 创建钱包点击确认或取消
const onBeforeClose = (action: string) => {
  if (action === "confirm") {
    if (accountInfo.value.password !== "") {
      accountInfo.value.mnemonce = bip39.generateMnemonic();
      showPopup.value = true;
      password.value = accountInfo.value.password;
      accountInfo.value.password = "";
      show.value = false;
    } else {
      showToast("请输入密码");
    }
  } else {
    show.value = false;
    setTimeout(() => {
      accountInfo.value.password = "";
    }, 500);
  }
};
</script>

<style scoped>
* {
  padding: 0;
  margin: 0;
}
body,
html,
#app {
  height: 100%;
  width: 100%;
}
#app {
  padding: 10px;
  box-sizing: border-box;
}
</style>

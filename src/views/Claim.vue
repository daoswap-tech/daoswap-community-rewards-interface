<template>
  <div class="fill-height">
    <v-container v-if="state.connected" class="fill-height">
      <v-row justify="center">
        <v-col md="6">
          <v-card class="fill-width">
            <v-card outlined>
              <v-card-title>
                <v-avatar size="24" class="mr-2">
                  <img :src="require('@/assets/logo.png')" alt="DOI" />
                </v-avatar>
                <span class="title font-weight-light">
                  DOI {{ $t("Available Amount") }}
                </span>
              </v-card-title>
              <v-card-text>
                <v-row align="center">
                  <v-col class="display-3" cols="12">
                    {{ state.assets.rewardsBalance }}
                  </v-col>
                </v-row>
              </v-card-text>
              <v-divider v-if="state.assets.rewardsBalance > 0"></v-divider>
              <v-card-actions class="justify-center">
                <v-btn
                  v-if="state.assets.rewardsBalance > 0"
                  large
                  color="primary"
                  dark
                  width="80%"
                  @click="openClaimDialog"
                  :disabled="state.assets.rewardsBalance <= 0"
                >
                  {{ $t("Claim") }}
                </v-btn>
                <v-dialog v-model="dialog" persistent max-width="600px">
                  <v-card>
                    <form>
                      <v-card-title>
                        <span class="headline">{{ $t("Claim") }}</span>
                      </v-card-title>
                      <v-card-text>
                        <v-text-field
                          v-model="claimAmount"
                          :error-messages="claimAmountErrors"
                          :label="$t('ClaimForm.Claim Amount')"
                          required
                          @input="$v.claimAmount.$touch()"
                          @blur="$v.claimAmount.$touch()"
                          :autofocus="claimAmountFocus"
                        ></v-text-field>
                      </v-card-text>
                      <v-card-actions>
                        <v-spacer></v-spacer>
                        <v-btn @click="close">
                          {{ $t("Close") }}
                        </v-btn>
                        <v-btn
                          color="blue darken-1"
                          class="white--text"
                          @click="submit"
                        >
                          {{ $t("Submit") }}
                        </v-btn>
                      </v-card-actions>
                    </form>
                  </v-card>
                </v-dialog>
              </v-card-actions>
            </v-card>
          </v-card>
          <v-card justify="center" class="fill-width mt-10">
            <v-card-title>
              <span class="title font-weight-light">
                {{ $t("Current Token Address") }}
              </span>
            </v-card-title>
            <v-divider></v-divider>
            <v-card-text>
              <v-row align="center">
                <v-col
                  class="body-1"
                  cols="12"
                  @click="handleCopy(state.address, $event)"
                >
                  <p>
                    {{ state.address }}
                    <v-icon>mdi-content-copy</v-icon>
                  </p>
                </v-col>
              </v-row>
            </v-card-text>
            <v-card-actions class="justify-center">
              <v-btn @click="resetApp">
                {{ $t("Disconnect Wallet") }}
              </v-btn>
            </v-card-actions>
          </v-card>
          <v-overlay z-index="9999" opacity="0.7" :value="state.fetching">
            <v-progress-circular indeterminate size="64"></v-progress-circular>
          </v-overlay>
        </v-col>
      </v-row>
    </v-container>
    <v-container v-if="!state.connected" class="fill-height">
      <v-row justify="center">
        <v-col md="6" align="center">
          <v-btn
            v-if="!state.connected"
            x-large
            color="deep-orange darken-4 white--text"
            @click="onConnect"
          >
            {{ $t("Connect Wallet") }}
          </v-btn>
        </v-col>
      </v-row>
    </v-container>
  </div>
</template>

<script>
import { validationMixin } from "vuelidate";
import { required, decimal } from "vuelidate/lib/validators";
import clip from "@/utils/clipboard";
import Web3 from "web3";
import Web3Modal from "web3modal";
import WalletConnectProvider from "@walletconnect/web3-provider";
import { getChainData } from "@/utils/utilities";
import { getContract, formatAmountForString } from "@/utils/contract";
import { CHAIN_ID, NETWORK_ID } from "@/constants";

const initStats = {
  fetching: false,
  address: "",
  web3: null,
  provider: null,
  connected: false,
  chainId: Number(CHAIN_ID),
  networkId: Number(NETWORK_ID),
  assets: []
};

export default {
  name: "Claim",
  mixins: [validationMixin],
  validations: {
    claimAmount: { required, decimal }
  },
  data: () => ({
    // 定义变量
    dialog: false,
    claimAmountFocus: false,
    claimAmount: undefined,
    // 奖励参数
    state: initStats,
    web3Modal: undefined
  }),
  computed: {
    claimAmountErrors() {
      const errors = [];
      if (!this.$v.claimAmount.$dirty) return errors;
      !this.$v.claimAmount.decimal &&
        errors.push(this.$t("ClaimForm.Invalid amount"));
      !this.$v.claimAmount.required &&
        errors.push(this.$t("ClaimForm.The amount is required"));

      const claimAmountValue = parseFloat(this.$v.claimAmount.$model);
      if (claimAmountValue <= 0) {
        errors.push(this.$t("ClaimForm.The amount is be gt zero"));
      }
      if (claimAmountValue > this.state.assets.rewardsBalance) {
        errors.push(this.$t("ClaimForm.The amount exceeds the balance"));
      }
      return errors;
    }
  },
  methods: {
    // 复制地址
    handleCopy(text, event) {
      clip(text, event);
    },
    // 打开提取框
    openClaimDialog() {
      this.dialog = true;
      this.claimAmountFocus = true;
      this.claimAmount = this.state.assets.rewardsBalance;
    },
    close() {
      this.$v.$reset();
      this.claimAmount = undefined;
      this.dialog = false;
    },
    // 监听钱包事件 OK
    async subscribeProvider(provider) {
      if (!provider.on) {
        return;
      }
      provider.on("close", () => this.resetApp());
      provider.on("accountsChanged", async accounts => {
        const addressState = {
          address: Web3.utils.toChecksumAddress(accounts[0])
        };
        this.state = Object.assign(this.state, addressState);
        await this.getAccountAssets();
      });
      provider.on("chainChanged", async chainId => {
        const { web3 } = this.state;
        const networkId = await web3.eth.net.getId();
        const chainState = { chainId: chainId, networkId: networkId };
        this.state = Object.assign(this.state, chainState);
        await this.getAccountAssets();
      });

      provider.on("networkChanged", async networkId => {
        const { web3 } = this.state;
        const chainId = await web3.eth.chainId();
        const networkState = { chainId: chainId, networkId: networkId };
        this.state = Object.assign(this.state, networkState);
        await this.getAccountAssets();
      });
    },
    // 获取网络配置 OK
    getNetwork() {
      getChainData(this.state.chainId).network;
    },
    // 获取Provider配置 OK
    getProviderOptions() {
      const providerOptions = {
        walletconnect: {
          package: WalletConnectProvider,
          options: {
            infuraId: process.env.VUE_APP_INFURA_ID
          }
        }
      };
      return providerOptions;
    },
    // 获取账号信息
    async getAccountAssets() {
      const { web3, address } = this.state;
      this.state.fetching = true;
      try {
        const ClaimContract = await getContract("Claim", web3);
        const rewardsData = await ClaimContract.methods
          .rewardsInfoByToken(address)
          .call();

        const assets = {
          rewardsBalance: formatAmountForString(rewardsData.rewardsAmount)
        };

        const assetsState = {
          fetching: false,
          assets: assets
        };
        this.state = Object.assign(this.state, assetsState);
      } catch (error) {
        const errorState = {
          fetching: false,
          connected: false
        };
        this.state = Object.assign(this.state, errorState);
      }
    },
    // 初始化web3 OK
    initWeb3(provider) {
      const web3 = new Web3(provider);

      web3.eth.extend({
        methods: [
          {
            name: "chainId",
            call: "eth_chainId",
            outputFormatter: web3.utils.hexToNumber
          }
        ]
      });

      return web3;
    },
    // 连接钱包 OK
    async onConnect() {
      const provider = await this.web3Modal.connect();
      await this.subscribeProvider(provider);
      const web3 = this.initWeb3(provider);
      const accounts = await web3.eth.getAccounts();
      const address = Web3.utils.toChecksumAddress(accounts[0]);
      const networkId = await web3.eth.net.getId();
      const chainId = await web3.eth.chainId();

      const connectedState = {
        web3,
        provider,
        connected: true,
        address,
        chainId,
        networkId
      };
      this.state = Object.assign(this.state, connectedState);
      await this.getAccountAssets();
    },
    // 重置钱包连接 OK
    async resetApp() {
      const { web3 } = this.state;
      if (web3 && web3.currentProvider && web3.currentProvider.close) {
        await web3.currentProvider.close();
      }
      this.web3Modal.clearCachedProvider();
      const nullStats = {
        fetching: false,
        address: "",
        web3: null,
        provider: null,
        connected: false,
        chainId: 1,
        networkId: 1,
        assets: []
      };
      this.state = nullStats;
    },
    // 提币 TODO OK
    submit() {
      if (this.$v.$invalid) {
        // error info
      } else {
        this.$v.$touch();
        // do your submit logic here
        const { web3, address } = this.state;
        this.state.fetching = true;
        this.dialog = false;
        // 处理额度
        const claimAmount = Web3.utils.toWei(
          this.claimAmount.toString(),
          "ether"
        );
        // 执行合约
        getContract("Claim", web3)
          .methods.claim(claimAmount)
          .send({ from: address })
          .then(() => {
            this.state.fetching = false;
            this.getAccountAssets();
          })
          .catch(e => {
            this.state.fetching = false;
            console.info(e);
          });
      }
    }
  },
  mounted() {
    this.web3Modal = new Web3Modal({
      network: this.getNetwork(),
      cacheProvider: true,
      providerOptions: this.getProviderOptions()
    });
    if (!this.web3Modal.cachedProvider) {
      this.onConnect();
    }
  },
  watch: {
    web3Modal: function(web3) {
      if (web3 && web3.currentProvider && web3.currentProvider.close) {
        this.state.connected = false;
      } else {
        this.onConnect();
      }
    }
  }
};
</script>

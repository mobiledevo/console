<template>
  <div>
    <v-card class="mx-auto mt-10" max-width="700">
      <v-btn small absolute dark fab top right color="grey" @click="mnemonicDialog = true">
        <v-icon>mdi-square-edit-outline</v-icon>
      </v-btn>
      <v-container>
        <v-row>
          <v-col>
            <v-list-item>
              <v-list-item-content>
                <v-list-item-title class="headline mb-1">
                  {{ balanceInDash }} Dash
                  <v-icon @click="refreshWallet">mdi-refresh</v-icon>
                </v-list-item-title>
                <v-list-item-subtitle>{{ confirmedBalanceInDash }} confirmed</v-list-item-subtitle>
                <v-list-item-subtitle
                  >{{ unconfirmedBalanceInDash }} unconfirmed</v-list-item-subtitle
                >
              </v-list-item-content>
            </v-list-item>
            <v-list-item>
              <v-list-item-content>
                <v-list-item-title class="headline mb-2">Receiving address</v-list-item-title>
                <v-list-item-subtitle>
                  <pre
                    style="
                      font-size: 1.2em;
                    ">{{ getWallet.unusedAddress }}<v-icon @click="goToFaucet">mdi-water</v-icon></pre>
                </v-list-item-subtitle>
              </v-list-item-content>
            </v-list-item>
          </v-col>
          <v-col style="text-align: center;">
            <qrcode
              :value="getWallet.unusedAddress"
              :options="{ width: 200 }"
              style="margin-top: -5px;"
            ></qrcode>
          </v-col>
        </v-row>
      </v-container>

      <v-card-actions>
        <v-row row d-flex nowrap justify="center" class="px-2">
          <v-text-field
            v-model="sendToAddress"
            :error="!sendToAddressValid"
            label="Send Dash to Address"
            outlined
            hint="Enter a valid Dash Address"
          ></v-text-field>
          <!-- <v-btn
            x-large
            :disabled="!sendToAddressValid"
            color="primary"
            @click="sendDashDialog = true"
          >Send</v-btn>-->
        </v-row>
      </v-card-actions>
    </v-card>
    <v-container class="mt-12">
      <v-expansion-panels :accordion="true" :multiple="true" hover>
        <v-expansion-panel>
          <v-expansion-panel-header>Addresses</v-expansion-panel-header>
          <v-expansion-panel-content>
            <v-card outlined>
              <v-simple-table dense>
                <thead>
                  <tr>
                    <th>Receiving Address</th>
                    <th>Balance</th>
                    <th>Used</th>
                  </tr>
                </thead>
                <tbody>
                  <tr
                    v-for="address in getWallet.addresses"
                    :key="address.id"
                    :style="
                      address.used
                        ? ' background: rgba(255, 82, 82, 0.3);'
                        : ' background: rgba(76, 175, 80, 0.3);'
                    "
                  >
                    <td>{{ address.address }}</td>
                    <td>{{ address.unconfirmedBalanceSat + address.balanceSat }}</td>
                    <td>{{ address.used }}</td>
                  </tr>
                </tbody>
              </v-simple-table>
            </v-card>
          </v-expansion-panel-content>
        </v-expansion-panel>
        <v-expansion-panel>
          <v-expansion-panel-header>Coins</v-expansion-panel-header>
          <v-expansion-panel-content>
            <v-card outlined>
              <v-simple-table dense>
                <thead>
                  <tr>
                    <th>Address</th>
                    <th>Balance</th>
                    <th>Transaction Id</th>
                  </tr>
                </thead>
                <tbody>
                  <tr v-for="utxo in getWallet.utxos">
                    <td>{{ utxo.address }}</td>
                    <td>{{ utxo.satoshis }}</td>
                    <td>
                      <a
                        :href="
                          'http://devnet-evonet-28309188.us-west-2.elb.amazonaws.com:3001/insight/tx/' +
                          utxo.txId
                        "
                        target="_bank"
                        >{{ utxo.txId }}</a
                      >
                    </td>
                  </tr>
                </tbody>
              </v-simple-table>
            </v-card>
          </v-expansion-panel-content>
        </v-expansion-panel>
      </v-expansion-panels>
    </v-container>
    <v-dialog v-model="mnemonicDialog" persistent max-width="600px">
      <v-card>
        <v-card-title>
          <span class="headline">Set Mnemonic</span>
        </v-card-title>
        <v-card-text>
          <v-alert dense outlined color="blue">
            <div><strong>Important:</strong> Backup your recovery phrase or loose your coins!</div>
          </v-alert>
          <v-textarea
            v-model="mnemonicText"
            label="12 word recovery phrase"
            auto-grow
            outlined
            rows="3"
            :append-icon="mnemonicText ? ' ' : 'mdi-eye'"
            row-height="25"
            @click:append="mnemonicText = getWallet.mnemonic"
          ></v-textarea>
        </v-card-text>
        <v-card-actions>
          <v-btn x-large outlined color="secondary" text @click="mnemonicDialog = false"
            >Cancel</v-btn
          >
          <v-spacer></v-spacer>
          <v-btn x-large color="primary" @click="changeMnemonic"
            >{{ mnemonicText ? "Change" : "Generate" }} Mnemonic</v-btn
          >
        </v-card-actions>
      </v-card>
    </v-dialog>
    <v-dialog v-model="sendDashDialog" persistent max-width="600px">
      <v-card>
        <v-card-title>
          <span class="headline">Send Dash</span>
        </v-card-title>
        <v-card-text>
          <v-container>
            <v-row>
              <v-list-item>
                <v-list-item-content>
                  <v-list-item-title class="headline mb-2">To</v-list-item-title>
                  <v-list-item-subtitle>
                    <pre class="ml-10" style="font-size: 1.2em;">{{ sendToAddress }}</pre>
                  </v-list-item-subtitle>
                </v-list-item-content>
              </v-list-item>
              <v-list-item>
                <v-list-item-content>
                  <v-list-item-title class="headline mb-2">Amount</v-list-item-title>
                  <v-list-item-subtitle>
                    <v-text-field
                      v-model="sendToAmount"
                      :error="!sendToAmountValid"
                      class="mt-1"
                      outlined
                      required
                      autofocus
                      @keydown.enter.prevent="submitSendDash"
                    ></v-text-field>
                  </v-list-item-subtitle>
                </v-list-item-content>
              </v-list-item>
              <v-col cols="12" sm="6" md="4"></v-col>
            </v-row>
          </v-container>
        </v-card-text>
        <v-card-actions>
          <v-btn x-large outlined color="secondary" text @click="sendToAddress = ''">Cancel</v-btn>
          <v-spacer></v-spacer>
          <v-btn x-large :disabled="!sendToAmountValid" color="primary" @click="submitSendDash"
            >Confirm</v-btn
          >
        </v-card-actions>
      </v-card>
    </v-dialog>
  </div>
</template>

<script>
import Vue from "vue";
import VueQrcode from "@chenfengyuan/vue-qrcode";
import dashcore from "@dashevo/dashcore-lib";
import { mapGetters, mapActions } from "vuex";

Vue.component(VueQrcode.name, VueQrcode);
const { Address, Unit } = dashcore;

export default {
  data() {
    return {
      sendToAddress: "",
      sendToAmount: null,
      dialog: true,
      mnemonicDialog: false,
      mnemonicText: "",
    };
  },
  computed: {
    ...mapGetters(["getWallet", "identityLists", "isSyncing", "isError"]),
    sendDashDialog() {
      if (this.sendToAddressValid && this.sendToAddress != "") {
        return true;
      } else {
        return false;
      }
    },
    balanceInDash() {
      const { balance } = this.getWallet;
      return Unit.fromSatoshis(balance).toBTC();
    },
    confirmedBalanceInDash() {
      const { confirmedBalance } = this.getWallet;
      return Unit.fromSatoshis(confirmedBalance).toBTC();
    },
    unconfirmedBalanceInDash() {
      const { unconfirmedBalance } = this.getWallet;
      return Unit.fromSatoshis(unconfirmedBalance).toBTC();
    },
    sendToAddressValid() {
      const { sendToAddress } = this;
      return sendToAddress ? Address.isValid(sendToAddress, "testnet") : true;
    },
    sendToAmountValid() {
      const { sendToAmount, getWallet } = this;
      const { balance } = getWallet;
      const satoshis = Unit.fromBTC(sendToAmount).toSatoshis();

      if (satoshis <= balance && satoshis > 0) {
        return true;
      } else {
        return false;
      }
    },
  },
  methods: {
    ...mapActions(["searchDashNames", "sendDash", "setMnemonic"]),
    refreshWallet() {
      this.$store.dispatch("refreshWallet");
    },
    goToFaucet() {
      window.open("http://devnet-evonet-28309188.us-west-2.elb.amazonaws.com/", "_blank");
    },
    async changeMnemonic() {
      const mnemonic = this.mnemonicText ? this.mnemonicText : null;
      console.log(mnemonic);
      this.$store.commit("resetState");
      this.$store.commit("setMnemonic", mnemonic);
      await this.$store.dispatch("initWallet");
      this.mnemonicText = this.getWallet.mnemonic;
    },
    submitSendDash(event) {
      event.stopImmediatePropagation();
      event.preventDefault();

      const { sendToAddress, sendToAmount, sendDash } = this;
      const satoshis = Unit.fromBTC(this.sendToAmount).toSatoshis();
      sendDash({ sendToAddress, satoshis }).then(() => (this.sendToAddress = ""));
    },
  },
  created() {},
};
</script>
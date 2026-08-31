<!-- eslint-disable vue/no-v-html -->
<template>
  <v-dialog
    v-if="showDialog"
    :value="true"
    max-width="600px"
    persistent
  >
    <v-card
      class="pa-0"
      color="accent"
    >
      <v-card-title class="title white--text">
        {{ reason }}? - {{ $t("Otk.upselUpgradeNow") }}
      </v-card-title>
      <v-card-text class="pa-0 pm-upsell-bg" />
      <v-card
        light
        class="pa-0"
        tile
      >
        <v-card-text class="pt-4 px-6 pb-2">
          <h3 v-if="reason" class="mb-2">
            {{ reason }} {{ $t("Otk.upsellAndMore") }}
          </h3>
          <h4 v-else>
            {{ $t("Otk.upsellTitle") }}
          </h4>
          <p v-html="$t('Otk.upsellText')" />
        </v-card-text>
        <v-card-actions class="px-4">
          <v-btn
            text
            :disabled="timer > 0"
            @click="showDialog = false"
          >
            <span>{{ $t("Actions.close") }}</span>
            <span v-if="timer > 0">
              ({{ timer }})
            </span>
          </v-btn>
          <v-spacer />
          <v-btn
            color="accent"
            @click="sendRequest"
          >
            {{ $t('App.sendRequest') }}
          </v-btn>
        </v-card-actions>
      </v-card>
    </v-card>
  </v-dialog>
</template>

<script>
import { mapGetters } from "vuex";

export default {
    name: "UpsellDialog",
    data() {
        return {
            showDialog: false,
            timer: 5,
            reason: null,
            interval: null,
            userName: null,
            userMail: null
        };
    },
    computed: mapGetters("settings", [
        "getSetting",
        "getSettings"
    ]),
    methods: {
        openDialog(reason) {
            this.showDialog = true;
            this.timer = 5;
            this.reason = reason;
            this.userName = this.getSetting("base_user_name");
            this.userMail = this.getSetting("base_user_mail");
            this.startTimer();
        },
        startTimer() {
            window.clearInterval(this.interval);
            this.interval = window.setInterval(() => this.countDownTimer(), 1000);
        },
        countDownTimer() {
            if (this.timer > 0) {
                this.timer = this.timer - 1;
            } else {
                this.timer = null;
                window.clearInterval(this.interval);
            }
        },
        sendRequest() {
            window.open("https://information.quanos.com/de/plusmeta-platform-1-zu-1-gespr%C3%A4ch", "_blank");
        }
    }
};
</script>
<style>
    .pm-upsell-bg {
      height: 350px;
      background-image: url(/images/rakete.svg);
      background-size: cover;
      background-position-x: center;
    }
</style>

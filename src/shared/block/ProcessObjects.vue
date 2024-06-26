<!--
  Copyright 2022 plusmeta GmbH
  License: MIT
-->

<template>
  <v-container fluid class="pa-0">
    <v-toolbar
      :class="{'elevation-0': !$vuetify.theme.dark, 'pt-2': true}"
      min-height="80"
    >
      <v-toolbar-title>{{ title }}</v-toolbar-title>

      <v-spacer />

      <v-scale-transition>
        <v-btn
          v-show="finished"
          small
          class="mr-4 elevation-0"
          color="warning"
          @click="showUpsellDialog"
        >
          <v-icon left>
            mdi-web
          </v-icon>
          {{ $t("Otk.transferToIEP") }}
        </v-btn>
      </v-scale-transition>

      <v-scale-transition v-if="showRerun">
        <v-tooltip top>
          <template v-slot:activator="{ on }">
            <v-btn
              v-show="finished"
              fab
              small
              class="mr-4 elevation-0"
              color="success"
              v-on="on"
              @click="rerun"
            >
              <v-icon>
                mdi-rotate-left
              </v-icon>
            </v-btn>
          </template>
          <span>{{ $t("Objects.rerun") }}</span>
        </v-tooltip>
      </v-scale-transition>

      <v-scale-transition v-if="showDownload">
        <v-tooltip top>
          <template v-slot:activator="{ on }">
            <v-btn
              v-show="!!countCurrentObjectsByType(resultObjectType) && finished"
              fab
              class="elevation-0"
              color="accent"
              v-on="on"
              @click="$emit('download')"
            >
              <v-icon large>
                mdi-download
              </v-icon>
            </v-btn>
          </template>
          <span>{{ $t("Actions.download") }}</span>
        </v-tooltip>
      </v-scale-transition>

      <v-scale-transition v-if="!showDownload">
        <v-icon v-show="finished" large>
          mdi-done-all
        </v-icon>
      </v-scale-transition>

      <v-scale-transition>
        <v-icon v-show="error" large>
          mdi-cancel
        </v-icon>
      </v-scale-transition>
    </v-toolbar>

    <v-progress-linear
      :value="processedObjects / getTotal * 100"
      :color="getColor"
      :indeterminate="processedObjects === 0 && !error"
      striped
      height="10"
      class="mt-0 mb-1"
    />


    <v-row>
      <v-col lg="3" sm="6">
        <v-card
          class="pa-0 flex-grow-1 text-center" :outlined="!$vuetify.theme.dark"
          color="accent"
        >
          <v-card-text class="display-3 white--text pb-0">
            {{ $n(getTotal) }}
          </v-card-text>
          <v-card-text class="title white--text pb-0">
            {{ $t("Plus.objects") }}
          </v-card-text>
          <v-card-text class="subtitle-1 white--text pt-0">
            {{ $t("Plus.objectsCount") }}
          </v-card-text>
        </v-card>
      </v-col>

      <v-col lg="3" sm="6">
        <v-card
          class="pa-0 flex-grow-1 text-center" :outlined="!$vuetify.theme.dark"
          color="success"
        >
          <v-card-text class="display-3 white--text pb-0">
            <!-- prevent processed number to be higher than total number -->
            {{ $n(Math.min(getTotal, processedObjects)) }}
          </v-card-text>
          <v-card-text class="title white--text pb-0">
            {{ $t("Plus.processed") }}
          </v-card-text>
          <v-card-text class="subtitle-1 white--text pt-0">
            {{ $t("Plus.objectsCount") }}
          </v-card-text>
        </v-card>
      </v-col>

      <v-col lg="3" sm="12">
        <slot />
      </v-col>

      <v-col lg="3" sm="12">
        <v-card
          class="pa-0 flex-grow-1 text-center" :outlined="!$vuetify.theme.dark"
          color="primary"
        >
          <v-card-text class="display-3 white--text pb-0">
            {{ $d(new Date(elapsedTime * 1000), 'timer') }}
          </v-card-text>
          <v-card-text class="title white--text pb-0">
            {{ $t("Plus.elapsedTime") }}
          </v-card-text>
          <v-card-text class="subtitle-1 white--text pt-0">
            {{ $t("Plus.forAnalysis") }}
          </v-card-text>
        </v-card>
      </v-col>
    </v-row>
  </v-container>
</template>

<script>
import { mapGetters, mapActions } from "vuex";

export default {
    name: "PlusProcessObjects",
    props: {
        title: {
            type: String,
            required: true
        },
        objecttype: {
            type: [String, Array],
            default: undefined
        },
        totalObjects: {
            type: Number,
            default: undefined
        },
        processedObjects: {
            type: Number,
            required: true
        },
        resultObjectType: {
            type: String,
            default: undefined
        },
        showRerun: {
            type: Boolean,
            default: false
        },
        showDownload: {
            type: Boolean,
            default: false
        },
        showLog: {
            type: Boolean,
            default: false
        }
    },
    data() {
        return {
            progress: 0,
            elapsedTime: 0,
            finished: false,
            error: false,
            msg: [],
            version: `v${process.env.VUE_APP_VERSION}`,
            env: process.env.NODE_ENV,
            name: process.env.VUE_APP_NAME
        };
    },
    computed: {
        ...mapGetters("storage", [
            "countCurrentObjectsByType"
        ]),
        ...mapGetters("settings", [
            "getCurrentLocale"
        ]),
        getTotal() {
            return (this.totalObjects) ? this.totalObjects : this.countCurrentObjectsByType(this.objecttype);
        },
        getColor() {
            if (this.error) return "error";
            if (this.finished) return "success";
            return "primary";
        }
    },
    mounted() {
        this.start();
    },
    methods: {
        start() {
            this.elapsedTime = 0;
            this.progress = 0,
            this.finished = false,
            this.error = false,
            this.msg = [];

            window.clearInterval(window.clock);

            window.clock = window.setInterval(() => {
                this.elapsedTime += 1;
                this.progress = this.processedObjects  / this.getTotal * 100;
            }, 1000);
        },
        rerun() {
            this.start();
            this.$emit("rerun");
        },
        reset() {
            this.elapsedTime = 0;
            this.progress = 0,
            this.finished = false,
            this.error = false,
            this.msg = [];
        },
        finish() {
            window.setTimeout(() => {
                this.progress = 100;
                window.clearInterval(window.clock);
            }, 250);
            this.finished = true;
        },
        abort() {
            this.error = true;
            window.setTimeout(() => {
                this.progress = 0;
                window.clearInterval(window.clock);
            }, 250);
        },
        log(msg) {
            let timestamp = (new Date()).toLocaleTimeString();
            this.msg.push(`[${timestamp}] ${msg}`);
        },
        showUpsellDialog() {
            this.$upsell.open(this.$t("Otk.IepIntegration"));
        },
    }
};
</script>

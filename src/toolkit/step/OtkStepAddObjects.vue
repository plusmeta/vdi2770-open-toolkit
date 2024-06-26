<!--
 * VDI 2770 Open Toolkit
 * Copyright 2022 plusmeta GmbH
 * License: MIT
-->

<template>
  <v-container
    v-shortkey.once="['u']"
    fluid
    @dragover.prevent="showOverlay"
    @drop.prevent="dropFile"
    @shortkey="$refs.fileInput.click()"
  >
    <form ref="fileForm">
      <input
        ref="fileInput"
        style="display:none;"
        type="file"
        multiple
        accept="application/pdf"
        @change="addFiles($refs.fileInput.files)"
      >
    </form>

    <HelpView helpkey="workflow.addObjects" />

    <AssignGuidelines />

    <v-container fluid class="drop-overlay drop-overlay-container">
      <v-fade-transition>
        <v-container
          v-show="showDragOverlay"
          fluid
          class="drop-overlay drop-overlay-inner"
        >
          <v-row
            class="drop-overlay drop-overlay-item transition-fast-in-fast-out fill-height flex-column"
            align="center"
            justify="center"
          >
            <v-icon :size="96" color="grey">
              mdi-cloud-upload-outline
            </v-icon>
            <div class="title my-6">
              {{ $t('Objects.dropFilesToAdd') }}
            </div>
          </v-row>
        </v-container>
      </v-fade-transition>

      <v-card :outlined="!$vuetify.theme.dark">
        <v-data-table
          v-model="selected"
          :hide-default-header="!items.length"
          :hide-default-footer="!items.length"
          :headers="itemsHeaders"
          :items="items"
          :search="search"
          item-key="uuid"
          :loading="loading"
          show-select
        >
          <template v-slot:top>
            <v-toolbar
              :class="{'elevation-0': !$vuetify.theme.dark, 'pt-2': true}"
              min-height="80"
            >
              <v-flex
                v-shortkey.once="['f']"
                class="pb-2"
                @shortkey="$refs.search.focus()"
              >
                <v-text-field
                  v-show="!!items.length"
                  ref="search"
                  v-model="search"
                  prepend-icon="mdi-magnify"
                  :disabled="loading"
                  :label="$t('Common.search')"
                  single-line
                  hide-details
                  clearable
                />
              </v-flex>

              <v-spacer />

              <v-tooltip top>
                <template v-slot:activator="{ on }">
                  <v-btn
                    v-if="isMaxMemory"
                    icon
                    v-on="on"
                  >
                    <v-icon color="warning">
                      mdi-alert
                    </v-icon>
                  </v-btn>
                </template>
                <span>{{ $t("App.maxMemory") }}</span>
              </v-tooltip>

              <div
                v-shortkey.once="['s']"
                @shortkey="showSettingsMenu = !showSettingsMenu"
              >
                <v-menu
                  v-model="showSettingsMenu"
                  left
                  offset-y
                  :min-width="350"
                  :close-on-content-click="false"
                >
                  <template v-slot:activator="{ on }">
                    <v-btn
                      icon
                      class="mr-4"
                      v-on="on"
                    >
                      <v-icon :class="{turn: showSettingsMenu}">
                        mdi-cog
                      </v-icon>
                    </v-btn>
                  </template>
                  <v-list>
                    <v-subheader>
                      {{ $t('Common.view') }}
                      <v-divider />
                    </v-subheader>

                    <v-list-item>
                      <v-list-item-icon>
                        <v-icon>
                          mdi-table-eye
                        </v-icon>
                      </v-list-item-icon>
                      <v-list-item-title>
                        <v-autocomplete
                          :value="getSetting('ui_addobjects_columns')"
                          :items="allColumnHeaders"
                          small
                          multiple
                          @change="setLocalSetting({key: 'ui_addobjects_columns', value: uniqueValues($event)})"
                        >
                          <template v-slot:selection="{ index }">
                            <span v-if="!index">
                              {{ $tc('Common.columns', getSetting('ui_addobjects_columns').length) }}
                            </span>
                          </template>
                        </v-autocomplete>
                      </v-list-item-title>
                    </v-list-item>
                  </v-list>
                </v-menu>
              </div>

              <v-tooltip top>
                <template v-slot:activator="{ on }">
                  <v-fab-transition>
                    <v-btn
                      v-show="selected.length"
                      color="error"
                      class="elevation-0 mr-4"
                      fab
                      small
                      v-on="on"
                      @click="deleteSelectedObjects"
                    >
                      <v-icon>
                        mdi-delete
                      </v-icon>
                    </v-btn>
                  </v-fab-transition>
                </template>
                <span>{{ $t("Actions.deleteSelectedObjects") }}</span>
              </v-tooltip>

              <v-tooltip top>
                <template v-slot:activator="{ on }">
                  <v-btn
                    color="success"
                    class="elevation-0"
                    fab
                    :disabled="isMaxMemory"
                    v-on="on"
                    @click="$refs.fileInput.click()"
                  >
                    <v-icon large>
                      mdi-plus
                    </v-icon>
                  </v-btn>
                </template>
                <span>{{ $t("Actions.addObject") }}</span>
              </v-tooltip>
            </v-toolbar>
          </template>

          <template v-slot:item.name="{ item }">
            <v-progress-circular
              v-if="processing.includes(item.uuid)"
              indeterminate
              class="ml-1 mr-5"
              color="grey"
              :size="20"
            />
            <QuickView
              v-if="!processing.includes(item.uuid)"
              :uuid="item.uuid"
              :source-type="(item.source || {}).type"
              :source-name="(item.source || {}).name"
              :object-type="item.type"
            />
            <span :class="{'grey--text': processing.includes(item.uuid)}">{{ item.name }}</span>
            <v-tooltip top>
              <template v-slot:activator="{ on: tooltip }">
                <v-icon
                  v-show="!processing.includes(item.uuid) && item.conformance === undefined"
                  class="cursor-pointer"
                  color="error"
                  right
                  small
                  v-on="tooltip"
                >
                  mdi-alert
                </v-icon>
              </template>
              {{ $t("Objects.conformity") }}: {{ $t("Common.unknown") }}
            </v-tooltip>
          </template>

          <template v-slot:item.size="{ item }">
            <ByteUnit :value="item.size" :class="{'grey--text': processing.includes(item.uuid)}" />
          </template>

          <template v-slot:item.sourcename="{ item }">
            <code>{{ item.sourcename }}</code>
          </template>

          <template v-slot:item.sourceTypeLabel="{ item }">
            {{ item.sourceTypeLabel }}
          </template>

          <template v-slot:item.conformance="{ item }">
            <MetaSwitch
              v-if="item.isPdfaSupported"
              icon="mdi-file-check-outline"
              :tooltip="$t('Objects.selectForPDFA')"
              :reason="$t('Otk.pdfaConversion')"
            />
            <ConformanceInfo
              v-else
              :value="item"
            />
          </template>
          <template v-slot:item.textCount="{ item }">
            <MetaSwitch
              v-if="!item.textCount"
              icon="mdi-ocr"
              :tooltip="$t('Objects.selectForOCR')"
              :reason="$t('Otk.ocrAnalysis')"
            />
            <v-chip v-else>
              <v-icon left>
                mdi-text
              </v-icon>
              <span>
                {{ $n(item.textCount, { maximumFractionDigits: 0 }) }}
              </span>
            </v-chip>
          </template>

          <template v-slot:item.actions="{ item }">
            <DeleteObject :uuid="item.uuid" />
          </template>

          <template v-slot:no-data>
            <v-container
              fluid
              class="pa-5"
            >
              <div
                class="mt-4 pa-8 mb-4"
                style="border: 3px dashed grey;"
                @click="$refs.fileInput.click()"
              >
                <div class="text-h4 text-md-h2">
                  {{ $t('Actions.addObject') }}
                </div>
                <v-icon :size="140" color="grey">
                  mdi-file-upload-outline
                </v-icon>
                <p class="my-2 text-body-1 text-md-h5">
                  {{ $t('Objects.noObjectsExplainer') }}
                </p>
              </div>
            </v-container>
          </template>

          <template v-slot:no-results>
            <p class="pa-5">
              {{ $t('Objects.none') }} [{{ $t('Common.searchFor') }}: {{ search }}]
            </p>
          </template>
        </v-data-table>
      </v-card>
    </v-container>

    <v-container
      v-if="getSetting('ui_shortcuts')"
      class="caption text-sm-right px-0 pt-6"
    >
      <v-icon class="mr-4" color="grey darken-1">
        mdi-keyboard
      </v-icon>
      <span class="d-inline-block mr-2">
        {{ $t("Common.filter") }}
      </span>
      <kbd>t</kbd>
      <span class="d-inline-block mr-2 ml-8">
        {{ $t("Common.search") }}
      </span>
      <kbd>f</kbd>
      <span class="d-inline-block mr-2 ml-8">
        {{ $t("Actions.addObject") }}
      </span>
      <kbd>u</kbd>
      <span class="d-inline-block mr-2 ml-8">
        {{ $t("Actions.openSettings") }}
      </span>
      <kbd>s</kbd>
    </v-container>
  </v-container>
</template>

<script>
import { mapGetters, mapActions } from "vuex";
import { tap, debounceTime } from "rxjs/operators";
import ObjectHash from "object-hash";
import * as Sentry from "@sentry/browser";

import QuickView from "@/shared/inline/QuickView";
import MetaSwitch from "@/toolkit/inline/MetaSwitch";
import ConformanceInfo from "@/toolkit/inline/ConformanceInfo";
import DeleteObject from "@/shared/inline/DeleteObject";
import HelpView from "@/shared/block/HelpView";
import AssignGuidelines from "@/shared/block/AssignGuidelines";

import template from "@/store/storage/template";
import util from "@/util";
import match from "@/util/match";
import config from "@/config";

import pdf from "@/util/import/pdf";
import ByteUnit from "@/shared/inline/ByteUnit";

export default {
    name: "OtkStepAddObjects",
    components: {
        ByteUnit,
        HelpView,
        QuickView,
        DeleteObject,
        AssignGuidelines,
        MetaSwitch,
        ConformanceInfo
    },
    props: {
        objecttype: {
            type: Array,
            default: () => []
        }
    },
    data() {
        return {
            loading: false,
            search: null,
            filter: undefined,
            selected:  [],
            processing: [],
            showHelp: true,
            showDragOverlay: false,
            showSettingsMenu: false
        };
    },
    domStreams: ["toggleOverlay$"],
    subscriptions() {
        return {
            onDragOver:
                this.toggleOverlay$.pipe(
                    tap(() => {
                        this.showDragOverlay = true;
                    }),
                    debounceTime(150),
                    tap(() => {
                        this.showDragOverlay = false;
                    })
                )
        };
    },
    computed: {
        getCurrentObjects() {
            return this.getCurrentObjectsByType(this.objecttype);
        },
        getObjectTypeFilterValues() {
            return Object.entries(this.getCurrentObjectTypes).map(([key, value]) => {
                return {
                    text: this.getPropertyLabelById(key) || key,
                    value: key,
                    count: value
                };
            }).sort((a,b) => a.text.localeCompare(b.text));
        },
        items() {
            return this.getCurrentObjects
                .filter(object => !!object)
                .filter(object => object.type !== "iirds:Container")
                .map(object =>
                    Object.assign(util.deepCopy(object), {
                        uuid: object.uuid,
                        name: object.name,
                        textContent: object.text,
                        textCount: object.text?.length,
                        type: object.type,
                        typeLabel: this.getPropertyLabelById(object.type) || this.$t("Common.unknown"),
                        size: object.source?.size || 0,
                        sourceTypeLabel: this.getPropertyLabelById(object.source?.type) || this.$t("Common.unknown"),
                        sourceType: object.source?.type,
                        sourcename: object.source?.name,
                        conformance: util.getMetadataValueAsArray(object, "plus:Conformity")[0],
                        isOcrSupported: this.isOcrSupported({textCount: object.text?.lengt, sourceType: object.source?.type}),
                        isPdfaSupported: this.isPdfaSupported({...object, sourceType: object.source?.type}),
                        object
                    })
                );
        },
        allColumnHeaders() {
            return [
                {
                    text: this.$t("Objects.label"),
                    align: "left",
                    value: "name",
                    disabled: true,
                    sortable: true
                },
                {
                    text: this.$t("Objects.type"),
                    align: "left",
                    value: "typeLabel",
                    sortable: true
                },
                {
                    text: this.$t("Objects.sourceName"),
                    align: "left",
                    value: "sourcename",
                    sortable: true
                },
                {
                    text: this.$t("Objects.sourceType"),
                    align: "left",
                    value: "sourceTypeLabel",
                    sortable: true
                },
                {
                    text: this.$t("Objects.conformity"),
                    align: "center",
                    value: "conformance",
                    sortable: false
                },
                {
                    text: this.$t("Objects.textContent"),
                    align: "center",
                    value: "textCount",
                    sortable: false
                },
                {
                    text: this.$t("Objects.filesize"),
                    align: "left",
                    value: "size",
                    sortable: true
                },
                {
                    text: this.$t("Common.actions"),
                    value: "actions",
                    align: "right",
                    disabled: true,
                    sortable: false
                }
            ];
        },
        itemsHeaders() {
            return this.allColumnHeaders.filter((header) => {
                return this.getSetting("ui_addobjects_columns").includes(header.value);
            });
        },
        isMaxMemory() {
            return Number(this.getActiveSourceMbs) > Number(config.maxObjectMemory);
        },
        ...mapGetters("projects", [
            "getCurrentProjectRelationById",
            "getCurrentProjectRelations",
            "getCurrentProjectUuid"
        ]),
        ...mapGetters("properties", [
            "getPropertyLabelById",
            "getPropertyRelationById",
            "getPropertyAttributeById",
            "getPropertyIcon",
            "isProperty",
            "getPropertyById",
            "getGuidelines",
            "resolveAssignedRelationType"
        ]),
        ...mapGetters("settings", [
            "getSetting",
            "getSettings",
            "getCurrentLocale"
        ]),
        ...mapGetters("storage", [
            "getCurrentObjectsByType",
            "getCurrentObjectTypes",
            "getActiveSourceMbs"
        ])
    },
    watch: {
        getCurrentObjects() {
        // make sure that a set filter doesn't show an empty object list
            if (this.getCurrentObjects.length === 0 && this.filter) {
                this.filter = undefined;
            }
            let currentUuids = this.getCurrentObjects.map(o => o.uuid);
            this.selected = this.selected.filter(uuid => currentUuids.includes(uuid));
        }
    },
    mounted() {
        const baseSettings = Object.entries(this.getSettings).filter(entry => entry[0].startsWith("base")).reduce((obj, [key, value]) => {
            obj[key] = value;
            return obj;
        }, {});

        const storedBaseSettingsHash = this.getSetting("user_settings_hash");
        const currentBaseSettingsHash = ObjectHash(JSON.stringify(baseSettings));

        if (storedBaseSettingsHash !== currentBaseSettingsHash) {
            this.setLocalSetting({key: "user_settings_hash", value: currentBaseSettingsHash });

            Sentry.setUser({
                id: currentBaseSettingsHash,
                username: baseSettings.base_user_name,
                email: baseSettings.base_user_mail,
            });

            Sentry.setTag("org.name", baseSettings.base_orga_name);
            Sentry.setTag("org.full", baseSettings.base_orga_fullname);
            Sentry.setTag("org.id", baseSettings.base_orga_id);

            Sentry.setTag("locale", this.getCurrentLocale);

            Sentry.captureMessage("UsageTag", "debug");
        }

    },
    methods: {
        ...util,
        isOcrSupported(item) {
            const supportedFormats = [ "application/pdf", "image/jpeg", "image/png", "image/svg+xml", "image/gif" ];
            const noText = item.textCount <= 5;
            const isSupported = supportedFormats.includes(item.sourceType);
            return noText && isSupported;
        },
        isPdfaSupported(item) {
            const conformance = util.getMetadataValueAsArray(item, "plus:Conformity");
            const hasConformance = conformance?.length > 0;
            const isSupported = item.sourceType === "application/pdf";
            return !hasConformance && isSupported;
        },
        isOfficeSupported(item) {
            const supportedFormats = [
                "application/vnd.openxmlformats-officedocument.wordprocessingml.document",
                "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet",
                "application/vnd.openxmlformats-officedocument.presentationml.presentation",
                "application/vnd.ms-excel",
                "application/msword",
                "application/vnd.ms-powerpoint",
                "text/plain",
                "application/rtf",
                "text/csv"
            ];
            return supportedFormats.includes(item.sourceType);
        },
        async deleteSelectedObjects() {
            if (await this.$confirm.open(
                this.$t("Actions.deleteSelectedObjects"),
                this.$t("Actions.deleteObjectsInfo")
            )) {
                const objectUuids = this.selected.map(item => item.uuid);
                await this.deleteObjects(objectUuids);
            }
        },
        async addFiles(uploadedFiles) {
            const batchSizeInMb = Array.from(uploadedFiles).reduce((size, file) => {
                size += file.size;
                return size;
            }, 0) / (1000 * 1000);

            if ((batchSizeInMb + this.getActiveSourceMbs) > config.maxObjectMemory) {
                return this.$notify.send(this.$t("App.filesTooLarge"), "warning", 5);
            }

            try {
                this.loading = true;
                let objectUuids = await Promise.all([...uploadedFiles].map(this.handleFile));
            } finally {
                this.$refs.fileForm.reset();
                this.loading = false;
            }
        },
        async handleFile(file) {
            let mimeType = match.mimeType(this.$store, file.type, file.name);
            let objectType = match.objectType(this.$store, file.type, file.name, this.objecttype);

            const predefined = [
                {
                    id: "vdi:revision",
                    value: 1
                },
                {
                    id: "vdi:Language",
                    value: [match.language(this.$store, this.getCurrentLocale)]
                }
            ];

            const guidelines = this.getGuidelines;

            let meta = [...guidelines, ...predefined].reduce((prev, gl) => {
                const uri = this.resolveAssignedRelationType(gl.id) ?? gl.id;
                prev[uri] = template.metadata({
                    uri,
                    value: gl.value
                });

                return prev;
            }, {});


            let object = template.object({
                type: objectType,
                name: file.name,
                source: {
                    type: mimeType,
                    data: file,
                    size: file.size,
                    name: file.name,
                    uri: undefined
                },
                meta
            });

            this.processing.push(object.uuid);

            this.saveObjectLocal(object);
            this.addObjectsToProject({
                projectUuid: this.getCurrentProjectUuid,
                objectUuids: [object.uuid]
            });

            let analyzePayload = [
                this.getCurrentProjectUuid,
                object.uuid,
                object.source.data,
                object.source.name,
                this.$store
            ];

            if (object.source.type === "application/pdf") {
                await pdf.analyze(...analyzePayload);
            }

            this.processing.splice(this.processing.indexOf(object.uuid), 1);
            return object.uuid;
        },
        async dropFile(event) {
            const files = event.dataTransfer.files;
            this.showDragOverlay = false;
            await util.waitForUI();

            this.addFiles(files);
        },
        showOverlay(e) {
            this.toggleOverlay$.next(e);
        },
        ...mapActions("storage", [
            "deleteObjects",
            "saveObjectLocal"
        ]),
        ...mapActions("projects", [
            "addObjectsToProject"
        ]),
        ...mapActions("settings", [
            "setLocalSetting"
        ])
    }
};
</script>

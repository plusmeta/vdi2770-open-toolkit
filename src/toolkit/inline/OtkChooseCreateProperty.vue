<!--
  Copyright 2022 plusmeta GmbH
  License: MIT
-->

<template>
  <v-layout>
    <v-flex>
      <v-form ref="form">
        <v-combobox
          :value="getAssignedProperties"
          :items="getProperties"
          :label="getLabel"
          :prepend-icon="icon"
          :multiple="multiple"
          :rules="[checkRequired]"
          :small-chips="multiple"
          :deletable-chips="multiple"
          :search-input.sync="search"
          return-object
          :class="{ 'required': required }"
          :readonly="isReadonly"
          @input="selectProperty"
        >
          <template v-slot:item="{ item }">
            {{ item.text }}
          </template>
          <template v-slot:no-data>
            <v-list-item class="py-0">
              <v-list-item-content>
                <span v-html="$t('Actions.createEntry')" />
              </v-list-item-content>
            </v-list-item>
          </template>
          <template v-slot:selection="data">
            <PropertyPanel :data="data" :icon="icon" />
          </template>
        </v-combobox>
      </v-form>
    </v-flex>
  </v-layout>
</template>

<script>
import { mapGetters, mapActions } from "vuex";

import template from "@/store/properties/template";

import PropertyPanel from "@/shared/inline/PropertyPanel";

export default {
    name: "ChooseCreateProperty",
    components: {
        PropertyPanel
    },
    props: {
        objectUuid: {
            type: String,
            required: true
        },
        propclass: {
            type: String,
            required: true
        },
        proprelation: {
            type: String,
            required: true
        },
        multiple: {
            type: Boolean,
            default: true
        },
        required: {
            type: Boolean,
            default: false
        },
        label: {
            type: Boolean,
            default: false
        },
        icon: {
            type: String,
            default: ""
        }
    },
    data () {
        return {
            loading: false,
            search: null
        };
    },
    computed: {
        isReadonly() {
            return this.getPropertyAttributeById(this.propclass, "plus:readonly") ?? false;
        },
        getLabel() {
            let label = "";
            if (this.label) label += this.getPropertyLabelById(this.propclass);
            return label;
        },
        getProperties() {
            return this.getInstancesByClassOrRole(this.propclass).map((prop) => {
                return {
                    text: this.getPropertyLabelById(prop.identifier)
                    || `${prop.identifier} [${this.$t("Common.unknown")}]`,
                    value: prop.identifier
                };
            }).sort((a,b) => a.text.localeCompare(b.text));
        },
        getAssignedProperties() {
            let properties = this.getMetadataValueByURI(this.objectUuid, this.proprelation);
            if (this.multiple && !!properties && typeof properties === "string") {
                return [{
                    value: properties,
                    text: this.getPropertyLabelById(properties)
                    || `${prop.identifier} [${this.$t("Common.unknown")}]`
                }];
            } else if (!!properties && typeof properties === "string") {
                return {
                    value: properties,
                    text: this.getPropertyLabelById(properties)
                    || `${prop.identifier} [${this.$t("Common.unknown")}]`
                };
            } else if (!!properties && Array.isArray(properties)) {
                return properties.map((identifier) => {
                    return {
                        value: identifier,
                        text: this.getPropertyLabelById(identifier)
                        || `${identifier} [${this.$t("Common.unknown")}]`
                    };
                });
            } else {
                return (this.multiple) ? [] : undefined;
            }
        },
        ...mapGetters("storage", [
            "getMetadataValueByURI"
        ]),
        ...mapGetters("properties", [
            "getPropertyLabelById",
            "getInstancesByClassOrRole",
            "getPropertyAttributeById"
        ])
    },
    mounted() {
        this.$nextTick(() => this.$refs?.form?.validate());
    },
    methods: {
        selectProperty(selected) {
            let value = undefined;
            if (this.multiple && Array.isArray(selected)) {
                // to prevent search string from beeing interpreted as ad-hoc property
                // if more objects in selection delete search string from selection
                // if objects are the same, then search string is ad-hoc property
                let onlyObjects = selected.filter(selection => typeof selection !== "string");
                let isSelectedObject = onlyObjects.length > this.getAssignedProperties.length;
                if (isSelectedObject) {
                    selected = onlyObjects;
                    this.search = null;
                }
                // go through all values of selection and decide if select or create property
                value = selected.map((selection) => {
                    if (!!selection && typeof selection === "string") {
                        return this.createAdHocProperty(selection);
                    } else if (!!selection && typeof selection === "object" && selection.hasOwnProperty("value")) {
                        this.search = null;
                        return selection.value;
                    } else {
                        return undefined;
                    }
                }).filter(Boolean);

            } else {
                if (!!selected && typeof selected === "string") {
                    value = this.createAdHocProperty(selected);
                } else if (!!selected && typeof selected === "object" && selected.hasOwnProperty("value")) {
                    value = selected.value;
                }
            }

            if (value) {
                this.saveMetaDatum({
                    objectUuid: this.objectUuid,
                    objectMeta: {
                        uri: this.proprelation,
                        value
                    }
                });
            }
        },
        createAdHocProperty(label) {
            let newProperty = template({
                datatype: "plus:Instance",
                subClassOf: this.propclass,
                rels: { "plus:has-roles": ["plus:CustomMetadata"] },
                label
            });

            this.createProperty([newProperty]);

            this.$notify.send(this.$t("Notification.propertyCreated"), "success");

            return newProperty.identifier;
        },
        checkRequired(value) {
            if (this.required && !value.length) {
                return this.$t("Validations.noEmptyInput");
            } else {
                return true;
            }
        },
        ...mapActions("storage", [
            "saveMetaDatum"
        ]),
        ...mapActions("properties", [
            "createProperty"
        ])
    }
};
</script>

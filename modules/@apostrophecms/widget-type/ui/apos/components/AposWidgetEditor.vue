<template>
  <AposModal
    class="apos-widget-editor"
    :modal="modal" :modal-title="editLabel"
    @inactive="modal.active = false" @show-modal="modal.showModal = true"
    @esc="confirmAndCancel" @no-modal="$emit('safe-close')"
  >
    <template #breadcrumbs>
      <AposModalBreadcrumbs
        v-if="breadcrumbs && breadcrumbs.length" :items="breadcrumbs"
      />
    </template>
    <template #main>
      <AposModalBody>
        <template #bodyMain>
          <div class="apos-widget-editor__body">
            <AposSchema
              :trigger-validation="triggerValidation"
              :schema="schema"
              :value="docFields"
              @input="updateDocFields"
              :following-values="followingValues()"
              :conditional-fields="conditionalFields()"
              ref="schema"
            />
          </div>
        </template>
      </AposModalBody>
    </template>
    <template #footer>
      <AposButton
        type="default" label="apostrophe:cancel"
        @click="confirmAndCancel"
      />
      <AposButton
        type="primary" @click="save"
        :label="saveLabel"
        :disabled="docFields.hasErrors"
      />
    </template>
  </AposModal>
</template>

<script>
import AposModifiedMixin from 'Modules/@apostrophecms/ui/mixins/AposModifiedMixin';
import AposEditorMixin from 'Modules/@apostrophecms/modal/mixins/AposEditorMixin';
import { detectDocChange } from 'Modules/@apostrophecms/schema/lib/detectChange';
import cuid from 'cuid';
import { klona } from 'klona';

export default {
  name: 'AposWidgetEditor',
  mixins: [ AposModifiedMixin, AposEditorMixin ],
  props: {
    type: {
      required: true,
      type: String
    },
    breadcrumbs: {
      type: Array,
      default: function () {
        return [];
      }
    },
    options: {
      required: true,
      type: Object
    },
    value: {
      required: false,
      type: Object,
      default() {
        return {};
      }
    }
  },
  emits: [ 'safe-close', 'modal-result' ],
  data() {
    return {
      id: this.value && this.value._id,
      original: null,
      docFields: {
        data: { ...this.value },
        hasErrors: false
      },
      modal: {
        title: this.editLabel,
        active: false,
        type: 'slide',
        showModal: false
      },
      triggerValidation: false
    };
  },
  computed: {
    moduleOptions() {
      return window.apos.modules[apos.area.widgetManagers[this.type]];
    },
    typeLabel() {
      return this.moduleOptions.label;
    },
    editLabel() {
      if (this.moduleOptions.editLabel) {
        return this.moduleOptions.editLabel;
      } else {
        return {
          key: 'apostrophe:editType',
          type: this.$t(this.typeLabel)
        };
      }
    },
    saveLabel() {
      if (this.moduleOptions.saveLabel) {
        return this.moduleOptions.saveLabel;
      } else {
        return {
          key: 'apostrophe:saveType',
          type: this.$t(this.typeLabel)
        };
      }
    },
    schema() {
      return (this.moduleOptions.schema || []).filter(field => apos.schema.components.fields[field.type]);
    },
    isModified() {
      return detectDocChange(this.schema, this.original, this.docFields.data);
    }
  },
  async mounted() {
    this.modal.active = true;
  },
  created() {
    this.original = this.value ? klona(this.value) : this.getDefault();
  },
  methods: {
    updateDocFields(value) {
      this.docFields = value;
    },
    save() {
      this.triggerValidation = true;
      this.$nextTick(async () => {
        if (this.docFields.hasErrors) {
          this.triggerValidation = false;
          return;
        }
        const widget = this.docFields.data;
        if (!widget.type) {
          widget.type = this.type;
        }
        if (!this.id) {
          widget._id = cuid();
        }
        this.$emit('modal-result', widget);
        this.modal.showModal = false;
      });
    },
    getDefault() {
      const widget = {};
      this.schema.forEach(field => {
        widget[field.name] = field.def ? klona(field.def) : field.def;
      });
      return widget;
    }
  }
};
</script>

<style lang="scss" scoped>
  .apos-widget-editor ::v-deep .apos-modal__inner {
    max-width: 458px;
  }
</style>

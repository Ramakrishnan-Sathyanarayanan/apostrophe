<template>
  <div
    class="apos-media-editor"
    :class="{
      'apos-is-replacing': showReplace
    }"
  >
    <div class="apos-media-editor__inner" v-if="activeMedia">
      <div class="apos-media-editor__thumb-wrapper">
        <img
          v-if="activeMedia.attachment && activeMedia.attachment._urls"
          class="apos-media-editor__thumb"
          :src="activeMedia.attachment._urls[restoreOnly ? 'one-sixth' : 'one-third']" :alt="activeMedia.description"
        >
      </div>
      <ul class="apos-media-editor__details">
        <li class="apos-media-editor__detail" v-if="createdDate">
          {{ $t('apostrophe:mediaCreatedDate', { createdDate }) }}
        </li>
        <li class="apos-media-editor__detail" v-if="fileSize">
          {{ $t('apostrophe:mediaFileSize', { fileSize }) }}
        </li>
        <li
          class="apos-media-editor__detail"
          v-if="activeMedia.attachment && activeMedia.attachment.width"
        >
          {{
            $t('apostrophe:mediaDimensions', {
              width: activeMedia.attachment.width,
              height: activeMedia.attachment.height
            })
          }}
        </li>
      </ul>
      <ul class="apos-media-editor__links">
        <li class="apos-media-editor__link" aria-hidden="true">
          <AposButton
            type="quiet" label="apostrophe:replace"
            @click="showReplace = true"
            :disabled="isArchived"
          />
        </li>
        <li class="apos-media-editor__link" v-if="activeMedia.attachment && activeMedia.attachment._urls">
          <AposButton
            type="quiet" label="apostrophe:view"
            @click="viewMedia"
            :disabled="isArchived"
          />
        </li>
        <li class="apos-media-editor__link" v-if="activeMedia.attachment && activeMedia.attachment._urls">
          <AposButton
            type="quiet" label="apostrophe:download"
            :href="!isArchived ? activeMedia.attachment._urls.original : false"
            :disabled="isArchived"
            download
          />
        </li>
      </ul>
      <AposSchema
        v-if="docFields.data.title !== undefined"
        :schema="schema"
        v-model="docFields"
        :modifiers="['small', 'inverted']"
        :trigger-validation="triggerValidation"
        :doc-id="docFields.data._id"
        :following-values="followingValues()"
        @reset="$emit('modified', false)"
        ref="schema"
        :server-errors="serverErrors"
      />
    </div>
    <AposModalLip :refresh="lipKey">
      <div
        class="apos-media-editor__lip"
      >
        <AposContextMenu
          v-if="!restoreOnly"
          :button="{
            label: 'apostrophe:moreOperations',
            iconOnly: true,
            icon: 'dots-vertical-icon',
            type: 'subtle',
            modifiers: [ 'small', 'no-motion' ]
          }"
          :menu="moreMenu"
          @item-clicked="moreMenuHandler"
          menu-placement="top-end"
        />
        <AposButton
          @click="save" class="apos-media-editor__save"
          :disabled="docFields.hasErrors"
          :label="restoreOnly ? 'apostrophe:restore' : 'apostrophe:save'" type="primary"
        />
      </div>
    </AposModalLip>
  </div>
</template>

<script>
import AposEditorMixin from 'Modules/@apostrophecms/modal/mixins/AposEditorMixin';
import AposAdvisoryLockMixin from 'Modules/@apostrophecms/ui/mixins/AposAdvisoryLockMixin';
import { detectDocChange } from 'Modules/@apostrophecms/schema/lib/detectChange';
import { klona } from 'klona';
import dayjs from 'dayjs';
import { isEqual } from 'lodash';
import advancedFormat from 'dayjs/plugin/advancedFormat';
import cuid from 'cuid';

dayjs.extend(advancedFormat);

export default {
  mixins: [ AposEditorMixin, AposAdvisoryLockMixin ],
  props: {
    media: {
      type: Object,
      default() {
        return {};
      }
    },
    selected: {
      type: Array,
      default() {
        return [];
      }
    },
    moduleLabels: {
      type: Object,
      default() {
        return {
          label: 'Image',
          pluralLabel: 'Images'
        };
      }
    }
  },
  emits: [ 'saved', 'back', 'modified' ],
  data() {
    return {
      // Primarily use `activeMedia` to support hot-swapping image docs.
      activeMedia: klona(this.media),
      restoreOnly: this.media && this.media.archived,
      // Unlike `activeMedia` this changes ONLY when a new doc is swapped in.
      // For overall change detection.
      original: klona(this.media),
      lipKey: '',
      triggerValidation: false,
      showReplace: false
    };
  },
  computed: {
    moduleOptions() {
      return window.apos.modules[this.activeMedia.type] || {};
    },
    moreMenu() {
      const menu = [ {
        label: 'apostrophe:discardChanges',
        action: 'cancel'
      } ];
      if (this.activeMedia._id && !this.restoreOnly) {
        menu.push({
          label: 'apostrophe:archiveImage',
          action: 'archive',
          modifiers: [ 'danger' ]
        });
      }
      return menu;
    },
    fileSize() {
      if (
        !this.activeMedia.attachment || !this.activeMedia.attachment.length
      ) {
        return '';
      }
      const size = this.activeMedia.attachment.length;
      const formatter = new Intl.NumberFormat(apos.locale, {
        maximumFractionDigits: 2
      });
      if (size >= 1000000) {
        const formatted = formatter.format(size / 1000000);
        return this.$t('apostrophe:mediaMB', {
          size: formatted
        });
      } else {
        const formatted = formatter.format(size / 1000);
        return this.$t('apostrophe:mediaKB', {
          size: formatted
        });
      }
    },
    createdDate() {
      if (!this.activeMedia.attachment || !this.activeMedia.attachment.createdAt) {
        return '';
      }
      return dayjs(this.activeMedia.attachment.createdAt).format(this.$t('apostrophe:dayjsMediaCreatedDateFormat'));
    },
    isArchived() {
      return this.media.archived;
    }
  },
  watch: {
    'docFields.data': {
      handler(newData, oldData) {
        this.$nextTick(() => {
          // If either old or new state are an empty object, it's not "modified."
          if (!(Object.keys(oldData).length > 0 && Object.keys(newData).length > 0)) {
            this.$emit('modified', false);
          } else {
            this.$emit('modified', detectDocChange(this.schema, this.original, newData));
          }
        });

        if ((this.activeMedia.attachment && !newData.attachment)) {
          this.updateActiveAttachment({});
        } else if (
          (newData.attachment && !this.activeMedia.attachment) ||
          (this.activeMedia.attachment && !newData.attachment) ||
          !isEqual(newData.attachment, this.activeMedia.attachment)
        ) {
          this.updateActiveAttachment(newData.attachment);
        }
      }
    },
    media(newVal) {
      this.updateActiveDoc(newVal);
    }
  },
  mounted() {
    this.generateLipKey();
    this.$emit('modified', false);
  },
  methods: {
    moreMenuHandler(action) {
      this[action]();
    },
    async updateActiveDoc(newMedia) {
      this.showReplace = false;
      this.activeMedia = klona(newMedia);
      this.restoreOnly = !!this.activeMedia.archived;
      this.original = klona(newMedia);
      this.docFields.data = klona(newMedia);
      this.generateLipKey();
      await this.unlock();
      // Distinguish between an actual doc and an empty placeholder
      if (newMedia._id) {
        if (!await this.lock(`${this.moduleOptions.action}/${newMedia._id}`)) {
          this.lockNotAvailable();
        }
      }
    },
    async archive() {
      if (!await apos.confirm({
        heading: 'apostrophe:areYouSure',
        description: 'apostrophe:willMoveImageToArchive'
      })) {
        return;
      }
      const route = `${this.moduleOptions.action}/${this.activeMedia._id}`;
      const patched = await apos.http.patch(route, {
        busy: true,
        body: {
          archived: true
        },
        draft: true
        // Autopublish will take care of the published side
      });
      apos.bus.$emit('content-changed', {
        doc: patched,
        action: 'archive'
      });
      await this.cancel();
    },
    save() {
      this.triggerValidation = true;
      const route = `${this.moduleOptions.action}/${this.activeMedia._id}`;
      // Repopulate `attachment` since it was removed from the schema.
      this.docFields.data.attachment = this.activeMedia.attachment;

      this.$nextTick(async () => {
        if (this.docFields.hasErrors) {
          await apos.notify('apostrophe:resolveErrorsBeforeSaving', {
            type: 'warning',
            icon: 'alert-circle-icon',
            dismiss: true
          });
          return;
        }

        let body = this.docFields.data;
        this.addLockToRequest(body);
        try {
          const requestMethod = this.restoreOnly ? apos.http.patch : apos.http.put;
          if (this.restoreOnly) {
            body = {
              archived: false
            };
          }
          const doc = await requestMethod(route, {
            busy: true,
            body,
            draft: true
          });
          apos.bus.$emit('content-changed', {
            doc,
            action: 'update'
          });
          this.original = klona(this.docFields.data);
          this.$emit('modified', false);
          this.$emit('saved');
        } catch (e) {
          if (this.isLockedError(e)) {
            await this.showLockedError(e);
            this.lockNotAvailable();
          } else {
            await this.handleSaveError(e, {
              fallback: `Error ${this.restoreOnly ? 'Restoring' : 'Saving'} ${this.moduleLabels.label}`
            });
          }
        } finally {
          this.showReplace = false;
        }
      });
    },
    generateLipKey() {
      this.lipKey = cuid();
    },
    cancel() {
      this.showReplace = false;
      this.$emit('back');
    },
    lockNotAvailable() {
      this.isModified = false;
      this.cancel();
    },
    updateActiveAttachment(attachment) {
      console.info('☄️', attachment);
      this.activeMedia.attachment = attachment;
    },
    viewMedia () {
      window.open(this.activeMedia.attachment._urls.original, '_blank');
    }
  }
};
</script>

<style lang="scss" scoped>
  .apos-media-editor {
    position: relative;
    height: 100%;
    padding: 20px;
  }

  .apos-media-editor__thumb-wrapper {
    display: flex;
    justify-content: center;
    align-items: center;
    height: 180px;
    border: 1px solid var(--a-base-7);
    margin-bottom: 20px;
  }
  .apos-media-editor__thumb {
    max-width: 100%;
    max-height: 100%;
  }

  .apos-media-editor ::v-deep .apos-field {
    margin-bottom: $spacing-double;
  }

  .apos-media-editor__details {
    @include apos-list-reset();
    margin-bottom: $spacing-double;
  }

  .apos-media-editor__detail {
    @include type-base;
    line-height: var(--a-line-tallest);
    color: var(--a-base-4);
  }

  .apos-media-editor__links {
    @include apos-list-reset();
    display: flex;
    margin-bottom: $spacing-triple;

    ::v-deep .apos-button--quiet {
      display: inline;
    }
  }

  .apos-media-editor__link {
    display: inline-block;
    & + & {
      margin-left: 20px;
    }
  }

  ::v-deep [data-apos-field='attachment'] {
    .apos-media-editor:not(.apos-is-replacing) & {
      position: absolute;
      left: -999rem;
      opacity: 0;
    }
  }

  .apos-media-editor__controls {
    margin-bottom: 20px;
  }

  .apos-media-editor__lip {
    display: flex;
    justify-content: flex-end;
    & > .apos-context-menu, & > .apos-button__wrapper {
      margin-left: 7.5px;
    }
  }
</style>

<template>
  <table class="apos-table">
    <tbody>
      <tr>
        <th
          class="apos-table__header"
          v-if="!options.hideCheckboxes"
        />
        <th
          v-for="header in headers" scope="col"
          class="apos-table__header" :key="header.label"
          :class="`apos-table__header--${header.name}`"
        >
          <component
            :is="getEl(header)" class="apos-table__header-label"
          >
            <component
              v-if="header.labelIcon"
              :is="icons[header.labelIcon]"
              :size="iconSize(header)"
              class="apos-table__header-icon"
            />
            {{ $t(header.label) }}
          </component>
        </th>
        <th class="apos-table__header" key="contextMenu">
          <component
            :is="getEl({})"
            class="apos-table__header-label is-hidden"
          >
            {{ $t('apostrophe:moreOperations') }}
          </component>
        </th>
      </tr>
      <tr
        class="apos-table__row"
        v-for="item in items"
        :key="item._id"
        :class="{'apos-is-selected': false }"
        @mouseover="over(item._id)" @mouseout="out(item._id)"
      >
        <td
          class="apos-table__cell"
          v-if="!options.hideCheckboxes"
        >
          <AposCheckbox
            v-if="item._id"
            :field="{
              name: item._id,
              hideLabel: true,
              label: `Toggle selection of ${item.title}`,
              readOnly:
                (options.disableUnchecked && !checkProxy.includes(item._id)) ||
                (options.disableUnpublished && !item.lastPublishedAt)
            }"
            v-apos-tooltip="options.disableUnpublished && !item.lastPublishedAt ? 'apostrophe:publishBeforeUsingTooltip' : null"
            :choice="{ value: item._id }"
            v-model="checkProxy"
            @updated="emitUpdated(item._id)"
          />
        </td>
        <td
          v-for="header in headers"
          class="apos-table__cell apos-table__cell--pointer"
          :class="`apos-table__cell--${header.name}`"
          :key="header.name"
          @click="options.canEdit && $emit('open', item)"
        >
          <component
            v-if="header.component" :is="header.component"
            :header="header" :item="item._publishedDoc || item"
            :draft="item" :published="item._publishedDoc"
            :manually-published="options.manuallyPublished"
          />
          <AposCellBasic
            v-else
            :header="header"
            :draft="item"
            :published="item._publishedDoc"
          />
        </td>
        <!-- append the context menu -->
        <td v-if="options.canEdit" class="apos-table__cell apos-table__cell--context-menu">
          <AposCellContextMenu
            :state="state[item._id]" :item="item"
            :draft="item"
            :published="item._publishedDoc"
            :header="{
              columnHeader: '',
              property: 'contextMenu',
              component: 'AposCellContextMenu'
            }"
            :options="contextMenuOptions"
          />
        </td>
      </tr>
    </tbody>
  </table>
</template>

<script>
import { klona } from 'klona';
export default {
  model: {
    prop: 'checked',
    event: 'change'
  },
  props: {
    headers: {
      type: Array,
      required: true
    },
    items: {
      type: Array,
      required: true
    },
    checked: {
      type: Array,
      default() {
        return [];
      }
    },
    options: {
      type: Object,
      default() {
        return {};
      }
    }
  },
  emits: [
    'open',
    'change',
    'updated'
  ],
  data() {
    const state = {
      _template: {
        hover: false
      }
    };
    this.items.forEach(item => {
      state[item._id] = klona(state._template);
    });
    return {
      state
    };
  },
  computed: {
    checkProxy: {
      get() {
        return this.checked;
      },
      set(val) {
        this.$emit('change', val);
      }
    },
    contextMenuOptions() {
      return this.options;
    }
  },
  watch: {
    items() {
      this.refreshState();
    }
  },
  methods: {
    over(id) {
      this.state[id].hover = true;
    },
    out(id) {
      this.state[id].hover = false;
    },
    refreshState() {
      const template = klona(this.state._template);
      const state = {};
      this.items.forEach(item => {
        state[item._id] = klona(template);
      });
      state._template = template;
      this.state = state;
    },
    getEl(header) {
      if (header.action) {
        return 'button';
      } else {
        return 'span';
      }
    },
    // TODO: Factor this out to relationship manager.
    emitUpdated($event) {
      this.$emit('updated', $event);
    }
  }
};
</script>

<style lang="scss" scoped>
  .apos-table__cell--pointer {
    cursor: pointer;
  }
</style>

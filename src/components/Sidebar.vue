<template>
  <div
    class="py-2 h-full flex justify-between flex-col bg-gray-25 relative"
    :class="{
      'window-drag': platform !== 'Windows',
    }"
  >
    <div>
      <!-- Company name and DB Switcher -->
      <div
        class="px-4 flex flex-row items-center justify-between mb-4"
        :class="
          platform === 'Mac' && languageDirection === 'ltr' ? 'mt-10' : 'mt-2'
        "
      >
        <h6
          class="
            font-semibold
            whitespace-nowrap
            overflow-auto
            no-scrollbar
            select-none
          "
        >
          {{ companyName }}
        </h6>
      </div>

      <!-- Sidebar Items -->
      <div v-for="group in groups" :key="group.label">
        <div
          class="px-4 flex items-center cursor-pointer hover:bg-gray-100 h-10"
          :class="
            isGroupActive(group) && !group.items
              ? 'bg-gray-100 border-s-4 border-blue-500'
              : ''
          "
          @click="onGroupClick(group)"
        >
          <Icon
            class="flex-shrink-0"
            :name="group.icon"
            :size="group.iconSize || '18'"
            :height="group.iconHeight"
            :active="isGroupActive(group)"
            :class="isGroupActive(group) && !group.items ? '-ms-1' : ''"
          />
          <div
            class="ms-2 text-lg text-gray-900"
            :class="isGroupActive(group) && !group.items && 'text-blue-600'"
          >
            {{ group.label }}
          </div>
        </div>

        <!-- Expanded Group -->
        <div v-if="group.items && isGroupActive(group)">
          <div
            v-for="item in group.items"
            :key="item.label"
            class="
              text-base text-gray-800
              h-10
              ps-10
              cursor-pointer
              flex
              items-center
              hover:bg-gray-100
            "
            :class="
              isItemActive(item)
                ? 'bg-gray-100 text-blue-600 border-s-4 border-blue-500'
                : ''
            "
            @click="onItemClick(item)"
          >
            <p :style="isItemActive(item) ? 'margin-left: -4px' : ''">
              {{ item.label }}
            </p>
          </div>
        </div>
      </div>
    </div>

    <!-- Report Issue and App Version -->
    <div class="window-no-drag flex flex-col gap-2 py-2 px-4">
      <button
        class="
          flex
          text-sm text-gray-600
          hover:text-gray-800
          gap-1
          items-center
        "
        @click="openDocumentation"
      >
        <feather-icon name="help-circle" class="h-4 w-4 flex-shrink-0" />
        <p>
          {{ t`Help` }}
        </p>
      </button>

      <button
        class="
          flex
          text-sm text-gray-600
          hover:text-gray-800
          gap-1
          items-center
        "
        @click="$emit('change-db-file')"
      >
        <feather-icon name="database" class="h-4 w-4 flex-shrink-0" />
        <p>{{ t`Change DB` }}</p>
      </button>

      <button
        class="
          flex
          text-sm text-gray-600
          hover:text-gray-800
          gap-1
          items-center
        "
        @click="() => reportIssue()"
      >
        <feather-icon name="flag" class="h-4 w-4 flex-shrink-0" />
        <p>
          {{ t`Report Issue` }}
        </p>
      </button>

      <p
        v-if="fyo.store.isDevelopment"
        class="text-xs text-gray-500 select-none"
      >
        dev mode
      </p>
    </div>

    <!-- Hide Sidebar Button -->
    <button
      class="
        absolute
        bottom-0
        end-0
        text-gray-600
        hover:bg-gray-100
        rounded
        p-1
        m-4
        rtl-rotate-180
      "
      @click="$emit('toggle-sidebar')"
    >
      <feather-icon name="chevrons-left" class="w-4 h-4" />
    </button>
  </div>
</template>
<script>
import Button from 'src/components/Button.vue';
import { reportIssue } from 'src/errorHandling';
import { fyo } from 'src/initFyo';
import { openLink } from 'src/utils/ipcCalls';
import { getSidebarConfig } from 'src/utils/sidebarConfig';
import { docsPath, routeTo } from 'src/utils/ui';
import router from '../router';
import Icon from './Icon.vue';

export default {
  components: [Button],
  inject: ['languageDirection'],
  emits: ['change-db-file', 'toggle-sidebar'],
  data() {
    return {
      companyName: '',
      groups: [],
      activeGroup: null,
    };
  },
  computed: {
    appVersion() {
      return fyo.store.appVersion;
    },
  },
  components: {
    Icon,
  },
  async mounted() {
    const { companyName } = await fyo.doc.getDoc('AccountingSettings');
    this.companyName = companyName;
    this.groups = await getSidebarConfig();

    this.setActiveGroup();
    router.afterEach(() => {
      this.setActiveGroup();
    });
  },
  methods: {
    routeTo,
    reportIssue,
    openDocumentation() {
      openLink('https://docs.frappebooks.com/' + docsPath.value);
    },
    setActiveGroup() {
      const { fullPath } = this.$router.currentRoute.value;
      const fallBackGroup = this.activeGroup;
      this.activeGroup = this.groups.find((g) => {
        if (fullPath.startsWith(g.route) && g.route !== '/') {
          return true;
        }

        if (g.route === fullPath) {
          return true;
        }

        if (g.items) {
          let activeItem = g.items.filter(
            ({ route }) => route === fullPath || fullPath.startsWith(route)
          );

          if (activeItem.length) {
            return true;
          }
        }
      });

      if (!this.activeGroup) {
        this.activeGroup = fallBackGroup || this.groups[0];
      }
    },
    isItemActive(item) {
      const { path: currentRoute, params } = this.$route;
      const routeMatch = currentRoute === item.route;

      const schemaNameMatch =
        item.schemaName && params.schemaName === item.schemaName;

      const isMatch = routeMatch || schemaNameMatch;
      if (params.name && item.schemaName && !isMatch) {
        return currentRoute.includes(`${item.schemaName}/${params.name}`);
      }

      return isMatch;
    },
    isGroupActive(group) {
      return this.activeGroup && group.label === this.activeGroup.label;
    },
    onGroupClick(group) {
      if (group.action) {
        group.action();
      }

      if (group.route) {
        routeTo(this.getPath(group));
      }
    },
    onItemClick(item) {
      if (item.action) {
        item.action();
      }

      if (item.route) {
        routeTo(this.getPath(item));
      }
    },
    getPath(item) {
      const { route: path, filters } = item;
      if (!filters) {
        return path;
      }

      return { path, query: { filters: JSON.stringify(filters) } };
    },
  },
};
</script>

<!--
This Source Code Form is subject to the terms of the Mozilla Public
License, v. 2.0. If a copy of the MPL was not distributed with this
file, You can obtain one at http://mozilla.org/MPL/2.0/.
-->
<template>
  <div class="flex flex-col px-6 py-3">
    <div class="flex items-center">
      <div class="flex flex-1 items-center grid grid-cols-5 gap-4">
        <div class="col-span-2 block flex items-center gap-3">
          <button type="button" @click="expand" class="relative inline-flex items-center px-1 py-1 text-sm font-medium leading-5 transition-colors duration-200 rounded-full select-none text-talos-gray-700 dark:text-talos-gray-400 bg-talos-gray-200 dark:bg-talos-gray-800 hover:bg-talos-gray-200 dark:hover:bg-talos-gray-700 focus:outline-none focus:ring-2 focus:ring-blue-600 focus:ring-opacity-60">
            <div :class="{ 'w-5': true, 'h-5': true, transform: expanded, 'rotate-180': expanded }">
              <chevron-down-icon class="w-5 h-5"/>
            </div>
          </button>
          <div class="flex-1 block truncate overflow-hidden">
            <p
              class="text-sm font-medium truncate text-talos-gray-900 dark:text-talos-gray-100"
              >
              {{ item.metadata.name }}
            </p>
            <p
              v-if="cpu"
              class="text-xs leading-4 text-talos-gray-500"
              >
              {{ cpu }}
            </p>
          </div>
        </div>
        <div class="block">
          <p
            class="text-sm truncate text-talos-gray-900 dark:text-talos-gray-100"
            >
            {{ created }}
          </p>
        </div>
        <div class="block">
          <p
            class="text-sm truncate text-talos-gray-900 dark:text-talos-gray-100 space-x-1 space-y-1 flex items-center"
            >
            <light-bulb-icon fill="currentColor" :class="{
              'w-5': true,
              'h-5': true,
              'mr-1': true,
              'on': (item.status || {}).power === 'on',
              'off': (item.status || {power: 'off'}).power === 'off'
              }"/>
            <span class="text-sm text-talos-gray-900 dark:text-talos-gray-100 capitalize">
              {{ (item.status || {power: 'off'}).power }}
            </span>
          </p>
        </div>
        <div class="hidden md:block">
          <span :class="{
              tag: true,
              uppercase: true,
              ready: item.status.ready,
              'not-ready': !item.status.ready,
            }">
            <check-circle-icon v-if="item.status.ready" class="w-5 h-5 mr-1 text-green-400 dark:text-green-300" fill="currentColor"/>
            <exclamation-circle-icon v-else class="w-5 h-5 mr-1 text-red-400 dark:text-red-300" fill="currentColor"/>
            {{ item.status.ready ? 'ready' : 'error' }}
          </span>
        </div>
      </div>
      <div v-on:click.stop.prevent.self class="justify-self-end">
        <t-dropdown xs>
          <template v-slot:icon>
            <dots-horizontal-icon
              class="w-5 h-5 text-talos-gray-900 dark:text-talos-gray-50"
              aria-hidden="true"
              />
          </template>
          <template v-slot:default>
            <menu-item v-slot="{ active }">
              <a
                v-on:click="acceptServer"
                :class="{ active }"
                >Accept Server</a
              >
            </menu-item>
          </template>
        </t-dropdown>
      </div>
    </div>

    <div v-if="expanded" class="block col-span-5">
      <div class="flex items-center grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-1 lg:gap-4">
        <div class="flex-1">
          <div class="label">Labels</div>
          <template v-if="item.metadata.labels">
          <div :class="{ tag: true, 'details-tag': true, 'mt-1': index > 0 }" v-for="value, key, index in item.metadata.labels" :key="key">
            {{ key }}: {{ value }}
          </div>
          </template>
        </div>
        <div class="flex-1">
          <div class="label">Accepted</div>
          <div class="tag details-tag capitalize">
            <check-icon class="w-5 h-5" v-if="item.spec.accepted"/>
            <x-icon class="w-5 h-5" v-else/>
            {{ item.spec.accepted || false }}
          </div>
        </div>
        <div class="flex-1">
          <div class="label">Clean</div>
          <div class="tag details-tag capitalize">
            <check-icon class="w-5 h-5" v-if="item.status && item.status.isClean"/>
            <x-icon class="w-5 h-5" v-else/>
            {{ !!(item.status || {}).isClean }}
          </div>
        </div>
        <div class="flex-1">
          <div class="label">Resource Version</div>
          <div>{{ item.metadata.resourceVersion }}</div>
        </div>
        <div class="flex-1">
          <div class="label">Hostname</div>
          <div>{{ item.spec.hostname }}</div>
        </div>
        <div class="flex-1">
          <div class="label">In Use</div>
          <div class="tag details-tag capitalize">
            <check-icon class="w-5 h-5" v-if="item.status && item.status.inUse"/>
            <x-icon class="w-5 h-5" v-else/>
            {{ !!(item.status || {}).inUse }}
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script type="ts">
import {
  LightBulbIcon,
  CheckCircleIcon,
  ExclamationCircleIcon,
  ChevronDownIcon,
  CheckIcon,
  XIcon,
  DotsHorizontalIcon,
} from '@heroicons/vue/solid';
import TDropdown from './TDropdown.vue';
import { DateTime } from 'luxon';
import { computed, ref, toRefs } from 'vue';
import { ResourceService } from '../api/grpc';
import { KubernetesResourceType } from '../api/resources';
import { Runtime } from '../api/common/theila';
import { showError } from '../modal';
import { MenuItem } from '@headlessui/vue';

export default {
  components: {
    LightBulbIcon,
    CheckCircleIcon,
    ExclamationCircleIcon,
    ChevronDownIcon,
    CheckIcon,
    XIcon,
    DotsHorizontalIcon,
    TDropdown,
    MenuItem,
  },

  props: {
    item: {
      type: Object,
      required: true,
    }
  },

  setup(props) {
    const expanded = ref(false);
    const { item } = toRefs(props);

    const created = computed(() => {
      return DateTime.fromISO(props.item["metadata"]["creationTimestamp"]).toRelative();
    });

    const cpu = computed(() => {
      return (props.item["spec"]["cpu"] || {})["version"];
    });

    return {
      created,
      cpu,
      expanded,
      expand: () => {
        expanded.value = !expanded.value;
      },
      acceptServer: async () => {
        if(item.value["spec"]["accepted"])
          return;

        const spec = {};

        Object.assign(spec, item.value);

        spec.spec["accepted"] = true;

        try {
          await ResourceService.Update({
            metadata: {
              type: KubernetesResourceType,
            },
            spec: spec,
          }, {
            runtime: Runtime.Kubernetes,
          });
        } catch(e) {
          showError(`Failed to accept the server ${item.value["metadata"]["name"]}`, e.toString())
        }
      },
    }
  }
};
</script>

<style scoped>
.ready {
  @apply bg-green-100 text-green-700 dark:bg-green-800 dark:text-green-100;
}

.not-ready {
  @apply bg-red-100 text-red-700 dark:bg-red-800 dark:text-red-100;
}

.tag {
  @apply font-normal inline-flex items-center gap-1 px-2 py-0.5 rounded-md text-sm leading-4 tracking-wider;
}

.details-tag {
  @apply bg-talos-gray-200 text-talos-gray-800 dark:bg-talos-gray-700 dark:text-talos-gray-100;
}

.on {
  @apply text-yellow-500;
}

.off {
  @apply text-gray-400;
}

.label {
  @apply text-sm font-medium leading-5 text-talos-gray-500;
}
</style>

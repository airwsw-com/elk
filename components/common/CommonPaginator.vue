<script setup lang="ts" generic="T, O, U = T">
// @ts-expect-error missing types
import { DynamicScroller } from 'vue-virtual-scroller'
import 'vue-virtual-scroller/dist/vue-virtual-scroller.css'
import type { Paginator, WsEvents } from 'masto'
import type { UnwrapRef } from 'vue'

const {
  paginator,
  stream,
  keyProp = 'id',
  virtualScroller = false,
  eventType = 'update',
  preprocess,
  endMessage = true,
} = defineProps<{
  paginator: Paginator<T[], O>
  keyProp?: keyof T
  virtualScroller?: boolean
  stream?: Promise<WsEvents>
  eventType?: 'notification' | 'update'
  preprocess?: (items: (U | T)[]) => U[]
  endMessage?: boolean | string
}>()

defineSlots<{
  default: (props: {
    items: U[]
    item: U
    index: number
    active?: boolean
    older: U
    newer: U // newer is undefined when index === 0
  }) => void
  items: (props: {
    items: UnwrapRef<U[]>
  }) => void
  updater: (props: {
    number: number
    update: () => void
  }) => void
  loading: (props: {}) => void
  done: (props: { items: U[] }) => void
}>()

const { t } = useI18n()
const nuxtApp = useNuxtApp()

const { items, prevItems, update, state, endAnchor, error } = usePaginator(paginator, $$(stream), eventType, preprocess)
const _error = (error as any) as Error

nuxtApp.hook('elk-logo:click', () => {
  update()
  nuxtApp.$scrollToTop()
})

function createEntry(item: any) {
  items.value = [...items.value, preprocess?.([item]) ?? item]
}

function updateEntry(item: any) {
  const id = item[keyProp]
  const index = items.value.findIndex(i => (i as any)[keyProp] === id)
  if (index > -1)
    items.value = [...items.value.slice(0, index), preprocess?.([item]) ?? item, ...items.value.slice(index + 1)]
}

function removeEntry(entryId: any) {
  items.value = items.value.filter(i => (i as any)[keyProp] !== entryId)
}

const { busy, oauth, singleInstanceServer } = useSignIn()

defineExpose({ createEntry, removeEntry, updateEntry })
</script>

<template>
  <div>
    <slot v-if="prevItems.length" name="updater" v-bind="{ number: prevItems.length, update }" />
    <slot name="items" :items="items">
      <template v-if="virtualScroller">
        <DynamicScroller
          v-slot="{ item, active, index }"
          :items="items"
          :min-item-size="200"
          :key-field="keyProp"
          page-mode
        >
          <slot
            v-bind="{ key: item[keyProp] }"
            :item="item"
            :active="active"
            :older="items[index + 1] as U"
            :newer="items[index - 1] as U"
            :index="index"
            :items="items as U[]"
          />
        </DynamicScroller>
      </template>
      <template v-else>
        <slot
          v-for="item, index of items"
          v-bind="{ key: item[keyProp as keyof U] }"
          :item="item as U"
          :older="items[index + 1] as U"
          :newer="items[index - 1] as U"
          :index="index"
          :items="items as U[]"
        />
      </template>
    </slot>
    <div ref="endAnchor" />
    <slot v-if="state === 'loading'" name="loading">
      <TimelineSkeleton />
    </slot>
    <slot v-else-if="state === 'done' && endMessage !== false" name="done" :items="items as U[]">
      <div p5 text-secondary italic text-center>
        {{ t(typeof endMessage === 'string' ? endMessage : 'common.end_of_list') }}
      </div>
    </slot>
    <div v-else-if="state === 'error'" p5 text-secondary>
      <div v-if="_error.name === 'MastoHttpUnauthorizedError' && singleInstanceServer" p5 flex="~ col gap2">
        <button
          flex="~ row" gap-x-2 items-center justify-center btn-text text-center rounded-3
          :disabled="busy"
          @click="oauth()"
        >
          <span v-if="busy" aria-hidden="true" block animate animate-spin preserve-3d class="rtl-flip">
            <span block i-ri:loader-2-fill aria-hidden="true" />
          </span>
          <span v-else aria-hidden="true" block i-ri:login-circle-line class="rtl-flip" />
          <i18n-t keypath="action.sign_in_to">
            {{ currentServer }}
          </i18n-t>
        </button>
      </div>
      <div v-else text-center italic>
        {{ t('common.error') }}: {{ _error.name }}
        <br>
        {{ _error.message }}
      </div>
    </div>
  </div>
</template>

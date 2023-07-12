<script setup lang="ts">
import type { mastodon } from 'masto'

const params = useRoute().params
const handle = $(computedEager(() => params.account as string))

definePageMeta({ name: 'account-index' })

const { t } = useI18n()

const account = await fetchAccountByHandle(handle)

function reorderAndFilter(items: mastodon.v1.Status[]) {
  return reorderedTimeline(items, 'account')
}

const paginatorPinned = useMastoClient().v1.accounts.listStatuses(account.id, { limit: 30, pinned: true })
const paginator = useMastoClient().v1.accounts.listStatuses(account.id, { limit: 30, excludeReplies: true })

if (account) {
  useHydratedHead({
    title: () => `${t('account.posts')} | ${getDisplayName(account)} (@${account.acct})`,
  })
}
</script>

<template>
  <div>
    <AccountTabs />
    <TimelinePaginator :paginator="paginatorPinned" :preprocess="reorderAndFilter" context="account" :account="account" :end-message="false" :is-pinned="true" />
    <TimelinePaginator :paginator="paginator" :preprocess="reorderAndFilter" context="account" :account="account" />
  </div>
</template>

<!-- Copyright (C) 2012-2023 Zammad Foundation, https://zammad-foundation.org/ -->

<script setup lang="ts">
import useFingerprint from '@shared/composables/useFingerprint'
import { getCSRFToken } from '@shared/server/apollo/utils/csrfToken'
import type { ThirdPartyAuthProvider } from '@shared/types/authentication'

export interface Props {
  providers: ThirdPartyAuthProvider[]
}

const props = defineProps<Props>()

const csrfToken = getCSRFToken()

const { fingerprint } = useFingerprint()
</script>

<template>
  <section class="mt-4 mb-16 w-full max-w-md">
    <p class="p-3 text-center">
      {{
        $c.user_show_password_login
          ? $t('Or sign in using')
          : $t('Sign in using')
      }}
    </p>
    <div class="flex flex-wrap gap-2">
      <form
        v-for="provider of props.providers"
        :key="provider.name"
        class="min-w-1/2-2 grow"
        method="post"
        :action="`${provider.url}?fingerprint=${fingerprint}`"
      >
        <input type="hidden" name="authenticity_token" :value="csrfToken" />
        <button
          class="flex h-14 w-full cursor-pointer select-none items-center justify-center rounded-xl bg-gray-600 py-2 px-4 text-white"
        >
          <CommonIcon
            :name="`mobile-${provider.icon}`"
            size="base"
            decorative
            class="shrink-0 ltr:mr-2.5 rtl:ml-2.5"
          />
          <span
            class="overflow-hidden text-ellipsis whitespace-nowrap text-xl leading-7"
          >
            {{ $t(provider.name) }}
          </span>
        </button>
      </form>
    </div>
  </section>
</template>

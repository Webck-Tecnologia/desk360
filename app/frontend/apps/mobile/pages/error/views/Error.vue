<!-- Copyright (C) 2012-2023 Zammad Foundation, https://zammad-foundation.org/ -->

<script setup lang="ts">
import { computed } from 'vue'
import CommonBackButton from '@mobile/components/CommonBackButton/CommonBackButton.vue'
import { errorOptions } from '@mobile/router/error'
import { ErrorStatusCodes } from '@shared/types/error'

const errorImage = computed(() => {
  switch (errorOptions.value.statusCode) {
    case ErrorStatusCodes.Forbidden:
      return '/assets/error/error-mobile-403.svg'
    case ErrorStatusCodes.NotFound:
      return '/assets/error/error-mobile-404.svg'
    case ErrorStatusCodes.InternalError:
    default:
      return '/assets/error/error-mobile-500.svg'
  }
})
</script>

<template>
  <div class="flex min-h-screen flex-col px-4">
    <header class="fixed">
      <div class="grid h-16 grid-cols-3">
        <div
          class="flex cursor-pointer items-center justify-self-start text-base"
        >
          <CommonBackButton fallback="/" />
        </div>
      </div>
    </header>
    <main class="flex grow flex-col items-center justify-center">
      <h1 class="mb-9 text-8xl font-extrabold">
        {{ errorOptions.statusCode }}
      </h1>
      <img :alt="$t('Error')" :src="errorImage" />
      <h2 class="mt-9 max-w-prose text-center text-xl font-semibold">
        {{ $t(errorOptions.title) }}
      </h2>
      <p class="mt-4 min-h-[4rem] max-w-prose text-center text-gray">
        {{
          $t(errorOptions.message, ...(errorOptions.messagePlaceholder || []))
        }}
      </p>
      <p v-if="errorOptions.route" class="max-w-prose text-center text-gray">
        {{ errorOptions.route }}
      </p>
    </main>
  </div>
</template>

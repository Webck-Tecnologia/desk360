<!-- Copyright (C) 2012-2023 Zammad Foundation, https://zammad-foundation.org/ -->

<script setup lang="ts">
import { computed, provide, ref, reactive } from 'vue'
import { onBeforeRouteLeave, RouterView, useRoute, useRouter } from 'vue-router'
import { noop } from 'lodash-es'
import type {
  TicketUpdatesSubscription,
  TicketUpdatesSubscriptionVariables,
} from '@shared/graphql/types'
import { EnumFormUpdaterId } from '@shared/graphql/types'
import UserError from '@shared/errors/UserError'
import { QueryHandler } from '@shared/server/apollo/handler'
import Form from '@shared/components/Form/Form.vue'
import { useForm, FormData } from '@shared/components/Form'
import {
  NotificationTypes,
  useNotifications,
} from '@shared/components/CommonNotifications'
import { convertToGraphQLId } from '@shared/graphql/utils'
import { useApplicationStore } from '@shared/stores/application'
import { useTicketView } from '@shared/entities/ticket/composables/useTicketView'
import type { TicketInformation } from '@mobile/entities/ticket/types'
import CommonLoader from '@mobile/components/CommonLoader/CommonLoader.vue'
import useConfirmation from '@mobile/components/CommonConfirmation/composable'
import { isDialogOpened } from '@shared/composables/useDialog'
import { useErrorHandler } from '@shared/errors/useErrorHandler'
import { useTicketEdit } from '../composable/useTicketEdit'
import { TICKET_INFORMATION_SYMBOL } from '../composable/useTicketInformation'
import { useTicketQuery } from '../graphql/queries/ticket.api'
import { TicketUpdatesDocument } from '../graphql/subscriptions/ticketUpdates.api'
import { useTicketArticleReply } from '../composable/useTicketArticleReply'
import { useTicketEditForm } from '../composable/useTicketEditForm'
import { useTicketLiveUser } from '../composable/useTicketLiveUser'

interface Props {
  internalId: string
}

const props = defineProps<Props>()
const ticketId = computed(() => convertToGraphQLId('Ticket', props.internalId))

const MENTIONS_LIMIT = 5

const { createQueryErrorHandler } = useErrorHandler()

const ticketQuery = new QueryHandler(
  useTicketQuery(() => ({
    ticketId: ticketId.value,
    mentionsCount: MENTIONS_LIMIT,
  })),
  {
    errorCallback: createQueryErrorHandler({
      notFound: __(
        'Ticket with specified ID was not found. Try checking the URL for errors.',
      ),
      forbidden: __('You have insufficient rights to view this ticket.'),
    }),
  },
)

const ticketResult = ticketQuery.result()
const ticket = computed(() => ticketResult.value?.ticket)

ticketQuery.subscribeToMore<
  TicketUpdatesSubscriptionVariables,
  TicketUpdatesSubscription
>(() => ({
  document: TicketUpdatesDocument,
  variables: {
    ticketId: ticketId.value,
  },
  onError: noop,
}))

const formLocation = ref('body')
const formVisible = computed(() => formLocation.value !== 'body')

const { form, canSubmit, isDirty } = useForm()

const { initialTicketValue, isTicketFormGroupValid, editTicket } =
  useTicketEdit(ticket, form)

const canUpdateTicket = computed(() => !!ticket.value?.policy.update)

const needSpaceForSaveBanner = computed(() => {
  return canUpdateTicket.value && isDirty.value
})

const {
  articleReplyDialog,
  newTicketArticleRequested,
  newTicketArticlePresent,
  isArticleFormGroupValid,
  openArticleReplyDialog,
  closeArticleReplyDialog,
} = useTicketArticleReply(ticket, form, needSpaceForSaveBanner)

const {
  currentArticleType,
  ticketEditSchema,
  articleTypeHandler,
  articleTypeSelectHandler,
} = useTicketEditForm(ticket)

const { isTicketAgent } = useTicketView(ticket)

const { notify } = useNotifications()

const submitForm = async (formData: FormData) => {
  const updateForm = currentArticleType.value?.updateForm

  if (updateForm) {
    formData = updateForm(formData)
  }

  try {
    const result = await editTicket(formData)

    if (result?.ticketUpdate?.ticket) {
      notify({
        type: NotificationTypes.Success,
        message: __('Ticket updated successfully.'),
      })

      // Reset article form after ticket update and reseted form.
      return () => {
        newTicketArticlePresent.value = false
        closeArticleReplyDialog()
      }
    }
  } catch (errors) {
    if (errors instanceof UserError) {
      notify({
        message: errors.generalErrors[0],
        type: NotificationTypes.Error,
      })
    }
  }
}

const updateFormLocation = (newLocation: string) => {
  formLocation.value = newLocation
}

const isFormValid = computed(() => {
  if (!newTicketArticlePresent.value) return isTicketFormGroupValid.value
  return isTicketFormGroupValid.value && isArticleFormGroupValid.value
})

const showArticleReplyDialog = () => {
  return openArticleReplyDialog({ updateFormLocation })
}

const { liveUserList } = useTicketLiveUser(ticket, isTicketAgent, isDirty)

provide<TicketInformation>(TICKET_INFORMATION_SYMBOL, {
  ticketQuery,
  initialFormTicketValue: initialTicketValue,
  ticket,
  form,
  newTicketArticleRequested,
  newTicketArticlePresent,
  updateFormLocation,
  canUpdateTicket,
  showArticleReplyDialog,
  liveUserList,
})

const { waitForConfirmation } = useConfirmation()

onBeforeRouteLeave(async () => {
  if (!isDirty.value) return true

  const confirmed = await waitForConfirmation(
    __('Are you sure? You have unsaved changes that will get lost.'),
  )

  return confirmed
})

const router = useRouter()
const route = useRoute()

const openErrorsForm = () => {
  if (!isTicketFormGroupValid.value && route.name !== 'Edit') {
    return router.push(`/tickets/${ticket.value?.internalId}/information`)
  }
  if (
    newTicketArticlePresent.value &&
    !isArticleFormGroupValid.value &&
    !articleReplyDialog.isOpened.value
  ) {
    return showArticleReplyDialog()
  }
}

const application = useApplicationStore()

const smimeIntegration = computed(
  () => (application.config.smime_integration as boolean) ?? false,
)

const ticketEditSchemaData = reactive({
  formLocation,
  smimeIntegration,
  newTicketArticleRequested,
  newTicketArticlePresent,
  currentArticleType,
})

const bannerClasses = computed(() => {
  if (isDialogOpened() && !articleReplyDialog.isOpened.value) return 'hidden'

  if (route.name !== 'TicketDetailArticlesView') return null

  return articleReplyDialog.isOpened.value ? null : 'ReplyButtonPadding'
})
</script>

<template>
  <RouterView />
  <div
    class="transition-all"
    :class="{ 'pb-12': needSpaceForSaveBanner }"
  ></div>
  <!-- submit form is always present in the DOM, so we can access FormKit validity state -->
  <!-- if it's visible, it's moved to the [data-ticket-edit-form] element, which is in TicketInformationDetail -->
  <Teleport v-if="canUpdateTicket" :to="formLocation">
    <CommonLoader
      :class="formVisible ? 'visible' : 'hidden'"
      :loading="!ticket"
    >
      <Form
        v-if="ticket?.id && initialTicketValue"
        id="form-ticket-edit"
        :key="ticket.id"
        ref="form"
        :schema="ticketEditSchema"
        :flatten-form-groups="['ticket']"
        :handlers="[articleTypeHandler()]"
        :form-kit-plugins="[articleTypeSelectHandler]"
        :schema-data="ticketEditSchemaData"
        :initial-values="initialTicketValue"
        :initial-entity-object="ticket"
        :form-updater-id="EnumFormUpdaterId.FormUpdaterUpdaterTicketEdit"
        use-object-attributes
        :aria-hidden="!formVisible"
        :class="formVisible ? 'visible' : 'hidden'"
        @submit="submitForm($event as FormData)"
      />
    </CommonLoader>
  </Teleport>
  <Teleport to="body">
    <Transition
      enter-from-class="translate-y-full"
      enter-active-class="translate-y-0"
      enter-to-class="-translate-y-1/3"
      leave-from-class="-translate-y-1/3"
      leave-active-class="translate-y-full"
      leave-to-class="translate-y-full"
    >
      <div
        v-if="canUpdateTicket && isDirty"
        class="fixed bottom-2 left-2 right-2 z-10 flex rounded-lg bg-gray-300 text-white transition"
        :class="bannerClasses"
      >
        <div class="relative flex flex-1 items-center gap-2 p-1.5">
          <div class="flex-1 text-sm ltr:pl-1 rtl:pr-1">
            {{ $t('You have unsaved changes.') }}
          </div>
          <FormKit
            input-class="font-semibold text-base px-4 py-1 !text-black formkit-variant-primary:bg-yellow rounded select-none"
            wrapper-class="flex justify-center items-center"
            type="submit"
            form="form-ticket-edit"
            :disabled="!isFormValid"
          >
            {{ $t('Save') }}
          </FormKit>
          <div
            v-if="canSubmit && !isFormValid"
            role="status"
            :aria-label="$t('Validation failed')"
            class="absolute bottom-7 h-5 w-5 cursor-pointer rounded-full bg-red text-center text-xs leading-5 text-black ltr:right-2 rtl:left-2"
            @click="openErrorsForm"
          >
            <CommonIcon
              class="mx-auto h-5"
              name="mobile-close"
              size="tiny"
              decorative
            />
          </div>
        </div>
      </div>
    </Transition>
  </Teleport>
</template>

<style lang="scss" scoped>
.ReplyButtonPadding {
  --reply-size: calc(0px - theme('height.12') - var(--safe-bottom, 0px));

  transform: translateY(var(--reply-size));
}
</style>

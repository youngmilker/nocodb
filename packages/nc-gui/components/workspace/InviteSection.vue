<script lang="ts" setup>
import { onKeyStroke } from '@vueuse/core'
import { OrderedWorkspaceRoles, WorkspaceUserRoles } from 'nocodb-sdk'
import { extractSdkResponseErrorMsg, useWorkspace } from '#imports'
import { validateEmail } from '~/utils/validation'

const inviteData = reactive({
  email: '',
  roles: WorkspaceUserRoles.VIEWER,
})

const focusRef = ref<HTMLInputElement>()
const isDivFocused = ref(false)
const divRef = ref<HTMLDivElement>()

const emailValidation = reactive({
  isError: false,
  message: '',
})

const workspaceStore = useWorkspace()

const { inviteCollaborator: _inviteCollaborator } = workspaceStore
const { isInvitingCollaborators } = storeToRefs(workspaceStore)
const { workspaceRoles } = useRoles()

// all user input emails are stored here
const emailBadges = ref<Array<string>>([])

const insertOrUpdateString = (str: string) => {
  // Check if the string already exists in the array
  const index = emailBadges.value.indexOf(str)

  if (index !== -1) {
    // If the string exists, remove it
    emailBadges.value.splice(index, 1)
  }

  // Add the new string to the array
  emailBadges.value.push(str)
}

const emailInputValidation = (input: string): boolean => {
  if (!input.length) {
    emailValidation.isError = true
    emailValidation.message = 'Email Should Not Be Empty'
    return false
  }
  if (!validateEmail(input.trim())) {
    emailValidation.isError = true
    emailValidation.message = 'Invalid Email'
    return false
  }
  return true
}

watch(inviteData, (newVal) => {
  const isNewEmail = newVal.email.charAt(newVal.email.length - 1) === ',' || newVal.email.charAt(newVal.email.length - 1) === ' '
  if (isNewEmail && newVal.email.trim().length) {
    const emailToAdd = newVal.email.split(',')[0].trim() || newVal.email.split(' ')[0].trim()
    if (!validateEmail(emailToAdd)) {
      emailValidation.isError = true
      emailValidation.message = 'Invalid Email'
      return
    }
    /** 
     if email is already enterd we delete the already
     existing email and add new one
     **/
    if (emailBadges.value.includes(emailToAdd)) {
      insertOrUpdateString(emailToAdd)
      inviteData.email = ''
      return
    }
    emailBadges.value.push(emailToAdd)
    inviteData.email = ''
  }
  if (!newVal.email.length && emailValidation.isError) {
    emailValidation.isError = false
  }
})

const handleEnter = () => {
  const isEmailIsValid = emailInputValidation(inviteData.email)
  if (!isEmailIsValid) return

  inviteData.email += ' '
  emailValidation.isError = false
  emailValidation.message = ''
}

const inviteCollaborator = async () => {
  try {
    const payloadData = emailBadges.value.join(',')

    await _inviteCollaborator(payloadData, inviteData.roles)
    message.success('Invitation sent successfully')
    inviteData.email = ''
    emailBadges.value = []
  } catch (e: any) {
    message.error(await extractSdkResponseErrorMsg(e))
  }
}
// all user input emails are stored here
const emailBadges = ref<Array<string>>([])
watch(inviteData, (newVal) => {
  if (newVal.email.includes(',')) {
    emailBadges.value.push(newVal.email.split(',')[0])
    inviteData.email = ''
  }
})

// allow only lower roles to be assigned
const allowedRoles = ref<WorkspaceUserRoles[]>([])

onMounted(async () => {
  try {
    const currentRoleIndex = OrderedWorkspaceRoles.findIndex(
      (role) => workspaceRoles.value && Object.keys(workspaceRoles.value).includes(role),
    )
    if (currentRoleIndex !== -1) {
      allowedRoles.value = OrderedWorkspaceRoles.slice(currentRoleIndex + 1).filter((r) => r)
    }
  } catch (e: any) {
    message.error(await extractSdkResponseErrorMsg(e))
  }
})

const focusOnDiv = () => {
  focusRef.value?.focus()
  isDivFocused.value = true
}

// remove one email per backspace
onKeyStroke('Backspace', () => {
  if (isDivFocused.value && inviteData.email.length < 1) {
    emailBadges.value.pop()
  }
})

// when bulk email is pasted
const onPaste = (e: ClipboardEvent) => {
  const pastedText = e.clipboardData?.getData('text')
  const inputArray = pastedText?.split(',') || pastedText?.split(' ')
  // if data is pasted to a already existing text in input
  // we add existingInput + pasted data
  if (inputArray?.length === 1 && inviteData.email.length > 1) {
    inputArray[0] = inviteData.email += inputArray[0]
  }
  inputArray?.forEach((el) => {
    const isEmailIsValid = emailInputValidation(el)

    if (!isEmailIsValid) return

    /** 
     if email is already enterd we delete the already
     existing email and add new one
     **/
    if (emailBadges.value.includes(el)) {
      insertOrUpdateString(el)
      return
    }

    emailBadges.value.push(el)
    inviteData.email = ''
  })
  inviteData.email = ''
}
</script>

<template>
  <div class="my-2 pt-3 ml-2" data-testid="invite">
    <div class="text-xl mb-4">Invite</div>
    <a-form>
      <div class="flex gap-2">
        <span
          v-for="(email, index) in emailBadges"
          :key="email"
          class="p-1 border-1 border-grey rounded-2xl flex items-center justify-between"
        >
          {{ email }}
          <component :is="iconMap.close" class="ml-1.5 hover:cursor-pointer" @click="emailBadges.splice(index, 1)" />
        </span>
        <a-input
          id="email"
          v-model:value="inviteData.email"
          placeholder="Enter emails to send invitation"
          class="!max-w-130 !rounded"
          @press-enter="inviteData.email += ','"
        />

        <RolesSelector
          class="px-1"
          :role="inviteData.roles"
          :roles="allowedRoles"
          :on-role-change="(role: WorkspaceUserRoles) => (inviteData.roles = role)"
          :description="true"
        />

        <a-button
          type="primary"
          class="!rounded-md"
          :disabled="!inviteData.email?.length || isInvitingCollaborators"
          @click="inviteCollaborator"
        >
          <div class="flex flex-row items-center gap-x-2 pr-1">
            <GeneralLoader v-if="isInvitingCollaborators" class="flex" />
            <MdiPlus v-else />
            {{ isInvitingCollaborators ? 'Adding' : 'Add' }} User(s)
          </div>
        </a-button>
      </div>
      <RolesSelector
        size="md"
        class="px-1"
        :role="inviteData.roles"
        :roles="allowedRoles"
        :on-role-change="(role: WorkspaceUserRoles) => (inviteData.roles = role)"
        :description="true"
      />

      <NcButton
        type="primary"
        size="small"
        :disabled="!emailBadges.length || isInvitingCollaborators || emailValidation.isError"
        :loading="isInvitingCollaborators"
        @click="inviteCollaborator"
      >
        <MdiPlus />
        {{ isInvitingCollaborators ? 'Adding' : 'Add' }} Member(s)
      </NcButton>
    </div>
  </div>
</template>

<style scoped>
:deep(.ant-select .ant-select-selector) {
  @apply rounded;
}

.badge-text {
  @apply text-[14px] pt-1 text-center;
}

:deep(.ant-select-selection-item) {
  @apply mt-0.75;
}
</style>

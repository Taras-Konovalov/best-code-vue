<script setup lang="ts">
import { AppColors } from '~/enums/appEnum'
import { SocketParserApiEvents } from '~/enums/socketEnum'
import { GptMessageStatus } from '~/enums/utpEnum'

const { t } = useI18n()
const userStore = useUserStore()
const socketStore = useSocketStore()
const utpStore = useUTPStore()

const message = ref<string>('')

function sendMessage() {
  // передаваемое сообщение в сокет событие
  const messageBodyFromUser = {
    chatId: utpStore.getCommonChatId,
    userId: userStore.getMyBitrixId,
    message: message.value,
  }

  // передаваемое сообщение в глобалный стейт
  const messageForLocal = {
    message: message.value,
    userId: { value: userStore.getMyBitrixId! },
    date: {
      seconds: Math.floor(new Date().getTime() / 1000),
      nanos: (new Date().getTime() % 1000) * 1e6,
    },
    status: GptMessageStatus.question,
    chatId: utpStore.getCommonChatId!,
  }

  // передача сообщения в глобалный стейт
  utpStore.setParserChatGptCommonMessages(messageForLocal)

  // отображаем loader после отправки сообщения
  utpStore.setIsLoadingChatGptMessage(true)

  // отправка сообщения по сокет событию
  socketStore.socket.emit(
    SocketParserApiEvents.PARSER_API_SEND_CHATGPT_COMMON_MESSAGE, messageBodyFromUser)
  message.value = ''
}

// отправка сообщения пользователя чату GPT
async function sendMessageToGhatGpt() {
  if (!message.value.trim())
    return

  if (!utpStore.getCommonChatId) {
    utpStore.loadChatId(userStore.getMyBitrixId!).then((data) => {
      if (data)
        sendMessage()
    },
    )
  }
  else {
    sendMessage()
  }
}
</script>

<template>
  <div class="d-flex my-3" style="gap: 8px;">
    <!-- поле ввода сообщения -->
    <v-textarea
      v-model="message"
      :label="t('pages.utp.input-send-label')"
      auto-grow
      hide-details
      rows="1"
      @keyup.enter="sendMessageToGhatGpt"
    >
      <template #append-inner>
        <v-progress-circular
          v-if="utpStore.getIsLoadingChatGptMessage"
          :color="AppColors.PRIMARY"
          :width="1.5"
          indeterminate
          size="18"
        />
      </template>
    </v-textarea>
    <!-- кнопка отправки сообщения -->
    <v-btn
      style="height: 56px;"
      :color="AppColors.PRIMARY"
      :disabled="!message.trim()"
      @click="sendMessageToGhatGpt"
    >
      <v-icon class="mdi mdi-send-variant" size="x-large" color="white" />
    </v-btn>
  </div>
</template>

<style lang="scss" scoped>
</style>

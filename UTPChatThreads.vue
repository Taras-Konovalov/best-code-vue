<script setup lang="ts">
import { SocketParserApiEvents } from '~/enums/socketEnum'
import { AppColors } from '~/enums/appEnum'
import { GptMessageStatus } from '~/enums/utpEnum'

const socketStore = useSocketStore()
const utpStore = useUTPStore()
const userStore = useUserStore()
const { isDark } = useTheme()
const { getTimeAsHoursAndMinutes } = useCustomDateFormat()

// сортировка сообщений в поряке убывания, по дате
const sortMessages = computed(
  () => utpStore.getParserChatGptCommonMessages.filter(message => message.chatId === utpStore.getCommonChatId).sort(
    (a, b) =>
      new Date((b.date.seconds) * 1000).getTime() - new Date(a.date.seconds * 1000).getTime(),
  ),
)

onBeforeMount(() =>
// Перед добавлением прослушивателя событий удаляем существующий прослушиватель
  socketStore.socket.off(SocketParserApiEvents.PARSER_API_SEND_CHATGPT_COMMON_MESSAGE))

onMounted(() =>
  // слушаем сокет событие и при получении нового сообщения сохраняем его в глобальный стейт
  socketStore.socket.on(
    SocketParserApiEvents.PARSER_API_SEND_CHATGPT_COMMON_MESSAGE, (data) => {
      const message = data.message
      message.status = GptMessageStatus.answer
      utpStore.setParserChatGptCommonMessages(message)
      utpStore.setIsLoadingChatGptMessage(false)
    }))
</script>

<template>
  <div
    class="d-flex flex-column-reverse h-100 overflow-y-auto chat-threads px-4"
    style="gap: 16px"
  >
    <transition-group name="message-list">
      <div
        v-for="{ message, date, userId, status } in sortMessages"
        :key="date.seconds"
        class="d-flex flex-column"
        style="gap: 16px"
      >
        <!-- блок сообщения -->
        <div
          class="d-flex"
          :style="{
            gap: '16px', alignSelf: status === GptMessageStatus.question ? 'flex-end' : '',
          }"
        >
          <!-- блок с текстом сообщения, временем и именем пользователя -->
          <div
            class="d-flex flex-column"
            :style="{ gap: '4px', order: status === GptMessageStatus.question ? 0 : 1 }"
          >
            <!-- текст сообщения -->
            <p
              class="px-4 py-4"
              :style="{
                maxWidth: '940px',
                width: '100%',
                backgroundColor: status === GptMessageStatus.question ? (!isDark ? AppColors.ULTRA_LIGHT_GRAY : '') : '',
                borderRadius: '4px',
                border: status === GptMessageStatus.answer ? '1px solid #95AFE5' : 'none',
              }"
            >
              {{ message }}
            </p>
            <p
              :style="{
                alignSelf: status === GptMessageStatus.question ? 'flex-end' : '',
                color: AppColors.LIGHT_MOOD_FADE,
              }"
            >
              <!-- имя пользовтеля -->
              <span style="font-size: 14px;" class="mr-2">
                {{ status === GptMessageStatus.question
                  ? userStore.getFullNameUserByB24Id(userId.value)
                  : 'GPT Chat' }}
              </span>
              <!-- время отправки сообщения -->
              <span>{{ getTimeAsHoursAndMinutes(date.seconds) }}</span>
            </p>
          </div>
          <!-- аватар пользователя -->
          <v-avatar
            v-if="status === GptMessageStatus.question"
            size="40"
            style="position: relative; overflow: visible;"
          >
            <img
              :src="userStore.getUserAvatarUrlById(userId.value)"
              style="overflow: hidden; width: 40px; height: 40px; border-radius: 50%;"
            >
            <span
              :style="{
                position: 'absolute',
                right: '2px',
                bottom: '2px',
                width: '8px',
                height: '8px',
                backgroundColor: '#47B27D',
                borderRadius: '50%',
                border: '2px solid white',
              }"
            />
          </v-avatar>
          <v-avatar v-else size="40" :color="AppColors.ULTRA_LIGHT_GRAY">
            <SvgIcon
              name="ChatGPTIconSvg"
              :fill="AppColors.LIGHT_MOOD_NORMAL"
              size="20"
            />
          </v-avatar>
        </div>
      </div>
    </transition-group>
  </div>
</template>

<style lang="scss" scoped>
.chat-threads::-webkit-scrollbar {
  width: 0;
  background: transparent;
}

.message-list-item {
  display: inline-block;
}

.message-list-enter-active {
  transition: all 0.5s ease;
}

.message-list-enter-from,
.message-list-leave-to {
  opacity: 0;
  transform: translateY(130px);
}
</style>

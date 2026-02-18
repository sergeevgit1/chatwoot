<script>
import { mapGetters } from 'vuex';
import { useVuelidate } from '@vuelidate/core';
import { required } from '@vuelidate/validators';
import { useAlert } from 'dashboard/composables';
import router from '../../../../index';
import PageHeader from '../../SettingsSubPageHeader.vue';
import NextButton from 'dashboard/components-next/button/Button.vue';

export default {
  components: {
    PageHeader,
    NextButton,
  },
  setup() {
    return { v$: useVuelidate() };
  },
  data() {
    return {
      channelName: '',
      groupId: '',
      accessToken: '',
      callbackSecret: '',
      confirmationCode: '',
    };
  },
  computed: {
    ...mapGetters({
      uiFlags: 'inboxes/getUIFlags',
    }),
  },
  validations: {
    channelName: { required },
    groupId: { required },
    accessToken: { required },
    callbackSecret: { required },
    confirmationCode: { required },
  },
  methods: {
    async createChannel() {
      this.v$.$touch();
      if (this.v$.$invalid) {
        return;
      }

      try {
        const vkChannel = await this.$store.dispatch('inboxes/createChannel', {
          name: this.channelName?.trim(),
          channel: {
            type: 'vk',
            group_id: this.groupId,
            access_token: this.accessToken,
            callback_secret: this.callbackSecret,
            confirmation_code: this.confirmationCode,
          },
        });

        router.replace({
          name: 'settings_inboxes_add_agents',
          params: {
            page: 'new',
            inbox_id: vkChannel.id,
          },
        });
      } catch (error) {
        useAlert(
          error.message ||
            this.$t('INBOX_MGMT.ADD.VK_CHANNEL.API.ERROR_MESSAGE')
        );
      }
    },
  },
};
</script>

<template>
  <div class="h-full w-full p-6 col-span-6">
    <PageHeader
      :header-title="$t('INBOX_MGMT.ADD.VK_CHANNEL.TITLE')"
      :header-content="$t('INBOX_MGMT.ADD.VK_CHANNEL.DESC')"
    />

    <form
      class="flex flex-wrap flex-col mx-0"
      @submit.prevent="createChannel()"
    >
      <div class="flex-shrink-0 flex-grow-0">
        <label :class="{ error: v$.channelName.$error }">
          {{ $t('INBOX_MGMT.ADD.VK_CHANNEL.CHANNEL_NAME.LABEL') }}
          <input
            v-model="channelName"
            type="text"
            :placeholder="
              $t('INBOX_MGMT.ADD.VK_CHANNEL.CHANNEL_NAME.PLACEHOLDER')
            "
            @blur="v$.channelName.$touch"
          />
        </label>
      </div>

      <div class="flex-shrink-0 flex-grow-0">
        <label :class="{ error: v$.groupId.$error }">
          {{ $t('INBOX_MGMT.ADD.VK_CHANNEL.GROUP_ID.LABEL') }}
          <input
            v-model="groupId"
            type="text"
            :placeholder="$t('INBOX_MGMT.ADD.VK_CHANNEL.GROUP_ID.PLACEHOLDER')"
            @blur="v$.groupId.$touch"
          />
        </label>
      </div>

      <div class="flex-shrink-0 flex-grow-0">
        <label :class="{ error: v$.accessToken.$error }">
          {{ $t('INBOX_MGMT.ADD.VK_CHANNEL.ACCESS_TOKEN.LABEL') }}
          <input
            v-model="accessToken"
            type="password"
            :placeholder="
              $t('INBOX_MGMT.ADD.VK_CHANNEL.ACCESS_TOKEN.PLACEHOLDER')
            "
            @blur="v$.accessToken.$touch"
          />
        </label>
      </div>

      <div class="flex-shrink-0 flex-grow-0">
        <label :class="{ error: v$.callbackSecret.$error }">
          {{ $t('INBOX_MGMT.ADD.VK_CHANNEL.CALLBACK_SECRET.LABEL') }}
          <input
            v-model="callbackSecret"
            type="password"
            :placeholder="
              $t('INBOX_MGMT.ADD.VK_CHANNEL.CALLBACK_SECRET.PLACEHOLDER')
            "
            @blur="v$.callbackSecret.$touch"
          />
        </label>
      </div>

      <div class="flex-shrink-0 flex-grow-0">
        <label :class="{ error: v$.confirmationCode.$error }">
          {{ $t('INBOX_MGMT.ADD.VK_CHANNEL.CONFIRMATION_CODE.LABEL') }}
          <input
            v-model="confirmationCode"
            type="text"
            :placeholder="
              $t('INBOX_MGMT.ADD.VK_CHANNEL.CONFIRMATION_CODE.PLACEHOLDER')
            "
            @blur="v$.confirmationCode.$touch"
          />
        </label>
        <p class="help-text">
          {{ $t('INBOX_MGMT.ADD.VK_CHANNEL.CONFIRMATION_CODE.SUBTITLE') }}
        </p>
      </div>

      <div class="w-full mt-4">
        <NextButton
          :is-loading="uiFlags.isCreating"
          type="submit"
          solid
          blue
          :label="$t('INBOX_MGMT.ADD.VK_CHANNEL.SUBMIT_BUTTON')"
        />
      </div>
    </form>
  </div>
</template>

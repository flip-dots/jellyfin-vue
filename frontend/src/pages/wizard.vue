<template>
  <v-container class="fill-height" fluid>
    <v-row justify="center">
      <v-col cols="12" sm="12" md="12" xl="8">
        <h1 class="text-h4 mb-6 text-center">{{ heading }}</h1>
        <!-- TODO: Wait for Vuetify 3 implementation (https://github.com/vuetifyjs/vuetify/issues/13509) -->
        <!-- <v-stepper v-model="wizardStage" class="transparent-background">
          <v-stepper-header>
            <v-stepper-step
              :complete="wizardStage > 1"
              step="1"
              :editable="maxWizardStage > 0">
              {{ t('wizard.languageLocale') }}
            </v-stepper-step>

            <v-divider />

            <v-stepper-step
              :complete="wizardStage > 2"
              step="2"
              :editable="maxWizardStage > 1">
              {{ t('wizard.administratorAccount') }}
            </v-stepper-step>

            <v-divider />

            <v-stepper-step
              :complete="wizardStage > 3"
              step="3"
              :editable="maxWizardStage > 2">
              {{ t('wizard.preferredMetadataLanguage') }}
            </v-stepper-step>

            <v-divider />

            <v-stepper-step
              :complete="wizardStage > 4"
              step="4"
              :editable="maxWizardStage > 3">
              {{ t('wizard.remoteAccess') }}
            </v-stepper-step>
          </v-stepper-header>

          <v-stepper-items>
            <v-stepper-content step="1">
              <wizard-language class="pt-4" @step-complete="nextStep" />
            </v-stepper-content>

            <v-stepper-content step="2">
              <wizard-admin-account
                class="pt-4"
                @step-complete="nextStep"
                @previous-step="previousStep" />
            </v-stepper-content>

            <v-stepper-content step="3">
              <wizard-metadata
                class="pt-4"
                @step-complete="nextStep"
                @previous-step="previousStep" />
            </v-stepper-content>

            <v-stepper-content step="4">
              <wizard-remote-access
                class="pt-4"
                @step-complete="nextStep"
                @previous-step="previousStep" />
            </v-stepper-content>
          </v-stepper-items>
        </v-stepper> -->
      </v-col>
    </v-row>
  </v-container>
</template>

<route lang="yaml">
meta:
  layout: server
</route>

<script setup lang="ts">
import { computed, ref } from 'vue';
import { useRouter } from 'vue-router';
import { useI18n } from 'vue-i18n';
import { getStartupApi } from '@jellyfin/sdk/lib/utils/api/startup-api';
import { useRemote, useSnackbar } from '@/composables';

const { t } = useI18n();
const router = useRouter();
const remote = useRemote();

const wizardStage = ref(1);
const maxWizardStage = ref(1);

const heading = computed(() => {
  switch (wizardStage.value) {
    case 1: {
      return t('wizard.languageLocale');
    }
    case 2: {
      return t('wizard.administratorAccount');
    }
    case 3: {
      return t('wizard.preferredMetadataLanguage');
    }
    case 4: {
      return t('wizard.remoteAccess');
    }
  }

  return '';
});

/**
 * Completes server setup
 */
async function completeWizard(): Promise<void> {
  try {
    const api = remote.sdk.oneTimeSetup(
      remote.auth.currentServer?.PublicAddress || ''
    );

    await getStartupApi(api).completeWizard();
    // Redirect to setup complete page
    await router.replace('/server/login');
  } catch (error) {
    console.error(error);
    useSnackbar(t('wizard.completeError'), 'success');
  }
}

/**
 * Change wizard step forward
 */
async function nextStep(): Promise<void> {
  if (wizardStage.value === 4) {
    await completeWizard();
  } else {
    wizardStage.value += 1;
  }

  // This allows the return to previous steps, but not going forward past incomplete steps
  if (wizardStage.value > maxWizardStage.value) {
    maxWizardStage.value = wizardStage.value;
  }
}

/**
 * Change wizard step backwards
 */
function previousStep(): void {
  wizardStage.value -= 1;
}
</script>

<style lang="scss" scoped>
.transparent-background {
  background-color: transparent !important;
}
</style>

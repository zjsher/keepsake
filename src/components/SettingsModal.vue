<script setup lang="ts">
import { ref, watch } from 'vue'
import { ask } from '@tauri-apps/plugin-dialog'
import { Fingerprint, Smartphone } from 'lucide-vue-next'
import { useVaultStore } from '../stores/vault'
import { bioName, isAndroid } from '../lib/platform'
import { biometricForget, biometricPaths } from '../lib/biometric'

const store = useVaultStore()

const bioVaults = ref<string[]>([])

const baseName = (p: string) => p.replace(/^dropbox:/, '').split(/[/\\]/).pop() ?? p
const where = (p: string) => (p.startsWith('dropbox:') ? 'Dropbox' : p)

watch(
  () => store.settingsOpen,
  (open) => {
    if (open) bioVaults.value = biometricPaths()
  },
)

// Turning off a stored password throws away the only copy outside the user's
// head, so it asks first — this is deliberately a few taps from the home screen.
async function forget(path: string) {
  const confirmed = await ask(
    `Unlocking "${baseName(path)}" will need your master password again. The stored password is deleted from this device.`,
    {
      title: `Turn off ${bioName} for this vault?`,
      kind: 'warning',
      okLabel: `Turn off ${bioName}`,
      cancelLabel: 'Keep it',
    },
  )
  if (!confirmed) return
  await biometricForget(path)
  bioVaults.value = biometricPaths()
  store.showToast(`${bioName} unlock turned off for ${baseName(path)}`)
}
</script>

<template>
  <Transition name="fade">
    <div
      v-if="store.settingsOpen"
      class="overlay centered"
      @mousedown.self="store.settingsOpen = false"
    >
      <Transition name="pop" appear>
        <div class="modal settings">
          <div class="settings-head">
            <h3>Settings</h3>
            <kbd>esc</kbd>
          </div>

          <section v-if="isAndroid" class="group">
            <h4>Locking</h4>
            <label class="toggle-row">
              <span class="row-icon"><Smartphone :size="15" /></span>
              <span class="row-text">
                <span class="row-title">Lock when I switch apps</span>
                <span class="row-note">
                  {{
                    store.lockOnAppSwitch
                      ? 'Your vaults close as soon as Keepsake goes to the background.'
                      : 'Your vaults stay open in the background until you lock them yourself.'
                  }}
                </span>
              </span>
              <input
                type="checkbox"
                :checked="store.lockOnAppSwitch"
                @change="store.setLockOnAppSwitch(($event.target as HTMLInputElement).checked)"
              />
            </label>
          </section>

          <section class="group">
            <h4>{{ bioName }} unlock</h4>
            <p v-if="!bioVaults.length" class="empty">
              No vault has a stored password on this device. You can turn
              {{ bioName }} on while unlocking a vault.
            </p>
            <div v-for="path in bioVaults" :key="path" class="bio-vault">
              <span class="row-icon"><Fingerprint :size="15" /></span>
              <span class="row-text">
                <span class="row-title">{{ baseName(path) }}</span>
                <span class="row-note">{{ where(path) }}</span>
              </span>
              <button class="btn danger" @click="forget(path)">Turn off</button>
            </div>
          </section>

          <div class="actions">
            <button class="btn" @click="store.settingsOpen = false">Done</button>
          </div>
        </div>
      </Transition>
    </div>
  </Transition>
</template>

<style scoped>
.centered {
  align-items: center;
  padding-bottom: 8vh;
}

.settings {
  width: 440px;
  max-width: calc(100vw - 32px);
  max-height: calc(100vh - 64px);
  overflow-y: auto;
  padding: 22px 24px 18px;
}

.settings-head {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 18px;
}
h3 {
  margin: 0;
  font-size: 16px;
  font-weight: 700;
  letter-spacing: -0.2px;
}

.group + .group {
  margin-top: 20px;
}

h4 {
  margin: 0 0 10px;
  font-size: 11px;
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: 0.7px;
  color: var(--muted);
}

.toggle-row,
.bio-vault {
  display: flex;
  align-items: center;
  gap: 11px;
  padding: 11px 12px;
  border-radius: var(--radius);
  background: var(--panel);
  border: 1px solid var(--border-soft);
}
.bio-vault + .bio-vault {
  margin-top: 8px;
}
.toggle-row {
  cursor: pointer;
  user-select: none;
}

.row-icon {
  display: flex;
  color: var(--muted);
  flex-shrink: 0;
}

.row-text {
  display: flex;
  flex-direction: column;
  gap: 3px;
  min-width: 0;
  flex: 1;
}
.row-title {
  font-size: 13px;
  font-weight: 600;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}
.row-note {
  color: var(--faint);
  font-size: 11.5px;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.toggle-row input {
  accent-color: var(--accent);
  flex-shrink: 0;
  width: 17px;
  height: 17px;
  margin: 0;
}

.empty {
  margin: 0;
  color: var(--faint);
  font-size: 12.5px;
  line-height: 1.5;
}

.actions {
  display: flex;
  justify-content: flex-end;
  margin-top: 20px;
}
</style>

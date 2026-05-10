<template>
  <div>
    <h2 class="mb-4">{{ $t('plugin_powertop.title') }}</h2>

    <v-skeleton-loader v-if="loading" :loading="true" type="card" />

    <div v-else style="margin-bottom: 80px">
      <!-- Settings Card -->
      <v-card class="mb-4 pa-0">
        <v-card-title class="d-flex align-center">
          <v-icon class="mr-2">mdi-cog</v-icon>
          <span>{{ $t('plugin_powertop.settings') }}</span>
        </v-card-title>
        <v-card-text>
          <v-switch
            v-model="settings.auto_tune"
            :label="$t('plugin_powertop.auto_tune')"
            inset color="green" hide-details
          />
          <v-btn color="primary" class="mt-4" @click="saveSettings" :loading="saving">
            <v-icon start>mdi-content-save</v-icon>
            {{ $t('plugin_powertop.save_settings') }}
          </v-btn>
        </v-card-text>
      </v-card>

      <!-- Terminal Section -->
      <v-container fluid class="pt-2 pr-0 pl-0 pb-2">
        <div class="d-flex align-center ga-3 mb-4">
          <div style="width: 4px; height: 32px; border-radius: 2px; background: rgb(var(--v-theme-primary))"></div>
          <h2 class="font-weight-medium ma-0" style="font-weight: 600; line-height: 1.1">{{ $t('plugin_powertop.terminal') }}</h2>
          <v-spacer></v-spacer>
          <v-btn
            v-if="!terminalActive"
            text class="d-flex align-center" density="compact"
            @click="openTerminal"
          >
            <v-icon small class="mr-1">mdi-console</v-icon>
            {{ $t('plugin_powertop.open_terminal') }}
          </v-btn>
          <v-btn
            v-else
            text class="d-flex align-center" density="compact" color="error"
            @click="closeTerminal"
          >
            <v-icon small class="mr-1">mdi-close</v-icon>
            {{ $t('plugin_powertop.close_terminal') }}
          </v-btn>
        </div>
        <p v-if="!terminalActive" class="text-medium-emphasis ml-5">
          {{ $t('plugin_powertop.terminal_hint') }}
        </p>
        <div v-else>
          <div ref="terminalContainer" style="width: 100%; height: 420px; padding: 8px; background: #000000; border-radius: 8px"></div>
        </div>
      </v-container>
    </div>

    <!-- Overlay -->
    <v-overlay :model-value="overlay" class="align-center justify-center">
      <v-progress-circular color="onPrimary" size="64" indeterminate />
    </v-overlay>
  </div>
</template>

<script setup>
import { ref, reactive, onMounted, onUnmounted, nextTick } from 'vue';
import { Terminal } from '@xterm/xterm';
import { FitAddon } from '@xterm/addon-fit';
import { ClipboardAddon } from '@xterm/addon-clipboard';
import { io } from 'socket.io-client';
import '@xterm/xterm/css/xterm.css';

const PLUGIN_NAME = 'powertop';

const loading = ref(true);
const saving = ref(false);
const overlay = ref(false);

const settings = reactive({
  auto_tune: false
});

let resizeHandler = null;
const terminalContainer = ref(null);
const terminalActive = ref(false);
let term = null;
let fitAddon = null;
let clipboardAddon = null;
let socket = null;
let terminalSessionId = null;

const getAuthHeaders = () => ({
  Authorization: 'Bearer ' + localStorage.getItem('authToken'),
});

const fetchSettings = async () => {
  try {
    const res = await fetch(`/api/v1/mos/plugins/settings/${PLUGIN_NAME}`, {
      headers: getAuthHeaders(),
    });
    if (res.ok) {
      const data = await res.json();
      if (data.auto_tune !== undefined) {
        settings.auto_tune = data.auto_tune;
      }
    }
  } catch (e) {
    console.error('Failed to fetch settings:', e);
  }
};

const saveSettings = async () => {
  saving.value = true;
  try {
    await fetch(`/api/v1/mos/plugins/settings/${PLUGIN_NAME}`, {
      method: 'POST',
      headers: {
        ...getAuthHeaders(),
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        auto_tune: settings.auto_tune,
      }),
    });
  } catch (e) {
    console.error('Failed to save settings:', e);
  } finally {
    saving.value = false;
  }
};

const openTerminal = async () => {
  terminalActive.value = true;
  await nextTick();

  try {
    const res = await fetch('/api/v1/terminal/create', {
      method: 'POST',
      headers: {
        ...getAuthHeaders(),
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        command: '/usr/bin/powertop',
        args: [],
      }),
    });

    if (!res.ok) {
      throw new Error('Failed to create terminal session');
    }

    const session = await res.json();
    terminalSessionId = session.sessionId;

    fitAddon = new FitAddon();
    clipboardAddon = new ClipboardAddon();

    term = new Terminal({ cursorBlink: true, fontFamily: 'monospace', fontSize: 14 });
    term.loadAddon(fitAddon);
    term.loadAddon(clipboardAddon);
    term.open(terminalContainer.value);
    term.focus();
    fitAddon.fit();

    term.onSelectionChange(() => {
      if (!term.hasSelection()) return;
      const selected = term.getSelection();

      if (navigator.clipboard && window.isSecureContext) {
        navigator.clipboard.writeText(selected);
      } else {
        const textarea = document.createElement('textarea');
        textarea.value = selected;
        textarea.style.cssText = 'position:fixed;opacity:0';
        document.body.appendChild(textarea);
        textarea.focus();
        textarea.select();
        try {
          document.execCommand('copy');
        } catch (e) {}
        document.body.removeChild(textarea);
      }
    });

    socket = io('/terminal', { path: '/api/v1/socket.io/' });

    socket.on('connect', () => {
      socket.emit('join-session', {
        sessionId: terminalSessionId,
        token: localStorage.getItem('authToken'),
      });
    });

    socket.on('session-joined', () => {
      fitAddon.fit();
      socket.emit('terminal-resize', { cols: term.cols, rows: term.rows });
    });

    socket.on('terminal-output', (data) => {
      term.write(data);
    });

    term.onData((data) => {
      socket.emit('terminal-input', data);
    });

    term.onResize(({ cols, rows }) => {
      socket.emit('terminal-resize', { cols, rows });
    });

    resizeHandler = () => {
      if (fitAddon) fitAddon.fit();
    };
    window.addEventListener('resize', resizeHandler);

    socket.on('disconnect', () => {
      if (term) term.write('\r\nConnection closed.\r\n');
    });
  } catch (e) {
    console.error('Failed to open terminal:', e);
    terminalActive.value = false;
  }
};

const closeTerminal = () => {
  if (resizeHandler) {
    window.removeEventListener('resize', resizeHandler);
    resizeHandler = null;
  }
  if (socket) {
    socket.emit('leave-session');
    socket.disconnect();
    socket = null;
  }
  if (term) {
    term.dispose();
    term = null;
  }
  if (fitAddon) {
    fitAddon.dispose();
    fitAddon = null;
  }
  if (clipboardAddon) {
    clipboardAddon.dispose();
    clipboardAddon = null;
  }
  terminalSessionId = null;
  terminalActive.value = false;
};

onMounted(async () => {
  try {
    await fetchSettings();
  } catch (e) {
    console.error('Failed to initialize:', e);
  } finally {
    loading.value = false;
  }
});

onUnmounted(() => {
  closeTerminal();
});
</script>

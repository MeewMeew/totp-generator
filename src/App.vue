<script setup lang="ts">
import ClipboardJS from 'clipboard';
import { TOTP, Secret } from 'otpauth';
import { ref, onMounted, onUnmounted, computed } from 'vue'

const secret_key = ref<string>('JBSWY3DPEHPK3PXP');
const digits = ref<number>(6);
const period = ref<number>(30);
const algorithm = ref<string>('sha1');
const updatingIn = ref<number>(30);
const clipboardButton = ref<ClipboardJS | null>(null);
const token = ref<string | null>(null);
const intervalHandle = ref<number | null>(null);

const totp = computed(() => {
  return new TOTP({
    secret: Secret.fromBase32(stripSpaces(secret_key.value)),
    digits: digits.value,
    period: period.value,
    algorithm: algorithm.value,
  });
});

function getKeyFromUrl() {
  // eslint-disable-next-line no-useless-escape
  const key = document.location.hash.replace(/[#\/]+/, '');
  if (key.length > 0) secret_key.value = key;
}

function getQueryParameters() {
  const queryParams = parseURLSearch(document.location.search);
  if (queryParams.secret) secret_key.value = queryParams.secret;
  if (queryParams.digits) digits.value = queryParams.digits;
  if (queryParams.period) period.value = queryParams.period;
  if (queryParams.algorithm) algorithm.value = queryParams.algorithm;
}

function update() {
  updatingIn.value = period.value - (getCurrentSeconds() % period.value);
  token.value = truncateTo(totp.value.generate(), digits.value);
}

onMounted(() => {
  getKeyFromUrl();
  getQueryParameters()
  update();

  intervalHandle.value = setInterval(update, 1000);

  clipboardButton.value = new ClipboardJS('#clipboard-button');
});

onUnmounted(() => {
  clearInterval(intervalHandle.value!);
  clipboardButton.value?.destroy();
});

function getCurrentSeconds() {
  return Math.round(new Date().getTime() / 1000.0);
}

function stripSpaces(str: string) {
  return str.replace(/\s/g, '');
}

function truncateTo(str: string, digits: number) {
  if (str.length <= digits) {
    return str;
  }

  return str.slice(-digits);
}

function parseURLSearch(search: string) {
  const queryParams = search.substr(1).split('&').reduce(function (q: any, query) {
    const chunks = query.split('=');
    const key = chunks[0];
    const raw = decodeURIComponent(chunks[1]);
    const value = isNaN(Number(raw)) ? raw : Number(raw);
    return (q[key] = value, q);
  }, {});

  return queryParams;
}


</script>

<template>

  <section id="app" class="section" v-cloak>

    <div class="container">

      <h1 class="title">TOTP Token Generator</h1>

      <div class="field">
        <label class="label is-uppercase">Your Secret Key</label>
        <div class="control">
          <input class="input" type="text" v-model="secret_key" placeholder="The secret key (in base-32 format)">
        </div>
      </div>

      <div class="field">
        <label class="label is-uppercase">Number of Digits</label>
        <div class="control">
          <input class="input" type="text" v-model="digits" placeholder="Usually 6">
        </div>
      </div>

      <div class="field">
        <label class="label is-uppercase">Token Period (in seconds)</label>
        <div class="control">
          <input class="input" type="text" v-model="period" placeholder="Usually 30">
        </div>
      </div>

      <div class="content">
        <span class="has-text-grey is-size-7">Updating in {{ updatingIn }} seconds</span>
        <progress class="progress is-info is-small" v-bind:value="updatingIn" :max="period"></progress>
      </div>

      <div class="box">
        <p class="title is-size-1 has-text-centered" id="token">{{ token }}</p>
      </div>

      <div class="control is-clearfix">
        <a class="button is-large is-light is-pulled-right" id="clipboard-button" data-clipboard-target="#token"
          title="Copy token to clipboard">
          <img src="./assets/clippy.svg" height="36" width="36">
        </a>
      </div>

    </div>

  </section>
</template>

<style scoped>
[v-cloak] {
  display: none;
}

html,
body {
  height: 100%;
}

body>footer {
  position: sticky;
  top: 100vh;
}

.footer {
  padding: 2rem;
}

@media screen and (min-width: 1068px) {
  .container {
    max-width: 600px;
    width: 600px;
  }
}
</style>

<!--
SPDX-FileCopyrightText: syuilo and misskey-project
SPDX-License-Identifier: AGPL-3.0-only
-->

<template>
	<div>
		<MkAnimBg style="position: fixed; top: 0;" />
		<div :class="$style.formContainer">
			<div :class="$style.form">
				<MkAuthConfirm ref="authRoot" :name="session?.app.name" :permissions="_permissions"
					:manualWaiting="manualWaiting" @accept="onAccept" @deny="onDeny">
					<template #consentAdditionalInfo>
						<div v-if="session?.app.callbackUrl != null" class="_gaps_s" :class="$style.redirectRoot">
							<div>{{ i18n.ts._auth.byClickingYouWillBeRedirectedToThisUrl }}</div>
							<div class="_monospace" :class="$style.redirectUrl">{{ session.app.callbackUrl }}</div>
						</div>
					</template>
				</MkAuthConfirm>
			</div>
		</div>
	</div>
</template>

<script lang="ts" setup>
import { onMounted, ref, computed } from 'vue';
import * as Misskey from 'misskey-js';
import { misskeyApi } from '@/scripts/misskey-api.js';
import { definePageMetadata } from '@/scripts/page-metadata.js';
import { i18n } from '@/i18n.js';
import { useTemplateRef } from 'vue';
import MkAnimBg from '@/components/MkAnimBg.vue';
import MkAuthConfirm from '@/components/MkAuthConfirm.vue';

const props = defineProps<{
	token: string;
}>();

const getUrlParams = (): Record<any, any> =>
	window.location.search
		.substring(1)
		.split('&')
		.reduce((result, query) => {
			const [k, v] = query.split('=');
			result[k] = decodeURI(v);
			return result;
		}, {});

const session = ref<Misskey.entities.AuthSessionShowResponse | null>(null);
const manualWaiting = ref(true)

const _permissions = computed(() => {
	return (session.value ? session.value.app.permission.filter((p): p is typeof Misskey.permissions[number] => (Misskey.permissions as readonly string[]).includes(p)) : []);
});

const authRoot = useTemplateRef('authRoot');

async function onAccept(token?: string) {
	manualWaiting.value = true
	await misskeyApi('auth/accept', {
		token: session.value.token,
	}, token);
	manualWaiting.value = false
	const isMastodon = !!getUrlParams().mastodon;
	if (session.value && session.value.app.callbackUrl && isMastodon) {
		const redirectUri = decodeURIComponent(getUrlParams().redirect_uri);
		if (!session.value.app.callbackUrl.includes('elk.zone') && !session.value.app.callbackUrl.split('\n').includes(redirectUri)) {
			authRoot.value?.showUI('failed');
			throw new Error('Callback URI doesn\'t match registered app');
		}
		const callbackUrl = session.value.app.callbackUrl.includes('elk.zone') ? new URL(session.value.app.callbackUrl) : new URL(redirectUri);
		callbackUrl.searchParams.append('code', session.value.token);
		if (getUrlParams().state) callbackUrl.searchParams.append('state', getUrlParams().state);
		location.href = callbackUrl.toString();
	} else if (session.value && session.value.app.callbackUrl) {
		const url = new URL(session.value.app.callbackUrl);
		if (['javascript:', 'file:', 'data:', 'mailto:', 'tel:', 'vbscript:'].includes(url.protocol)) throw new Error('invalid url');
		location.href = `${session.value.app.callbackUrl}?token=${session.value.token}`;
	}
	authRoot.value?.showUI('success');
}

function onDeny() {
	authRoot.value?.showUI('denied');
}

onMounted(async () => {
	try {
		session.value = await misskeyApi('auth/session/show', {
			token: props.token,
		});
		// 既に連携していた場合
		if (session.value?.app.isAuthorized) {
			onAccept();
			return
		}
	} catch (err) {
		authRoot.value?.showUI('failed');
	}
	manualWaiting.value = false
});

definePageMetadata(() => ({
	title: i18n.ts._auth.shareAccessTitle,
	icon: 'ti ti-apps',
}));
</script>

<style lang="scss" module>
.formContainer {
	min-height: 100svh;
	padding: 32px 32px calc(env(safe-area-inset-bottom, 0px) + 32px) 32px;
	box-sizing: border-box;
	display: grid;
	place-content: center;
}

.form {
	position: relative;
	z-index: 10;
	border-radius: var(--radius);
	background-color: var(--panel);
	box-shadow: 0 8px 16px rgba(0, 0, 0, 0.1);
	overflow: clip;
	max-width: 500px;
	width: calc(100vw - 64px);
	height: min(65svh, calc(100svh - calc(env(safe-area-inset-bottom, 0px) + 64px)));
	overflow-y: auto;
}

.redirectRoot {
	padding: 16px;
	border-radius: var(--radius);
	background-color: var(--bg);
}

.redirectUrl {
	font-size: 90%;
	padding: 12px;
	border-radius: var(--radius);
	background-color: var(--panel);
	overflow-x: scroll;
}
</style>

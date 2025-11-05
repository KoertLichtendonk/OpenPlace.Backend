<template>
	<Dialog
		modal
		:draggable="false"
		:visible="isOpen"
		:style="{
			width: '40rem',
			maxWidth: '100svw',
			height: 'calc(100svh - 4rem)',
		}"
		:breakpoints="{
			'500px': 'calc(100svw - 4rem)',
			'640px': '90svw',
		}"
		@update:visible="handleClose"
	>
		<template #header>
			<div />
		</template>

		<div>
			<div class="about-logo">
				<img
					src="/img/logo-512x512.png"
					alt=""
					width="64"
					height="64"
				>
				<span>openplace</span>
			</div>

			<div class="section">
				<p>
					openplace is a free unofficial open source backend for wplace. We aim to give the freedom and flexibility for all users to be able to make their own private wplace experience for themselves, their friends, or even their community.
				</p>

				<p>
					<strong>This is an instance of openplace.</strong> It is not affiliated with the openplace project. Please contact the administrators of this instance for any questions or issues.
				</p>
			</div>

			<div class="section">
				<h3>Rules</h3>

				<p>
					To keep openplace fair and safe for everyone, you are expected to follow these rules. Violations may result in a temporary or permanent ban.
				</p>

				<p v-if="rules === null">
					Loading…
				</p>

				<ul v-else>
					<li v-if="rules.isMultiAccountAllowed">You may create more than one account.</li>
					<li v-else>Do not create more than one account. If multiple accounts are detected, all accounts will be banned.</li>

					<li v-if="rules.isOffensiveContentAllowed">You may draw content that is likely to offend others.</li>
					<li v-else>Do not draw offensive content.</li>

					<li v-if="rules.isExplicitContentAllowed">You may draw explicit or suggestive content.</li>
					<li v-else>Do not draw content that is explicit or suggestive.</li>

					<li v-if="rules.isGriefingAllowed">You may draw over existing artwork to obscure it (griefing).</li>
					<li v-else>Do not draw over existing artwork to obscure it (griefing).</li>

					<li v-if="rules.isKindGriefingAllowed">You may draw over existing artwork if you are complementing it.</li>
					<li v-else>Do not modify existing artwork, even to improve it.</li>

					<li v-if="rules.isBottingAllowed">You may use bots to automate drawing on this instance.</li>
					<li v-else>Use of bots or any other automation is not allowed on this instance.</li>

					<li>Do not create excessive traffic that may affect other people’s experience on this instance.</li>
				</ul>

				<p v-if="rules?.extraRules">{{ rules?.extraRules }}</p>
			</div>

			<p class="muted">
				openplace is developed by <a href="https://github.com/openplaceteam/openplace/contributors" target="_blank">open source contributors</a>. It uses maps hosted by <a href="https://openfreemap.org/" target="_blank">OpenFreeMap</a>.
			</p>

			<p class="muted">
				<a href="https://www.openmaptiles.org/" target="_blank">© OpenMapTiles</a> Data from <a href="https://www.openstreetmap.org/copyright" target="_blank">OpenStreetMap</a>
			</p>
		</div>
	</Dialog>
</template>

<script setup lang="ts">
import Dialog from "primevue/dialog";

const props = defineProps<{
	isOpen: boolean;
}>();

const emit = defineEmits<{
	close: [];
}>();

const handleClose = () => {
	emit("close");
};

interface Rules {
	isMultiAccountAllowed: boolean;
	isOffensiveContentAllowed: boolean;
	isExplicitContentAllowed: boolean;
	isGriefingAllowed: boolean;
	isKindGriefingAllowed: boolean;
	isBottingAllowed: boolean;
	extraRules?: string;
}

const rules = ref<Rules | null>(null);

watch(() => props.isOpen, async (newValue) => {
	if (newValue && rules.value === null) {
		const config = useRuntimeConfig();
		rules.value = await $fetch(`${config.public.backendUrl}/checkrobots`);
	}
});
</script>

<style scoped>
.about-logo {
	display: flex;
	flex-direction: row;
	align-items: center;
	justify-content: center;
	gap: 1rem;
	margin: 0 0 3.5rem 0;
}

.about-logo span {
	font: 400 2.25rem/1 "Pixelify Sans Variable", sans-serif;
}

@media (min-width: 375px) {
	.about-logo span {
		font-size: 2.5rem;
	}
}

@media (min-width: 390px) {
	.about-logo span {
		font-size: 3rem;
	}
}

@media (min-width: 500px) {
	.about-logo span {
		font-size: 3.5rem;
	}
}

.section {
	margin: 3rem 0;
}

.muted {
	color: var(--p-text-muted-color);
	font-size: 0.85rem;
}

.muted a {
	color: inherit;
}
</style>

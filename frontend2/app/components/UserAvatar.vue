<template>
	<div class="avatar">
		<OverlayBadge
			:value="user.level || undefined"
			size="small"
			class="avatar-badge"
		>
			<Avatar
				:label="user.username.charAt(0).toUpperCase()"
				:image="user.avatar || undefined"
				size="large"
				shape="circle"
				class="avatar-avatar"
			/>
		</OverlayBadge>

		<div
			v-if="user.levelProgress !== undefined"
			class="avatar-progress"
			:style="{
				'--progress': `${user.levelProgress}%`
			}"
		/>
	</div>
</template>

<script setup lang="ts">
import Avatar from "primevue/avatar";
import OverlayBadge from "primevue/overlaybadge";

defineProps<{
	user: {
		username: string;
		level: number;
		levelProgress?: number;
		avatar: string;
	};
}>();
</script>

<style scoped>
.avatar {
	position: relative;
	display: inline-block;
}

.avatar-progress {
	position: absolute;
	inset: -1px;
	z-index: 10;
	border-radius: 50%;
	background: conic-gradient(
		var(--p-primary-color) 0%,
		var(--p-primary-color) var(--progress),
		transparent var(--progress),
		transparent 100%
	);
	mask: radial-gradient(
		circle,
		transparent 0%,
		transparent calc(50% + 5px),
		black calc(50% + 5.5px),
		black 100%
	);
	pointer-events: none;
	transition: --progress 1s ease;
}

.avatar :deep(.p-overlaybadge .p-badge) {
	--p-badge-padding: 0 0.25rem;
	inset: auto 0 0 auto;
	z-index: 11;
	transform: translate(25%, 25%);
	transform-origin: 100% 100%;
	outline-width: 0;
	border-radius: 999px;
}
</style>

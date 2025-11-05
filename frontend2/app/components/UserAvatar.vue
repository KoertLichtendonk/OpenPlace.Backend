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
	</div>
</template>

<script setup lang="ts">
import Avatar from "primevue/avatar";
import OverlayBadge from "primevue/overlaybadge";

defineProps<{
	user: {
		username: string;
		level: number;
		avatar: string;
	};
}>();
</script>

<style scoped>
.avatar :deep(.p-overlaybadge .p-badge) {
	--p-badge-padding: 0 0.25rem;
	inset: auto 0 0 auto;
	transform: translate(25%, 25%);
	transform-origin: 100% 100%;
	outline-width: 0;
	border-radius: 999px;
}
</style>

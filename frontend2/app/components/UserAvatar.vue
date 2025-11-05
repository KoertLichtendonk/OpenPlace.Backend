<template>
	<button
		class="avatar-button"
		@click="$emit('click', $event)"
	>
		<OverlayBadge
			:value="user.level"
			size="small"
			class="avatar-button-badge"
		>
			<Avatar
				:label="user.username.charAt(0).toUpperCase()"
				:image="user.avatar || undefined"
				size="large"
				shape="circle"
				class="avatar-button-avatar"
			/>
			</OverlayBadge>
	</button>
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

defineEmits<{
	click: [event: Event];
}>();
</script>

<style scoped>
.avatar-button {
	position: relative;
	background: transparent;
	padding: 0;
	margin: 0;
	border: 0;
}

.avatar-button-avatar {
	box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
}

.p-overlaybadge .p-badge {
	inset-block-start: auto;
	inset-block-end: 0;
	transform: translate(25%, 25%);
	transform-origin: 0 0;
	outline-width: 0;
	border-radius: 999px;
}
</style>

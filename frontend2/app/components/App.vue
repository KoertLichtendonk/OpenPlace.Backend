<template>
	<div class="app-container">
		<Toast />

		<ClientOnly>
			<Map
				ref="mapRef"
				:initial-location="savedLocation"
				:pixels="pixels"
				:is-drawing="isPaintOpen"
				:is-satellite="isSatellite"
				:favorite-locations="userProfile?.favoriteLocations"
				:selected-pixel-coords="selectedPixelCoords"
				@map-click="handleMapClick"
				@map-right-click="handleMapRightClick"
				@draw-pixels="handleDrawPixels"
				@bearing-change="mapBearing = $event"
				@favorite-click="handleFavoriteClick"
				@save-current-location="saveCurrentLocation"
			/>
			<template #fallback>
				<div class="map-loading" />
			</template>
		</ClientOnly>

		<div class="app-overlays">
			<div class="app-overlays-zoom">
				<Button
					severity="secondary"
					raised
					rounded
					class="app-overlays--button"
					aria-label="Zoom in"
					@click="zoomIn"
				>
					<Icon name="zoom_in" />
				</Button>

				<Button
					severity="secondary"
					raised
					rounded
					class="app-overlays--button"
					aria-label="Zoom out"
					@click="zoomOut"
				>
					<Icon name="zoom_out" />
				</Button>

				<Button
					v-if="mapBearing !== 0"
					severity="secondary"
					raised
					rounded
					class="app-overlays--button"
					aria-label="Reset map rotation"
					@click="resetMapRotation"
				>
					<Icon name="compass" />
				</Button>
			</div>

			<div class="app-overlays-profile">
				<div v-if="isLoggedIn">
					<Button
						severity="secondary"
						raised
						rounded
						class="app-overlays--avatar-button"
						aria-label="Toggle user menu"
						@click="toggleUserMenu"
					>
						<UserAvatar
							:user="user"
						/>
					</Button>

					<UserMenu
						ref="userMenuRef"
						:is-open="isUserMenuOpen"
						:user="user"
						@close="isUserMenuOpen = false"
						@logout="handleLogout"
					/>
				</div>

				<Button
					v-else
					severity="primary"
					raised
					rounded
					aria-label="Log in"
					@click="handleLogin"
				>
					Log in
				</Button>

				<Button
					severity="secondary"
					raised
					rounded
					class="app-overlays--button"
					aria-label="Toggle satellite"
					@click="toggleSatellite"
				>
					<Icon :name="isSatellite ? 'map_vector' : 'map_satellite'" />
				</Button>

				<Button
					severity="secondary"
					raised
					rounded
					class="app-overlays--button"
					aria-label="Go to random pixel"
					:loading="isLoadingRandom"
					@click="goToRandom"
				>
					<Icon name="explore" />
				</Button>
			</div>

			<div
				v-if="isLoggedIn"
				class="app-overlays-paint"
			>
				<PaintButton
					:charges="currentCharges ?? 0"
					:max-charges="maxCharges ?? 0"
					:is-drawing="isPaintOpen"
					:time-until-next="formattedTime"
					@click="handlePaintButtonClick"
				/>
			</div>

			<div
				v-if="isLoggedIn"
				class="app-overlays-palette"
			>
				<ColorPalette
					:is-open="isPaintOpen"
					:selected-color="selectedColor"
					:is-eraser-mode="isEraserMode"
					:charges="currentCharges ?? 0"
					:max-charges="maxCharges ?? 0"
					:pixel-count="pixels.length"
					:time-until-next="formattedTime"
					:extra-colors-bitmap="userProfile?.extraColorsBitmap ?? 0"
					@color-select="handleColorSelect"
					@close="handleClosePaint"
					@toggle-eraser="isEraserMode = !isEraserMode"
					@submit="handleSubmitPixels"
				/>
			</div>
		</div>

		<PixelInfo
			:is-open="selectedPixelCoords !== null"
			:coords="selectedPixelCoords"
			@close="selectedPixelCoords = null"
			@report="handleReportPixel"
			@favorite-added="handleFavoriteChanged"
			@favorite-removed="handleFavoriteChanged"
		/>
	</div>
</template>

<script setup lang="ts">
import { computed, onMounted, onUnmounted, ref } from "vue";
import Toast from "primevue/toast";
import Map, { type LocationWithZoom } from "~/components/Map.vue";
import PaintButton from "~/components/PaintButton.vue";
import ColorPalette from "~/components/ColorPalette.vue";
import UserAvatar from "~/components/UserAvatar.vue";
import UserMenu from "~/components/UserMenu.vue";
import PixelInfo from "~/components/PixelInfo.vue";
import { CLOSE_ZOOM_LEVEL, getPixelId, type LngLat, lngLatToTileCoords, type TileCoords, tileCoordsToLngLat, ZOOM_LEVEL } from "~/utils/coordinates";
import { type UserProfile, useUserProfile } from "~/composables/useUserProfile";
import { useCharges } from "~/composables/useCharges";
import { usePaint } from "~/composables/usePaint";
import { useErrorToast } from "~/composables/useErrorToast";

interface Pixel {
	id: string;
	tileCoords: TileCoords;
	color: string;
}

const USER_RELOAD_INTERVAL = 15_000;
const DEFAULT_COORDS: LngLat = [151.208, -33.852];

const isPaintOpen = ref(false);
const isSatellite = ref(false);
const isUserMenuOpen = ref(false);
const isPixelInfoOpen = ref(false);
const selectedColor = ref("rgba(0,0,0,1)");
const isEraserMode = ref(false);
const pixels = ref<Pixel[]>([]);
const selectedPixelCoords = ref<TileCoords | null>(null);
const userProfile = ref<UserProfile | null>(null);
const isLoading = ref(true);
const userMenuRef = ref();
const mapRef = ref();
const mapBearing = ref(0);
const isLoadingRandom = ref(false);
const isAnimatingToRandom = ref(false);
const randomTargetCoords = ref<{ lat: number; lng: number; zoom: number } | null>(null);

let lastUserProfileFetch = Date.now();

const {
	currentCharges,
	maxCharges,
	formattedTime,
	decrementCharge,
	incrementCharge,
	initialize,
	commitPixels
} = useCharges();

const { fetchUserProfile, logout, login } = useUserProfile();
const { submitPixels } = usePaint();
const { showToast, handleError } = useErrorToast();

const isLoggedIn = computed(() => userProfile.value !== null);

const savedLocation = computed((): LocationWithZoom => {
	let location: { lng: number; lat: number; zoom: number; } | null = null;
	try {
		const locationStr = localStorage["location"];
		if (locationStr) {
			location = JSON.parse(locationStr);
		}
	} catch {
		// Ignore
	}

	return {
		center: location ? [location.lng, location.lat] : DEFAULT_COORDS,
		zoom: location?.zoom ?? CLOSE_ZOOM_LEVEL
	};
});

const saveCurrentLocation = () => {
	try {
		localStorage["location"] = JSON.stringify({
			...mapRef.value.getCenter(),
			zoom: mapRef.value.getZoom()
		});
	} catch {
		// Ignore?
	}
};

const user = computed<UserProfile | null>(() => {
	const value = userProfile.value;
	if (!value) {
		return null;
	}

	const levelProgress = Math.round((value.level - Math.floor(value.level)) * 100);

	return {
		...value,
		username: value.name,
		verified: value.verified,
		level: Math.floor(value.level),
		levelProgress,
		pixelsPainted: Math.floor(value.pixelsPainted),
		avatar: value.picture
	};
});

const updateUserProfile = async () => {
	try {
		lastUserProfileFetch = Date.now();
		const profile = await fetchUserProfile();
		userProfile.value = profile;
		if (profile) {
			initialize(
				profile.charges.count,
				profile.charges.max,
				profile.charges.cooldownMs
			);
		}
	} catch (error) {
		console.error("Failed to fetch user profile:", error);
		handleError(error);
	}
};

const handleWindowFocus = async () => {
	const now = Date.now();
	if (now - lastUserProfileFetch < USER_RELOAD_INTERVAL) {
		return;
	}

	try {
		lastUserProfileFetch = Date.now();
		const profile = await fetchUserProfile();
		userProfile.value = profile;
		if (profile) {
			initialize(
				profile.charges.count,
				profile.charges.max,
				profile.charges.cooldownMs
			);
		}
	} catch (error) {
		console.error("Failed to refresh user profile on focus:", error);
		handleError(error);
	}
};

const handleBeforeUnload = (e: BeforeUnloadEvent) => {
	if (mapRef.value?.hasUncommittedPixels?.()) {
		// Show confirm navigation prompt
		e.preventDefault();
		e.returnValue = "";
		return "";
	}
};

onMounted(async () => {
	try {
		lastUserProfileFetch = Date.now();
		userProfile.value = await fetchUserProfile();
		if (userProfile.value) {
			initialize(
				userProfile.value.charges.count,
				userProfile.value.charges.max,
				userProfile.value.charges.cooldownMs
			);
		}
	} catch (error) {
		console.error("Failed to fetch user profile:", error);
		handleError(error);
	}

	isLoading.value = false;

	// Jump to url params
	const params = new URLSearchParams(location.search);
	const latStr = params.get("lat");
	const lngStr = params.get("lng");
	const zoomStr = params.get("zoom");

	if (latStr && lngStr && mapRef.value) {
		const [lat, lng] = [Number.parseFloat(latStr), Number.parseFloat(lngStr)];
		if (!Number.isNaN(lat) && !Number.isNaN(lng)) {
			const zoom = Number.parseFloat(zoomStr ?? "") || CLOSE_ZOOM_LEVEL;
			mapRef.value.jumpToLocation(lat, lng, zoom);
		}
	}

	globalThis.addEventListener("popstate", popMapLocation);
	globalThis.addEventListener("focus", handleWindowFocus);
	globalThis.addEventListener("beforeunload", handleBeforeUnload);
	document.addEventListener("keydown", handleKeyDown);
});

onUnmounted(() => {
	globalThis.removeEventListener("popstate", popMapLocation);
	globalThis.removeEventListener("focus", handleWindowFocus);
	globalThis.removeEventListener("beforeunload", handleBeforeUnload);
	document.removeEventListener("keydown", handleKeyDown);
});

const pushMapLocation = (center?: LngLat, zoom?: number) => {
	if (!mapRef.value) {
		return;
	}

	let lng = 0;
	let lat = 0;
	if (center) {
		[lng, lat] = center;
	} else {
		const mapCenter = mapRef.value.getCenter();
		[lng, lat] = [mapCenter.lng, mapCenter.lat];
	}
	const zoomValue = zoom ?? mapRef.value.getZoom();

	const url = new URL(location.href);
	url.searchParams.set("lat", lat.toFixed(6));
	url.searchParams.set("lng", lng.toFixed(6));
	url.searchParams.set("zoom", zoomValue.toFixed(2));
	history.pushState({}, "", url);
};

const popMapLocation = () => {
	if (!mapRef.value) {
		return;
	}

	const params = new URLSearchParams(location.search);
	const latStr = params.get("lat");
	const lngStr = params.get("lng");
	const zoomStr = params.get("zoom");

	if (latStr && lngStr) {
		const [lat, lng] = [Number.parseFloat(latStr), Number.parseFloat(lngStr)];
		const zoom = Number.parseFloat(zoomStr ?? "") || CLOSE_ZOOM_LEVEL;

		if (!Number.isNaN(lat) && !Number.isNaN(lng)) {
			mapRef.value.flyToLocation(lat, lng, zoom);
		}
	}
};

const clearPendingPixels = () => {
	const pixelCount = pixels.value.length;
	pixels.value = [];
	for (let i = 0; i < pixelCount; i++) {
		incrementCharge();
	}

	mapRef.value?.cancelPaint();
};

const handleClosePaint = () => {
	clearPendingPixels();
	isPaintOpen.value = false;
};

const handleColorSelect = (color: string) => {
	selectedColor.value = color;
	isEraserMode.value = false;
};

const handleKeyDown = (event: KeyboardEvent) => {
	if (event.code === "KeyE") {
		// Toggle eraser
		isEraserMode.value = !isEraserMode.value;
	}
};

const erasePixelAtCoords = (tileCoords: TileCoords) => {
	const pixelId = getPixelId(tileCoords);
	const existingPixelIndex = pixels.value.findIndex(item => item.id === pixelId);

	if (existingPixelIndex !== -1) {
		pixels.value = pixels.value.filter((_, index) => index !== existingPixelIndex);
		incrementCharge();
		mapRef.value?.drawPixelOnCanvas(tileCoords, "rgba(0,0,0,0)");
	}
};

const drawPixelAtCoords = (tileCoords: TileCoords) => {
	if (!isPaintOpen.value) {
		return;
	}

	if (isEraserMode.value) {
		// Eraser mode
		erasePixelAtCoords(tileCoords);
	} else {
		// Paint mode
		if (currentCharges.value === null) {
			return;
		}

		if (currentCharges.value <= 0) {
			showToast({
				severity: "warn",
				summary: "Not enough charges"
			});
			return;
		}

		const pixelId = getPixelId(tileCoords);
		const existingPixelIndex = pixels.value.findIndex(item => item.id === pixelId);
		const newPixel: Pixel = {
			id: pixelId,
			tileCoords,
			color: selectedColor.value
		};

		if (existingPixelIndex === -1) {
			pixels.value.push(newPixel);
			decrementCharge();
		} else {
			pixels.value[existingPixelIndex] = newPixel;
		}
	}
};

const drawPixel = (coords: LngLat) => drawPixelAtCoords(lngLatToTileCoords(coords));

const handleDrawPixels = (coords: TileCoords[]) => {
	for (const coord of coords) {
		drawPixelAtCoords(coord);
	}
};

let lastClickTime = 0;
const DOUBLE_CLICK_THRESHOLD = 300;

const handleMapClick = (event: LngLat) => {
	if (isPaintOpen.value) {
		drawPixel(event);
	} else {
		// Figure out if this is a double click
		const now = Date.now();
		lastClickTime = now;

		if (now - lastClickTime < DOUBLE_CLICK_THRESHOLD && isPixelInfoOpen.value) {
			// Double-click to zoom - dismiss pixel info
			selectedPixelCoords.value = null;
			return;
		}

		if (mapRef.value?.getZoom() < ZOOM_LEVEL) {
			showToast({
				summary: "Zoom in to view pixels"
			});
			return;
		}

		// Show pixel info
		const tileCoords = lngLatToTileCoords(event);
		selectedPixelCoords.value = tileCoords;
		pushMapLocation(event);
	}
};

const handleMapRightClick = (event: LngLat) => {
	if (!isPaintOpen.value) {
		return;
	}

	// Right-click in paint mode to erase
	const tileCoords = lngLatToTileCoords(event);
	erasePixelAtCoords(tileCoords);
};

const toggleUserMenu = (event: Event) => {
	if (userMenuRef.value) {
		userMenuRef.value.toggle(event);
	}
};

const toggleSatellite = () => {
	isSatellite.value = !isSatellite.value;
};

const resetMapRotation = () => {
	if (mapRef.value) {
		mapRef.value.resetBearing();
	}
};

const handlePaintButtonClick = () => {
	isPaintOpen.value = true;
	pushMapLocation();
};

const handleSubmitPixels = async () => {
	if (pixels.value.length === 0) {
		return;
	}

	try {
		// TODO: Tidy up
		const paintPixels = pixels.value.map(p => ({
			tileCoords: p.tileCoords,
			color: p.color
		}));
		await submitPixels(paintPixels);

		// Commit the painted pixels to our local state
		mapRef.value?.commitCanvases();
		commitPixels();

		// Reset state
		pixels.value = [];
		isPaintOpen.value = false;
	} catch (error) {
		console.error("Failed to submit pixels:", error);
		handleError(error);
	}

	// Get new charges from server
	updateUserProfile();
};

const handleLogin = () => {
	login();
};

const handleLogout = async () => {
	await logout();
	location.reload();
};

const handleReportPixel = () => {
	showToast({
		summary: "Reporting is not yet available. Please use the old frontend to report."
	});
};

const handleFavoriteChanged = async () => {
	try {
		lastUserProfileFetch = Date.now();
		userProfile.value = await fetchUserProfile();
	} catch (error) {
		console.error("Failed to refresh user profile:", error);
		handleError(error);
	}
};

const handleFavoriteClick = (favorite: { id: number; name: string; latitude: number; longitude: number }) => {
	// Center on favorite
	const zoom = Math.max(mapRef.value.getZoom(), CLOSE_ZOOM_LEVEL);
	mapRef.value.flyToLocation(favorite.latitude, favorite.longitude, zoom);
	pushMapLocation([favorite.longitude, favorite.latitude], zoom);

	// Open pixel info
	const tileCoords = lngLatToTileCoords([favorite.longitude, favorite.latitude]);
	selectedPixelCoords.value = tileCoords;
};

const zoomIn = () => mapRef.value?.zoomIn();
const zoomOut = () => mapRef.value?.zoomOut();

const goToRandom = async () => {
	// If already animating, jump instantly to the target
	if (isAnimatingToRandom.value && randomTargetCoords.value) {
		mapRef.value.jumpToLocation(
			randomTargetCoords.value.lat,
			randomTargetCoords.value.lng,
			randomTargetCoords.value.zoom
		);

		isAnimatingToRandom.value = false;
		randomTargetCoords.value = null;
		return;
	}

	isLoadingRandom.value = true;

	try {
		const config = useRuntimeConfig();
		const data = await $fetch<{
			pixel: { x: number; y: number };
			tile: { x: number; y: number };
		}>(`${config.public.backendUrl}/s0/tile/random`, {
			credentials: "include"
		});

		const tileCoords: TileCoords = {
			tile: [data.tile.x, data.tile.y],
			pixel: [data.pixel.x, data.pixel.y]
		};
		const [lng, lat] = tileCoordsToLngLat(tileCoords);

		randomTargetCoords.value = { lat, lng, zoom: CLOSE_ZOOM_LEVEL };
		isAnimatingToRandom.value = true;
		mapRef.value?.flyToLocation(lat, lng, CLOSE_ZOOM_LEVEL);
	} catch (error) {
		console.error("Failed to get random pixel:", error);
		handleError(error);
	}

	// To support skipping the animation by clicking the button again
	setTimeout(() => {
		isAnimatingToRandom.value = false;
		randomTargetCoords.value = null;
	}, 4000);

	isLoadingRandom.value = false;
};
</script>

<style scoped>
.app-container {
	width: 100vw;
	height: 100dvh;
	overflow: hidden;
	user-select: none;
}

.map-loading {
	width: 100vw;
	height: 100dvh;
}

.app-overlays {
	display: grid;
	grid-template-areas:
		"top-left . top-right"
		". . right"
		"paint paint paint";
	grid-template-rows: auto 1fr auto;
	grid-template-columns: auto 1fr auto;
	position: absolute;
	top: 0;
	left: 0;
	width: 100vw;
	height: 100dvh;
	z-index: 10;
	pointer-events: none;
}

.app-overlays > * {
	pointer-events: auto;
}

.app-overlays-zoom {
	display: flex;
	flex-direction: column;
	align-items: flex-start;
	gap: 0.75rem;
	grid-area: top-left;
	padding: 1rem;
}

.app-overlays-random {
	display: flex;
	flex-direction: column;
	align-self: center;
	justify-self: end;
	gap: 0.75rem;
	grid-area: right;
	padding: 1rem;
}

.app-overlays-profile {
	display: flex;
	flex-direction: column;
	align-items: flex-end;
	justify-content: flex-end;
	align-self: end;
	justify-self: end;
	gap: 0.75rem;
	grid-area: top-right;
	padding: 1rem;
}

.app-overlays-paint {
	grid-area: paint;
	align-self: end;
	justify-self: center;
	position: relative;
	z-index: 11;
	padding-bottom: 1rem;
}

.app-overlays-palette {
	grid-area: paint;
	align-self: end;
	justify-self: stretch;
	position: relative;
	z-index: 12;
}

.app-overlays--button {
	font-size: 1.1rem;
	aspect-ratio: 1;
}

.app-overlays--avatar-button {
	padding: 0;
	margin: 0;
	overflow: visible;
}
</style>

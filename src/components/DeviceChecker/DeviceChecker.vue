<!--
  - @copyright Copyright (c) 2021 Marco Ambrosini <marcoambrosini@icloud.com>
  -
  - @author Marco Ambrosini <marcoambrosini@icloud.com>
  -
  - @license GNU AGPL version 3 or any later version
  -
  - This program is free software: you can redistribute it and/or modify
  - it under the terms of the GNU Affero General Public License as
  - published by the Free Software Foundation, either version 3 of the
  - License, or (at your option) any later version.
  -
  - This program is distributed in the hope that it will be useful,
  - but WITHOUT ANY WARRANTY; without even the implied warranty of
  - MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the
  - GNU Affero General Public License for more details.
  -
  - You should have received a copy of the GNU Affero General Public License
  - along with this program. If not, see <http://www.gnu.org/licenses/>.
-->

<template>
	<NcModal v-if="modal"
		class="talk-modal"
		size="small"
		:container="$store.getters.getMainContainerSelector()"
		@close="closeModal">
		<div class="device-checker">
			<h2 class="device-checker__title">
				{{ t('spreed', 'Camera and microphone check') }}
			</h2>
			<!-- Preview -->
			<div class="device-checker__preview">
				<!-- eslint-disable-next-line -->
				<video v-show="showVideo"
					ref="video"
					class="preview__video"
					disable-picture-in-picture="true"
					tabindex="-1" />
				<div v-show="!showVideo"
					class="preview__novideo">
					<VideoBackground :display-name="displayName"
						:user="userId" />
					<NcAvatar v-if="userId"
						:size="128"
						:disable-menu="true"
						:disable-tooltip="true"
						:show-user-status="false"
						:user="userId"
						:display-name="displayName" />
					<div v-if="!userId"
						class="avatar avatar-128px guest">
						{{ firstLetterOfGuestName }}
					</div>
				</div>
			</div>

			<!--
				Toggle audio and video on and off before starting or joining
				a call.
			-->
			<div class="device-checker__call-preferences">
				<!-- Audio toggle -->
				<NcButton v-tooltip="audioButtonTooltip"
					type="tertiary"
					:aria-label="audioButtonTooltip"
					:disabled="!audioPreviewAvailable"
					@click="toggleAudio">
					<template #icon>
						<Microphone v-if="audioOn"
							:size="20" />
						<MicrophoneOff v-else
							:size="20" />
					</template>
				</NcButton>
				<VolumeIndicator class="indicator"
					:audio-preview-available="audioPreviewAvailable"
					:current-volume="currentVolume"
					:volume-threshold="volumeThreshold"
					:disabled="!audioOn" />

				<!-- Video toggle -->
				<NcButton v-tooltip="videoButtonTooltip"
					type="tertiary"
					:aria-label="videoButtonTooltip"
					:disabled="!videoPreviewAvailable"
					@click="toggleVideo">
					<template #icon>
						<VideoIcon v-if="videoOn"
							:size="20" />
						<VideoOff v-else
							:size="20" />
					</template>
				</NcButton>

				<!-- Blur toggle -->
				<NcButton v-if="videoPreviewAvailable && blurPreviewAvailable"
					v-tooltip="blurButtonTooltip"
					type="tertiary"
					:aria-label="blurButtonTooltip"
					:disabled="!blurPreviewAvailable"
					@click="toggleBlur">
					<template #icon>
						<Blur v-if="blurOn"
							:size="20" />
						<BlurOff v-else
							:size="20" />
					</template>
				</NcButton>
			</div>

			<!-- Device selection -->
			<div class="device-checker__device-selection">
				<NcButton v-if="!showDeviceSelection"
					type="tertiary"
					class="select-devices"
					@click="showDeviceSelection = true">
					<template #icon>
						<Cog :size="20" />
					</template>
					{{ t('spreed', 'Choose devices') }}
				</NcButton>
				<template v-if="showDeviceSelection">
					<MediaDevicesSelector kind="audioinput"
						:devices="devices"
						:device-id="audioInputId"
						@update:deviceId="audioInputId = $event" />
					<MediaDevicesSelector kind="videoinput"
						:devices="devices"
						:device-id="videoInputId"
						@update:deviceId="videoInputId = $event" />
				</template>
			</div>
			<NcCheckboxRadioSwitch :checked.sync="showDeviceChecker"
				class="checkbox">
				{{ t('spreed', 'Always show this dialog before joining a call in this conversation.') }}
			</NcCheckboxRadioSwitch>

			<div class="device-checker__call-buttons">
				<!-- Silent call -->
				<NcActions v-if="showSilentCallOption" :force-menu="true">
					<template v-if="!silentCall">
						<NcActionButton :close-after-click="true"
							icon="icon-upload"
							:title="t('spreed', 'Call without notification')"
							@click="silentCall= true">
							{{ t('spreed', 'The conversation participants will not be notified about this call') }}
							<BellOff slot="icon"
								:size="16" />
						</NcActionButton>
					</template>
					<template v-else>
						<NcActionButton :close-after-click="true"
							icon="icon-upload"
							:title="t('spreed', 'Normal call')"
							@click="silentCall= false">
							{{ t('spreed', 'The conversation participants will be notified about this call') }}
							<Bell slot="icon"
								:size="16" />
						</NcActionButton>
					</template>
				</NcActions>
				<!-- Join call -->
				<CallButton class="call-button"
					:force-join-call="true"
					:silent-call="silentCall" />
			</div>
		</div>
	</NcModal>
</template>

<script>
import NcModal from '@nextcloud/vue/dist/Components/NcModal.js'
import Tooltip from '@nextcloud/vue/dist/Directives/Tooltip.js'
import { devices } from '../../mixins/devices.js'
import MediaDevicesSelector from '../MediaDevicesSelector.vue'
import VideoBackground from '../CallView/shared/VideoBackground.vue'
import NcAvatar from '@nextcloud/vue/dist/Components/NcAvatar.js'
import Cog from 'vue-material-design-icons/Cog.vue'
import Microphone from 'vue-material-design-icons/Microphone.vue'
import MicrophoneOff from 'vue-material-design-icons/MicrophoneOff.vue'
import VideoIcon from 'vue-material-design-icons/Video.vue'
import VideoOff from 'vue-material-design-icons/VideoOff.vue'
import Blur from 'vue-material-design-icons/Blur.vue'
import BlurOff from 'vue-material-design-icons/BlurOff.vue'
import BellOff from 'vue-material-design-icons/BellOff.vue'
import Bell from 'vue-material-design-icons/Bell.vue'
import { localMediaModel } from '../../utils/webrtc/index.js'
import CallButton from '../TopBar/CallButton.vue'
import { subscribe, unsubscribe } from '@nextcloud/event-bus'
import NcCheckboxRadioSwitch from '@nextcloud/vue/dist/Components/NcCheckboxRadioSwitch.js'
import BrowserStorage from '../../services/BrowserStorage.js'
import VolumeIndicator from '../VolumeIndicator/VolumeIndicator.vue'
import NcButton from '@nextcloud/vue/dist/Components/NcButton.js'
import NcActions from '@nextcloud/vue/dist/Components/NcActions.js'
import NcActionButton from '@nextcloud/vue/dist/Components/NcActionButton.js'
import isInLobby from '../../mixins/isInLobby.js'

export default {
	name: 'DeviceChecker',

	directives: {
		Tooltip,
	},

	components: {
		NcModal,
		MediaDevicesSelector,
		VideoBackground,
		NcAvatar,
		Cog,
		Microphone,
		MicrophoneOff,
		VideoIcon,
		VideoOff,
		Blur,
		BlurOff,
		CallButton,
		NcCheckboxRadioSwitch,
		VolumeIndicator,
		NcButton,
		NcActionButton,
		NcActions,
		BellOff,
		Bell,
	},

	mixins: [devices, isInLobby],

	data() {
		return {
			model: localMediaModel,
			modal: false,
			showDeviceSelection: false,
			audioOn: undefined,
			videoOn: undefined,
			blurOn: undefined,
			showDeviceChecker: true,
			silentCall: false,

		}
	},

	computed: {
		displayName() {
			return this.$store.getters.getDisplayName()
		},

		guestName() {
			return this.$store.getters.getGuestName(
				this.$store.getters.getToken(),
				this.$store.getters.getActorId(),
			)
		},

		firstLetterOfGuestName() {
			const customName = this.guestName !== t('spreed', 'Guest') ? this.guestName : '?'
			return customName.charAt(0)
		},

		userId() {
			return this.$store.getters.getUserId()
		},

		token() {
			return this.$store.getters.getToken()
		},

		showVideo() {
			return this.videoPreviewAvailable && this.videoOn
		},

		blurPreviewAvailable() {
			return this.virtualBackground.isAvailable()
		},

		audioButtonTooltip() {
			if (!this.audioPreviewAvailable) {
				return t('spreed', 'No audio')
			}
			return this.audioOn ? t('spreed', 'Mute audio') : t('spreed', 'Unmute audio')
		},

		videoButtonTooltip() {
			if (!this.videoPreviewAvailable) {
				return t('spreed', 'No camera')
			}
			return this.videoOn ? t('spreed', 'Disable video') : t('spreed', 'Enable video')
		},

		blurButtonTooltip() {
			return this.blurOn ? t('spreed', 'Disable background blur') : t('spreed', 'Blur background')
		},

		conversation() {
			return this.$store.getters.conversation(this.token) || this.$store.getters.dummyConversation
		},

		hasCall() {
			return this.conversation.hasCall || this.conversation.hasCallOverwrittenByChat
		},

		showSilentCallOption() {
			return !(this.hasCall && !this.isInLobby)
		},
	},

	watch: {
		modal(newValue) {
			if (newValue) {
				this.audioOn = !localStorage.getItem('audioDisabled_' + this.token)
				this.videoOn = !localStorage.getItem('videoDisabled_' + this.token)
				this.blurOn = !!localStorage.getItem('virtualBackgroundEnabled_' + this.token)

				this.initializeDevicesMixin()
			} else {
				this.stopDevicesMixin()
			}
		},

		showDeviceChecker(newValue) {
			if (newValue) {
				BrowserStorage.setItem('showDeviceChecker' + this.token, 'true')
			} else {
				BrowserStorage.setItem('showDeviceChecker' + this.token, 'false')
			}
		},

		audioInputId(audioInputId) {
			if (this.showDeviceSelection && audioInputId && !this.audioOn) {
				this.toggleAudio()
			}
		},

		videoInputId(videoInputId) {
			if (this.showDeviceSelection && videoInputId && !this.videoOn) {
				this.toggleVideo()
			}
		},

		blurOn() {
			this.virtualBackground.setEnabled(this.blurOn)
		},
	},

	mounted() {
		subscribe('talk:device-checker:show', this.showModal)
		subscribe('talk:device-checker:hide', this.closeModal)
	},

	beforeDestroy() {
		unsubscribe('talk:device-checker:show', this.showModal)
		unsubscribe('talk:device-checker:hide', this.closeModal)
	},

	methods: {
		showModal() {
			this.modal = true
		},

		closeModal() {
			this.modal = false
			this.showDeviceSelection = false
		},

		toggleAudio() {
			if (!this.audioOn) {
				localStorage.removeItem('audioDisabled_' + this.token)
				this.audioOn = true
			} else {
				localStorage.setItem('audioDisabled_' + this.token, 'true')
				this.audioOn = false
			}
		},

		toggleVideo() {
			if (!this.videoOn) {
				localStorage.removeItem('videoDisabled_' + this.token)
				this.videoOn = true
			} else {
				localStorage.setItem('videoDisabled_' + this.token, 'true')
				this.videoOn = false
			}
		},

		toggleBlur() {
			if (!this.blurOn) {
				localStorage.setItem('virtualBackgroundEnabled_' + this.token, 'true')
				this.blurOn = true
			} else {
				localStorage.removeItem('virtualBackgroundEnabled_' + this.token)
				this.blurOn = false
			}
		},
	},
}
</script>

<style lang="scss" scoped>
@import '../../assets/variables';
@import '../../assets/avatar';
@include avatar-mixin(64px);
@include avatar-mixin(128px);

.device-checker {
	padding: 20px;
	background-color: var(--color-main-background);
	overflow-y: auto;
	overflow-x: hidden;
	margin: auto;
	&__title {
		text-align: center;
	}
	&__preview {
		position: relative;
		margin: 0 auto 12px auto;
		display: flex;
		align-items: center;
		justify-content: center;
		overflow: hidden;
		border-radius: 12px;
		height: 263px;
		background-color: var(--color-loading-dark);
	}

	&__device-selection {
		width: 100%;
	}

	&__call-preferences {
		height: $clickable-area;
		display: flex;
		justify-content: center;
		align-items: center;
	}

	&__call-buttons {
		display: flex;
		align-items: center;
		justify-content: center;
		gap: 4px;
	}
}

.preview {
	&__video {
		max-width: 100%;
		object-fit: contain;
		max-height: 100%;
	}

	&__novideo {
		display: flex;
		align-items: center;
		justify-content: center;
		overflow: hidden;
		width: 100%;
	}
}

.select-devices {
	display: flex;
	align-items: center;
	justify-content: center;
	margin: auto;
}

.call-button {
	display: flex;
	justify-content: center;
	align-items: center;
}

.checkbox {
	display: flex;
	justify-content: center;
	margin: 14px;
}

.indicator {
	margin-left: -8px;
}

::v-deep .modal-container {
	display: flex !important;
}
</style>

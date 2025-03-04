<!--
 - @copyright Copyright (c) 2019 Joas Schilling <coding@schilljs.com>
 -
 - @author Joas Schilling <coding@schilljs.com>
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
 - MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
 - GNU Affero General Public License for more details.
 -
 - You should have received a copy of the GNU Affero General Public License
 - along with this program. If not, see <http://www.gnu.org/licenses/>.
 -
 -->

<template>
	<div id="general_settings" class="videocalls section">
		<h2>{{ t('spreed', 'General settings') }}</h2>

		<h3>{{ t('spreed', 'Default notification settings') }}</h3>

		<div class="paragraph">
			<label for="default_group_notification">{{ t('spreed', 'Default group notification') }}</label>
			<NcMultiselect id="default_group_notification"
				v-model="defaultGroupNotification"
				name="default_group_notification"
				:options="defaultGroupNotificationOptions"
				:placeholder="t('spreed', 'Default group notification for new groups')"
				label="label"
				track-by="value"
				:disabled="loading || loadingDefaultGroupNotification"
				@input="saveDefaultGroupNotification" />
		</div>

		<h3>{{ t('spreed', 'Integration into other apps') }}</h3>

		<NcCheckboxRadioSwitch :checked="isConversationsFilesChecked"
			:disabled="loading || loadingConversationsFiles"
			@update:checked="saveConversationsFiles">
			{{ t('spreed', 'Allow conversations on files') }}
		</NcCheckboxRadioSwitch>

		<NcCheckboxRadioSwitch :checked="isConversationsFilesPublicSharesChecked"
			:disabled="loading || loadingConversationsFiles || !isConversationsFilesChecked"
			@update:checked="saveConversationsFilesPublicShares">
			{{ t('spreed', 'Allow conversations on public shares for files') }}
		</NcCheckboxRadioSwitch>
	</div>
</template>

<script>
import NcCheckboxRadioSwitch from '@nextcloud/vue/dist/Components/NcCheckboxRadioSwitch.js'
import NcMultiselect from '@nextcloud/vue/dist/Components/NcMultiselect.js'
import { loadState } from '@nextcloud/initial-state'

const defaultGroupNotificationOptions = [
	{ value: 1, label: t('spreed', 'All messages') },
	{ value: 2, label: t('spreed', '@-mentions only') },
	{ value: 3, label: t('spreed', 'Off') },
]
export default {
	name: 'GeneralSettings',

	components: {
		NcCheckboxRadioSwitch,
		NcMultiselect,
	},

	data() {
		return {
			loading: true,
			loadingConversationsFiles: false,
			loadingDefaultGroupNotification: false,

			defaultGroupNotificationOptions,
			defaultGroupNotification: defaultGroupNotificationOptions[1],

			conversationsFiles: parseInt(loadState('spreed', 'conversations_files')) === 1,
			conversationsFilesPublicShares: parseInt(loadState('spreed', 'conversations_files_public_shares')) === 1,
		}
	},

	computed: {
		isConversationsFilesChecked() {
			return this.conversationsFiles
		},
		isConversationsFilesPublicSharesChecked() {
			return this.conversationsFilesPublicShares
		},
	},

	mounted() {
		this.loading = true
		this.defaultGroupNotification = defaultGroupNotificationOptions[parseInt(loadState('spreed', 'default_group_notification')) - 1]
		this.loading = false
	},

	methods: {
		saveDefaultGroupNotification() {
			this.loadingDefaultGroupNotification = true

			OCP.AppConfig.setValue('spreed', 'default_group_notification', this.defaultGroupNotification.value, {
				success: function() {
					this.loadingDefaultGroupNotification = false
				}.bind(this),
			})
		},
		saveConversationsFiles(checked) {
			this.loadingConversationsFiles = true
			this.conversationsFiles = checked

			OCP.AppConfig.setValue('spreed', 'conversations_files', this.conversationsFiles ? '1' : '0', {
				success: function() {
					if (!this.conversationsFiles) {
						// When the file integration is disabled, the share integration is also disabled
						OCP.AppConfig.setValue('spreed', 'conversations_files_public_shares', '0', {
							success: function() {
								this.conversationsFilesPublicShares = false
								this.loadingConversationsFiles = false
							}.bind(this),
						})
					} else {
						this.loadingConversationsFiles = false
					}
				}.bind(this),
			})
		},
		saveConversationsFilesPublicShares(checked) {
			this.loadingConversationsFiles = true
			this.conversationsFilesPublicShares = checked

			OCP.AppConfig.setValue('spreed', 'conversations_files_public_shares', this.conversationsFilesPublicShares ? '1' : '0', {
				success: function() {
					this.loadingConversationsFiles = false
				}.bind(this),
			})
		},
	},
}
</script>
<style scoped lang="scss">

h3 {
	margin-top: 24px;
}

div.paragraph {
	display: flex;
	align-items: center;

	label {
		display: block;
		margin-right: 10px;
	}
}

.multiselect {
	flex-grow: 1;
	max-width: 300px;
}
</style>

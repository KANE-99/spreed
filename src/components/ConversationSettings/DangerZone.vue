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
	<div>
		<NcNoteCard type="warning">
			<p>{{ t('spreed', 'Be careful, these actions cannot be undone.') }}</p>
		</NcNoteCard>
		<div class="danger-zone">
			<div v-if="canLeaveConversation" class="app-settings-subsection">
				<h4 class="app-settings-section__subtitle">
					{{ t('spreed', 'Leave conversation') }}
				</h4>
				<p class="danger-zone__hint">
					{{ t('spreed', 'Once a conversation is left, to rejoin a closed conversation, an invite is needed. An open conversation can be rejoined at any time.') }}
				</p>
				<NcButton type="warning" @click.prevent.exact="leaveConversation">
					{{ t('spreed', 'Leave conversation') }}
				</NcButton>
			</div>
			<div v-if="canDeleteConversation" class="app-settings-subsection">
				<h4 class="app-settings-section__subtitle">
					{{ t('spreed', 'Delete conversation') }}
				</h4>
				<p class="danger-zone__hint">
					{{ t('spreed', 'Permanently delete this conversation.') }}
				</p>
				<NcButton type="error"
					@click.prevent.exact="deleteConversation">
					{{ t('spreed', 'Delete conversation') }}
				</NcButton>
			</div>
			<div v-if="canDeleteConversation" class="app-settings-subsection">
				<h4 class="app-settings-section__subtitle">
					{{ t('spreed', 'Delete chat messages') }}
				</h4>
				<p class="danger-zone__hint">
					{{ t('spreed', 'Permanently delete all the messages in this conversation.') }}
				</p>
				<NcButton type="error"
					@click.prevent.exact="clearChatHistory">
					{{ t('spreed', 'Delete chat messages') }}
				</NcButton>
			</div>
			<div />
		</div>
	</div>
</template>

<script>
import { showError } from '@nextcloud/dialogs'
import { emit } from '@nextcloud/event-bus'
import NcButton from '@nextcloud/vue/dist/Components/NcButton.js'
import NcNoteCard from '@nextcloud/vue/dist/Components/NcNoteCard.js'

export default {
	name: 'DangerZone',
	components: {
		NcButton,
		NcNoteCard,
	},
	props: {
		conversation: {
			type: Object,
			required: true,
		},

		canLeaveConversation: {
			type: Boolean,
			required: true,
		},

		canDeleteConversation: {
			type: Boolean,
			required: true,
		},
	},

	computed: {
		token() {
			return this.conversation.token
		},
	},

	methods: {

		hideConversationSettings() {
			emit('hide-conversation-settings')
		},
		/**
		 * Deletes the current user from the conversation.
		 */
		async leaveConversation() {
			try {
				await this.$store.dispatch('removeCurrentUserFromConversation', { token: this.token })
				this.hideConversationSettings()
			} catch (error) {
				if (error?.response?.status === 400) {
					showError(t('spreed', 'You need to promote a new moderator before you can leave the conversation.'))
				} else {
					console.error(`error while removing yourself from conversation ${error}`)
				}
			}
		},

		/**
		 * Deletes the conversation.
		 */
		async deleteConversation() {
			OC.dialogs.confirm(
				t('spreed', 'Do you really want to delete "{displayName}"?', this.conversation, undefined, {
					escape: false,
					sanitize: false,
				}),
				t('spreed', 'Delete conversation'),
				async function(decision) {
					if (!decision) {
						return
					}

					if (this.token === this.$store.getters.getToken()) {
						this.$router.push({ name: 'root' })
						this.$store.dispatch('updateToken', '')
					}

					try {
						await this.$store.dispatch('deleteConversationFromServer', { token: this.token })
						// Close the settings
						this.hideConversationSettings()
					} catch (error) {
						console.debug(`error while deleting conversation ${error}`)
						showError(t('spreed', 'Error while deleting conversation'))
					}
				}.bind(this)
			)
		},

		/**
		 * Clears the chat history
		 */
		async clearChatHistory() {
			OC.dialogs.confirm(
				t('spreed', 'Do you really want to delete all messages in "{displayName}"?', this.conversation, undefined, {
					escape: false,
					sanitize: false,
				}),
				t('spreed', 'Delete all chat messages'),
				async function(decision) {
					if (!decision) {
						return
					}

					try {
						await this.$store.dispatch('clearConversationHistory', { token: this.token })
						// Close the settings
						this.hideConversationSettings()
					} catch (error) {
						console.debug(`error while clearing chat history ${error}`)
						showError(t('spreed', 'Error while clearing chat history'))
					}
				}.bind(this)
			)
		},
	},
}
</script>

<style lang="scss" scoped>
h4 {
	font-weight: bold;
}

.danger-zone {
	&__hint {
		color: var(--color-text-maxcontrast);
	}
}

</style>

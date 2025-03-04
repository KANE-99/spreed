<!--
  - @copyright Copyright (c) 2022 Marco Ambrosini <marcoambrosini@pm.me>
  -
  - @author Marco Ambrosini <marcoambrosini@pm.me>
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
	<div class="wrapper">
		<!-- Poll card -->
		<a v-if="!showAsButton"
			v-observe-visibility="getPollData"
			:aria-label="t('spreed', 'Poll')"
			class="poll"
			role="button"
			@click="openPoll">
			<div class="poll__header">
				<PollIcon :size="20" />
				<p>
					{{ pollName }}
				</p>
			</div>
			<div class="poll__footer">
				{{ pollFooterText }}
			</div>

		</a>

		<!-- Poll results button in system message -->
		<div v-else class="poll-closed">
			<NcButton type="secondary" @click="openPoll">
				{{ t('spreed', 'See results') }}
			</NcButton>
		</div>

		<!-- voting and results dialog -->
		<NcModal v-if="vote !== undefined && showModal"
			size="small"
			:container="$store.getters.getMainContainerSelector()"
			@close="dismissModal">
			<div class="poll__modal">
				<!-- First screen, displayed while voting-->
				<template v-if="modalPage === 'voting'">
					<!-- Title -->
					<div class="poll__header">
						<PollIcon :size="20" />
						<h2 class="poll__modal-title">
							{{ pollName }}
						</h2>
					</div>

					<!-- options -->
					<div class="poll__modal-options">
						<template v-if="checkboxRadioSwitchType === 'radio'">
							<NcCheckboxRadioSwitch v-for="(option, index) in options"
								:key="'radio' + index"
								:checked.sync="vote"
								class="poll__option"
								:value="index.toString()"
								:type="checkboxRadioSwitchType"
								name="answerType">
								{{ option }}
							</NcCheckboxRadioSwitch>
						</template>
						<template v-else>
							<NcCheckboxRadioSwitch v-for="(option, index) in options"
								:key="'checkbox' + index"
								:checked.sync="vote"
								:value="index.toString()"
								:type="checkboxRadioSwitchType"
								name="answerType">
								{{ option }}
							</NcCheckboxRadioSwitch>
						</template>
					</div>

					<div class="poll__modal-actions">
						<!-- Submit vote button-->
						<NcButton type="primary" :disabled="!canSubmitVote" @click="submitVote">
							{{ t('spreed', 'Submit vote') }}
						</NcButton>
						<!-- End poll button-->
						<NcButton v-if="canEndPoll"
							type="error"
							@click="endPoll">
							{{ t('spreed', 'End poll') }}
						</NcButton>
					</div>
				</template>

				<!-- Results page -->
				<template v-if="modalPage === 'results'">
					<!-- Title -->
					<div class="poll__header">
						<PollIcon :size="20" />
						<h2 class="poll__modal-title">
							{{ pollName }}
						</h2>
					</div>
					<div class="poll__summary">
						<template v-if="currentUserIsPollCreator || currentUserIsModerator || pollIsPublic">
							{{ n('spreed', 'Poll results • %n vote', 'Poll results • %n votes', votersNumber) }}
						</template>
						<template v-else-if="selfHasVoted">
							{{ t('spreed', 'Poll ・ You voted already') }}
						</template>
					</div>
					<div class="results__options">
						<div v-for="(option, index) in options"
							:key="index"
							class="results__option">
							<div class="results__option-title">
								<p>
									{{ option }}
								</p>
								<p class="percentage">
									{{ getVotePercentage(index) + '%' }}
								</p>
							</div>
							<div v-if="getFilteredDetails(index).length > 0 || selfHasVotedOption(index)"
								class="results__option__details">
								<PollVotersDetails v-if="details"
									:details="getFilteredDetails(index)" />
								<p v-if="selfHasVotedOption(index)" class="results__option-subtitle">
									{{ t('spreed','You voted for this option') }}
								</p>
							</div>
							<NcProgressBar class="results__option-progress"
								:value="getVotePercentage(index)"
								size="medium" />
						</div>
					</div>
					<div v-if="pollIsOpen"
						class="poll__modal-actions">
						<!-- Vote again-->
						<NcButton type="secondary"
							@click="modalPage = 'voting'">
							{{ t('spreed', 'Change your vote') }}
						</NcButton>
						<!-- End poll button-->
						<NcButton v-if="canEndPoll"
							type="error"
							@click="endPoll">
							{{ t('spreed', 'End poll') }}
						</NcButton>
					</div>
				</template>
			</div>
		</NcModal>
	</div>
</template>

<script>

import NcCheckboxRadioSwitch from '@nextcloud/vue/dist/Components/NcCheckboxRadioSwitch.js'
import NcModal from '@nextcloud/vue/dist/Components/NcModal.js'
import NcButton from '@nextcloud/vue/dist/Components/NcButton.js'
import PollIcon from 'vue-material-design-icons/Poll.vue'
import NcProgressBar from '@nextcloud/vue/dist/Components/NcProgressBar.js'
import { PARTICIPANT } from '../../../../../constants.js'
import PollVotersDetails from './PollVotersDetails.vue'

export default {

	name: 'Poll',

	components: {
		NcCheckboxRadioSwitch,
		NcModal,
		NcButton,
		PollIcon,
		NcProgressBar,
		PollVotersDetails,
	},

	props: {
		pollName: {
			type: String,
			required: true,
		},

		id: {
			type: String,
			required: true,
		},

		token: {
			type: String,
			required: true,
		},

		showAsButton: {
			type: Boolean,
			default: false,
		},
	},

	data() {
		return {
			vote: undefined,
			showModal: false,
			modalPage: '',
		}
	},

	computed: {

		poll() {
			return this.$store.getters.getPoll(this.token, this.id)
		},

		pollLoaded() {
			return !!this.poll
		},

		votersNumber() {
			return this.pollLoaded ? this.poll.numVoters : undefined
		},

		question() {
			return this.pollLoaded ? this.poll.question : undefined
		},

		options() {
			return this.pollLoaded ? this.poll.options : undefined
		},

		pollVotes() {
			return this.pollLoaded ? this.poll.votes : undefined
		},

		selfHasVoted() {
			if (this.pollLoaded) {
				if (typeof this.votedSelf === 'object') {
					return this.votedSelf.length > 0
				} else {
					return !!this.votedSelf
				}
			} else {
				return undefined
			}
		},

		// The actual vote of the user as returned from the server
		votedSelf() {
			return this.pollLoaded ? this.poll.votedSelf : undefined
		},

		resultMode() {
			return this.pollLoaded ? this.poll.resultMode : undefined
		},

		pollIsPublic() {
			return this.resultMode === 0
		},

		status() {
			return this.pollLoaded ? this.poll.status : undefined
		},

		pollIsOpen() {
			return this.status === 0
		},

		pollIsClosed() {
			return this.status === 1
		},

		details() {
			if (!this.pollLoaded || this.pollIsOpen) {
				return undefined
			} else {
				return this.poll.details
			}
		},

		checkboxRadioSwitchType() {
			if (this.pollLoaded) {
				return this.poll.maxVotes === 0 ? 'checkbox' : 'radio'
			} else {
				return undefined
			}
		},

		canSubmitVote() {
			if (typeof this.vote === 'object') {
				return this.vote.length > 0
			} else {
				return this.vote !== undefined && this.vote !== ''
			}
		},

		getVotePercentage() {
			return (index) => {
				if (this.pollVotes[`option-${index}`] === undefined) {
					return 0
				}
				return parseInt(this.pollVotes[`option-${index}`] / this.votersNumber * 100)
			}
		},

		/**
		 * Current actor id
		 */
		actorId() {
			return this.$store.getters.getActorId()
		},

		/**
		 * Current actor type
		 */
		actorType() {
			return this.$store.getters.getActorType()
		},

		currentUserIsPollCreator() {
			return this.poll.actorType === this.actorType && this.poll.actorId === this.actorId
		},

		conversation() {
			return this.$store.getters.conversation(this.token)
		},

		participantType() {
			return this.conversation.participantType
		},

		currentUserIsModerator() {
			return [PARTICIPANT.TYPE.OWNER, PARTICIPANT.TYPE.MODERATOR, PARTICIPANT.TYPE.GUEST_MODERATOR].indexOf(this.participantType) !== -1
		},

		canEndPoll() {
			return (this.currentUserIsPollCreator || this.currentUserIsModerator) && this.pollIsOpen
		},

		pollFooterText() {
			if (this.pollIsOpen) {
				return this.selfHasVoted ? t('spreed', 'Poll ・ You voted already') : t('spreed', 'Poll ・ Click to vote')
			} else if (this.pollIsClosed) {
				return t('spreed', 'Poll ・ Ended')
			}
			return ''
		},
	},

	watch: {

		pollLoaded() {
			this.setVoteData()
		},

		modalPage(value) {
			if (value === 'voting') {
				this.setVoteData()
			}
		},

	},

	mounted() {
		this.setVoteData()
	},

	methods: {
		getPollData() {
			if (!this.pollLoaded) {
				this.$store.dispatch('getPollData', {
					token: this.token,
					pollId: this.id,
				})
			}
		},

		setVoteData() {
			if (this.checkboxRadioSwitchType === 'radio') {
				this.vote = ''
				if (this.selfHasVoted) {
					this.vote = this.votedSelf[0].toString()
				}
			} else {
				this.vote = []
				if (this.selfHasVoted) {
					this.vote = this.votedSelf.map(element => element.toString())
				}
			}
		},

		openPoll() {
			if (this.selfHasVoted || this.pollIsClosed) {
				this.modalPage = 'results'
			} else {
				this.modalPage = 'voting'
			}
			this.showModal = true
		},

		dismissModal() {
			this.showModal = false
			// Reset the data
			typeof this.vote === 'string' ? this.vote = '' : this.vote = []
		},

		submitVote() {
			let voteToSubmit = this.vote
			// If its a radio, we add the selected index to the array
			if (!Array.isArray(this.vote)) {
				voteToSubmit = [this.vote]
			}
			this.$store.dispatch('submitVote', {
				token: this.token,
				pollId: this.id,
				vote: voteToSubmit.map(element => parseInt(element)),
			})
			this.modalPage = 'results'
		},

		endPoll() {
			this.$store.dispatch('endPoll', {
				token: this.token,
				pollId: this.id,
			})
			this.modalPage = 'results'
		},

		selfHasVotedOption(index) {
			if (this.votedSelf.includes(index)) {
				return true
			} else {
				return false
			}
		},

		getFilteredDetails(index) {
			if (!this.details) {
				return []
			}
			return this.details.filter((item) => {
				return item.optionId === index
			})
		},
	},
}
</script>

<style lang="scss" scoped>
.wrapper {
	display: contents;
}
.poll {
	display: flex;
	transition: box-shadow 0.1s ease-in-out;
	border: 1px solid var(--color-border);
	box-shadow: 0 0 2px 0 var(--color-box-shadow);
	margin: 4px 0;
	max-width: 300px;
	padding: 0 16px 16px 16px;
	flex-direction: column;
	background: var(--color-main-background);
	border-radius: var(--border-radius-large);
	justify-content: space-between;

	&__header {
		display: flex;
		font-weight: bold;
		gap: 8px;
		white-space: normal;
		align-items: flex-start;
		top: 0;
		padding: 0 0 8px 0;
		word-wrap: anywhere;
		padding-top: 20px;

		span {
			margin-bottom: auto;
		}

	}
	&__footer {
		color: var(--color-text-lighter);
		white-space: normal;
		margin-top: 8px;
	}

	&__modal {
		position: relative;
		padding: 20px 20px 0 20px;
	}

	&__modal-title {
		margin: 0;
		font-size: 18px;
		font-weight: bold;
	}

	&__modal-options {
		word-wrap: anywhere;
		margin-top: 8px;
	}

	&__modal-actions {
		position: sticky;
		bottom: 0;
		display: flex;
		justify-content: center;
		gap: 8px;
		padding: 12px 0 0 0;
		background-color: var(--color-main-background);
		padding-bottom: 20px;
	}

	&__summary {
		color: var(--color-text-maxcontrast);
		margin-bottom: 16px;
	}
}

.results__options {
	display: flex;
	flex-direction: column;
	gap: 24px;
	word-wrap: anywhere;
	margin: 8px 0 20px 0;
}
.results__option {
	display: flex;
	flex-direction: column;

	&__details {
		display: flex;
		height: 32px;
	}

	&-subtitle {
		color: var(--color-text-maxcontrast);
	}

	&-progress {
		margin-top: 4px;
	}
}

.results__option-title {
	display: flex;
	justify-content: space-between;
	align-items: flex-start;
	margin-bottom: 4px;
	.percentage {
		white-space: nowrap;
		margin-left: 16px;
	}
}

.poll-closed {
	display: flex;
	justify-content: center;
	margin-top: 4px;
}

// Upstream
::v-deep .checkbox-radio-switch {
	&__label {
		align-items: unset;
		height: unset;
		margin: 4px 0;
		padding: 8px;
		width: 100%;
		border-radius: var(--border-radius-large);
		span {
			align-self: flex-start;
		}
	}
}
</style>

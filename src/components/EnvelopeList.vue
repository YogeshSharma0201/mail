<template>
	<transition-group name="list"
					  tag="div"
					  class="app-content-list"
					  v-infinite-scroll="loadMore"
					  infinite-scroll-disabled="loading"
					  infinite-scroll-distance="30"
					  v-scroll="onScroll"
					  v-shortkey.once="{next: ['arrowright'], prev: ['arrowleft']}" @shortkey.native="navigateEnvelope">
		<div id="list-refreshing"
			 key="loading"
			 class="icon-loading-small"
			 :class="{refreshing: refreshing}"/>
		<EmptyFolder v-if="envelopes.length === 0"
					 key="empty" />
		<Envelope v-else
				  v-for="env in envelopes"
				  :key="env.uid"
				  :data="env"
				  :show-account-color="folder.isUnified"/>
		<div id="load-more-mail-messages"
			 key="loadingMore"
			 :class="{'icon-loading-small': loadingMore}"/>
	</transition-group>
</template>

<script>
	import infiniteScroll from 'vue-infinite-scroll'
	import vuescroll from 'vue-scroll'
	import Vue from 'vue'

	import EmptyFolder from './EmptyFolder'
	import Envelope from './Envelope'

	Vue.use(vuescroll, {throttle: 600})

	export default {
		name: "EnvelopeList",
		computed: {
			envelopes () {
				return this.$store.getters.getEnvelopes(
					this.$route.params.accountId,
					this.$route.params.folderId
				)
			},
			folder () {
				return this.$store.getters.getFolder(
					this.$route.params.accountId,
					this.$route.params.folderId
				)
			}
		},
		components: {
			Envelope,
			EmptyFolder,
		},
		directives: {
			infiniteScroll,
		},
		data () {
			return {
				loadingMore: false,
				refreshing: false,
			}
		},
		methods: {
			loadMore () {
				this.loadingMore = true

				this.$store.dispatch('fetchNextEnvelopePage', {
					accountId: this.$route.params.accountId,
					folderId: this.$route.params.folderId,
				}).catch(console.error.bind(this)).then(() => {
					this.loadingMore = false
				})
			},
			onScroll (e, p) {
				if (p.scrollTop === 0 && !this.refreshing) {
					this.refreshing = true

					this.$store.dispatch('syncEnvelopes', {
						accountId: this.$route.params.accountId,
						folderId: this.$route.params.folderId,
					}).catch(console.error.bind(this)).then(() => {
						this.refreshing = false
					})
				}
			},
			navigateEnvelope (e) {
				const envelopes = this.envelopes
				const currentUid = this.$route.params.messageUid

				if (!currentUid) {
					console.debug('ignoring shortcut: no envelope selected')
					return
				}

				const current = envelopes.filter(e => e.uid == currentUid)
				if (current.length === 0) {
					console.debug('ignoring shortcut: currently displayed messages is not in current envelope list')
					return
				}

				const idx = envelopes.indexOf(current[0])
				let next
				if (e.srcKey === 'next') {
					next = envelopes[idx+1]
				} else {
					next = envelopes[idx-1]
				}

				if (!next) {
					console.debug('ignoring shortcut: head or tail of envelope list reached', envelopes, idx, e.srcKey)
					return
				}

				// Keep the selected account-folder combination, but navigate to a different message
				// (it's not a bug that we don't use next.accountId and next.folderId here)
				this.$router.push({
					name: 'message',
					params: {
						accountId: this.$route.params.accountId,
						folderId: this.$route.params.folderId,
						messageUid: next.uid,
					}
				});
			}
		}
	}
</script>

<style scoped>
	#load-more-mail-messages {
		margin: 10px auto;
		padding: 10px;
		margin-top: 50px;
		margin-bottom: 200px;
	}

	/* TODO: put this in core icons.css as general rule for buttons with icons */
	#load-more-mail-messages.icon-loading-small {
		padding-left: 32px;
		background-position: 9px center;
	}

	#list-refreshing {
		overflow-y: hidden;
		min-height: 0;

		transition-property: all;
		transition-duration: .5s;
		transition-timing-function: ease-in;
	}

	#list-refreshing.refreshing {
		min-height: 32px;
	}

	.list-enter-active, .list-leave-active {
		transition: all 1s;
	}
	.list-enter, .list-leave-to {
		opacity: 0;
		height: 0px;
		transform: scaleY(0);
	}
</style>

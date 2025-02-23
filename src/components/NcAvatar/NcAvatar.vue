<!--
  - @copyright Copyright (c) 2018 Julius Härtl <jus@bitgrid.net>
  -
  - @author Julius Härtl <jus@bitgrid.net>
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

<docs>

### Basic user avatar

```vue
	<NcAvatar user="willywonka" display-name="Willy Wonka" />
```

### Avatar with image

```vue
	<NcAvatar url="https://nextcloud.com/wp-content/themes/next/assets/img/common/nextcloud-square-logo.png" />
```

### Avatar with material design icon

```
 <template>
	<NcAvatar>
		<template #icon>
			<AccountMultiple :size="20" />
		</template>
	</NcAvatar>
</template>
<script>
import AccountMultiple from 'vue-material-design-icons/AccountMultiple'

export default {
	components: {
		AccountMultiple,
	},
 }
 </script>
```

### Avatar with preloaded status
```
 <template>
	<NcAvatar user="janedoe"
		display-name="Jane Doe"
		:preloaded-user-status="status">
	</NcAvatar>
</template>
<script>

export default {
	data() {
		return {
			status: {
				icon: '',
				status: 'dnd',
				message: 'Not busy',
			},
		}
	},
 }
 </script>
```

### Avatar for non-users

```vue
	<NcAvatar display-name="Robbie Hyeon-Jeong" :is-no-user="true" />
```

### Avatar on complex background

```
<template>
	<div class="avatar-background">
		<NcAvatar class="avatar" :is-no-user="true" display-name="Cecilia Rohese" />
	</div>
</template>
<style scoped>
.avatar-background {
	width: 80px;
	height: 60px;
	background: linear-gradient(to bottom, #0057b8 0%, #0057b8 49.99%, #ffd700 50%, #ffd700 100%);
}

.avatar {
	margin: 15px 25px;
}
</style>
```

</docs>
<template>
	<span ref="main"
		v-click-outside="closeMenu"
		:class="{
			'avatardiv--unknown': userDoesNotExist,
			'avatardiv--with-menu': hasMenu,
			'avatardiv--with-menu-loading': contactsMenuLoading
		}"
		:style="avatarStyle"
		class="avatardiv popovermenu-wrapper">
		<!-- @slot Icon slot -->
		<slot name="icon">
			<!-- Avatar icon or image -->
			<span v-if="iconClass" :class="iconClass" class="avatar-class-icon" />
			<img v-else-if="isAvatarLoaded && !userDoesNotExist"
				:src="avatarUrlLoaded"
				:srcset="avatarSrcSetLoaded"
				alt="">
		</slot>

		<!-- Contact menu -->
		<!-- We show a button if the menu is not loaded yet. -->
		<NcButton v-if="hasMenu && menu.length === 0"
			type="tertiary-no-background"
			class="action-item action-item__menutoggle"
			:aria-label="avatarAriaLabel"
			:title="tooltip"
			@click="toggleMenu">
			<template #icon>
				<NcLoadingIcon v-if="contactsMenuLoading" />
				<DotsHorizontal v-else :size="20" />
			</template>
		</NcButton>
		<NcActions v-else-if="hasMenu"
			force-menu
			manual-open
			type="tertiary-no-background"
			:container="menuContainer"
			:open.sync="contactsMenuOpenState"
			:aria-label="avatarAriaLabel"
			:title="tooltip"
			@click="toggleMenu">
			<NcActionLink v-for="(item, key) in menu"
				:key="key"
				:href="item.href"
				:icon="item.icon">
				{{ item.text }}
			</NcActionLink>
			<template v-if="contactsMenuLoading" #icon>
				<NcLoadingIcon />
			</template>
		</NcActions>

		<!-- Avatar status -->
		<span v-if="showUserStatusIconOnAvatar" class="avatardiv__user-status avatardiv__user-status--icon">
			{{ userStatus.icon }}
		</span>
		<NcUserStatusIcon v-else-if="canDisplayUserStatus"
			class="avatardiv__user-status"
			:status="userStatus.status"
			:aria-hidden="String(hasMenu)" />

		<!-- Show the letter if no avatar nor icon class -->
		<span v-if="showInitials"
			:style="initialsWrapperStyle"
			class="avatardiv__initials-wrapper">
			<span :style="initialsStyle" class="avatardiv__initials">
				{{ initials }}
			</span>
		</span>
	</span>
</template>

<script>
import NcActions from '../NcActions/index.js'
import NcActionLink from '../NcActionLink/index.js'
import NcButton from '../NcButton/index.js'
import NcLoadingIcon from '../NcLoadingIcon/index.js'
import NcUserStatusIcon from '../NcUserStatusIcon/index.js'
import usernameToColor from '../../functions/usernameToColor/index.js'
import { getUserStatusText } from '../../utils/UserStatus.ts'
import { userStatus } from '../../mixins/index.js'
import { t } from '../../l10n.js'

import axios from '@nextcloud/axios'
import DotsHorizontal from 'vue-material-design-icons/DotsHorizontal.vue'

import { getCurrentUser } from '@nextcloud/auth'
import { subscribe, unsubscribe } from '@nextcloud/event-bus'
import { getBuilder } from '@nextcloud/browser-storage'
import { generateUrl } from '@nextcloud/router'
import { vOnClickOutside as ClickOutside } from '@vueuse/components'

const browserStorage = getBuilder('nextcloud').persist().build()

/**
 * @param {string} userId The id of the user
 */
function getUserHasAvatar(userId) {
	const flag = browserStorage.getItem('user-has-avatar.' + userId)
	if (typeof flag === 'string') {
		return Boolean(flag)
	}
	return null
}

/**
 * @param {string} userId The id of the user
 * @param {boolean} flag Has the user an avatar
 */
function setUserHasAvatar(userId, flag) {
	if (userId) {
		browserStorage.setItem('user-has-avatar.' + userId, flag)
	}
}

export default {
	name: 'NcAvatar',

	directives: {
		ClickOutside,
	},
	components: {
		DotsHorizontal,
		NcActions,
		NcActionLink,
		NcButton,
		NcLoadingIcon,
		NcUserStatusIcon,
	},
	mixins: [userStatus],
	props: {
		/**
		 * Set a custom url to the avatar image
		 * either the url, user or displayName property must be defined
		 */
		url: {
			type: String,
			default: undefined,
		},
		/**
		 * Set a css icon-class for an icon to be used instead of the avatar.
		 */
		iconClass: {
			type: String,
			default: undefined,
		},
		/**
		 * Set the user id to fetch the avatar
		 * either the url, user or displayName property must be defined
		 */
		user: {
			type: String,
			default: undefined,
		},
		/**
		 * Whether or not to display the user-status
		 */
		showUserStatus: {
			type: Boolean,
			default: true,
		},
		/**
		 * Whether or not to the status-icon should be used instead of online/away
		 */
		showUserStatusCompact: {
			type: Boolean,
			default: true,
		},
		/**
		 * When the user status was preloaded via another source it can be handed in with this property to save the request.
		 * If this property is not set the status will be fetched automatically.
		 * If a preloaded no-status is available provide this object with properties "status", "icon" and "message" set to null.
		 */
		preloadedUserStatus: {
			type: Object,
			default: undefined,
		},
		/**
		 * Is the user a guest user (then we have to user a different endpoint)
		 */
		isGuest: {
			type: Boolean,
			default: false,
		},
		/**
		 * Set a display name that will be rendered as a tooltip
		 * either the url, user or displayName property must be defined
		 * specify just the displayname to generate a placeholder avatar without
		 * trying to fetch the avatar based on the user id
		 */
		displayName: {
			type: String,
			default: undefined,
		},
		/**
		 * Set a size in px for the rendered avatar
		 */
		size: {
			type: Number,
			default: 32,
		},
		/**
		 * Placeholder avatars will be automatically generated when this is set to true
		 */
		allowPlaceholder: {
			type: Boolean,
			default: true,
		},
		/**
		 * Disable the tooltip
		 */
		disableTooltip: {
			type: Boolean,
			default: false,
		},
		/**
		 * Disable the menu
		 */
		disableMenu: {
			type: Boolean,
			default: false,
		},
		/**
		 * Declares a custom tooltip when not null
		 * Fallback will be the displayName
		 *
		 * requires disableTooltip not to be set to true
		 */
		tooltipMessage: {
			type: String,
			default: null,
		},

		/**
		 * Declares username is not a user's name, when true.
		 * Prevents loading user's avatar from server and forces generating colored initials,
		 * i.e. if the user is a group
		 */
		isNoUser: {
			type: Boolean,
			default: false,
		},

		/**
		 * Selector for the popover menu container
		 */
		menuContainer: {
			type: [String, Object, Element, Boolean],
			default: 'body',
		},
	},
	data() {
		return {
			avatarUrlLoaded: null,
			avatarSrcSetLoaded: null,
			userDoesNotExist: false,
			isAvatarLoaded: false,
			isMenuLoaded: false,
			contactsMenuLoading: false,
			contactsMenuActions: [],
			contactsMenuOpenState: false,
		}
	},
	computed: {
		avatarAriaLabel() {
			// aria-label is only allowed on interactive elements
			if (!this.hasMenu) {
				return
			}
			if (this.canDisplayUserStatus || this.showUserStatusIconOnAvatar) {
				return t('Avatar of {displayName}, {status}', { displayName: this.displayName ?? this.user, status: getUserStatusText(this.userStatus.status) })
			}
			return t('Avatar of {displayName}', { displayName: this.displayName ?? this.user })
		},
		canDisplayUserStatus() {
			return this.showUserStatus
				&& this.hasStatus
				&& ['online', 'away', 'busy', 'dnd'].includes(this.userStatus.status)
		},
		showUserStatusIconOnAvatar() {
			return this.showUserStatus
				&& this.showUserStatusCompact
				&& this.hasStatus
				&& this.userStatus.status !== 'dnd'
				&& this.userStatus.icon
		},
		/**
		 * The user identifier, either the display name if set or the user property
		 * If both properties are not set an empty string is returned
		 */
		userIdentifier() {
			if (this.isDisplayNameDefined) {
				return this.displayName
			}
			if (this.isUserDefined) {
				return this.user
			}
			return ''
		},
		isUserDefined() {
			return typeof this.user !== 'undefined'
		},
		isDisplayNameDefined() {
			return typeof this.displayName !== 'undefined'
		},
		isUrlDefined() {
			return typeof this.url !== 'undefined'
		},
		hasMenu() {
			if (this.disableMenu) {
				return false
			}
			if (this.isMenuLoaded) {
				return this.menu.length > 0
			}
			return !(this.user === getCurrentUser()?.uid || this.userDoesNotExist || this.url)
		},

		/**
		 * True if initials should be shown as the user icon fallback
		 */
		showInitials() {
			return this.allowPlaceholder && this.userDoesNotExist && !(this.iconClass || this.$slots.icon)
		},

		avatarStyle() {
			const style = {
				'--size': this.size + 'px',
				lineHeight: this.size + 'px',
				fontSize: Math.round(this.size * 0.45) + 'px',
			}
			return style
		},
		initialsWrapperStyle() {
			const { r, g, b } = usernameToColor(this.userIdentifier)
			return {
				backgroundColor: `rgba(${r}, ${g}, ${b}, 0.1)`,
			}
		},
		initialsStyle() {
			const { r, g, b } = usernameToColor(this.userIdentifier)
			return {
				color: `rgb(${r}, ${g}, ${b})`,
			}
		},
		tooltip() {
			if (this.disableTooltip) {
				return false
			}
			if (this.tooltipMessage) {
				return this.tooltipMessage
			}

			return this.displayName
		},

		/**
		 * Get the (max. two) initials of the user as uppcase string
		 */
		initials() {
			let initials = '?'
			if (this.showInitials) {
				const user = this.userIdentifier.trim()
				if (user === '') {
					return '?'
				}

				/**
				 * Filtered user name, without special characters so only letters and numbers are allowed (prevent e.g. '(' as an initial)
				 * \p{L}: Letters of all languages
				 * \p{N}: Numbers of all languages
				 * \s: White space for breaking the string
				 * @type {string}
				 */
				const filtered = user.match(/[\p{L}\p{N}\s]/gu).join('')
				const idx = filtered.lastIndexOf(' ')
				initials = String.fromCodePoint(filtered.codePointAt(0))
				if (idx !== -1) {
					initials = initials.concat(String.fromCodePoint(filtered.codePointAt(idx + 1)))
				}
			}
			return initials.toLocaleUpperCase()
		},
		menu() {
			const actions = this.contactsMenuActions.map((item) => {
				return {
					href: item.hyperlink,
					icon: item.icon,
					text: item.title,
				}
			})

			/**
			 * @param {string} html The HTML to escape
			 */
			function escape(html) {
				const text = document.createTextNode(html)
				const p = document.createElement('p')
				p.appendChild(text)
				return p.innerHTML
			}

			if (this.showUserStatus && (this.userStatus.icon || this.userStatus.message)) {
				return [{
					href: '#',
					icon: `data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg'><text x='0' y='14' font-size='14'>${escape(this.userStatus.icon)}</text></svg>`,
					text: `${this.userStatus.message}`,
				}].concat(actions)
			}

			return actions
		},
	},

	watch: {
		url() {
			this.userDoesNotExist = false
			this.loadAvatarUrl()
		},
		user() {
			this.userDoesNotExist = false
			this.isMenuLoaded = false
			this.loadAvatarUrl()
		},
	},

	mounted() {
		this.loadAvatarUrl()
		subscribe('settings:avatar:updated', this.loadAvatarUrl)
		subscribe('settings:display-name:updated', this.loadAvatarUrl)
		if (this.showUserStatus && this.user && !this.isNoUser) {
			if (!this.preloadedUserStatus) {
				this.fetchUserStatus(this.user)
			} else {
				this.userStatus.status = this.preloadedUserStatus.status || ''
				this.userStatus.message = this.preloadedUserStatus.message || ''
				this.userStatus.icon = this.preloadedUserStatus.icon || ''
				this.hasStatus = this.preloadedUserStatus.status !== null
			}
			subscribe('user_status:status.updated', this.handleUserStatusUpdated)
		}
	},

	beforeDestroy() {
		unsubscribe('settings:avatar:updated', this.loadAvatarUrl)
		unsubscribe('settings:display-name:updated', this.loadAvatarUrl)
		if (this.showUserStatus && this.user && !this.isNoUser) {
			unsubscribe('user_status:status.updated', this.handleUserStatusUpdated)
		}
	},

	methods: {
		t,
		handleUserStatusUpdated(state) {
			if (this.user === state.userId) {
				this.userStatus = {
					status: state.status,
					icon: state.icon,
					message: state.message,
				}
			}
		},

		/**
		 * Toggle the popover menu on click or enter
		 * @param {KeyboardEvent|MouseEvent} event the UI event
		 */
		async toggleMenu(event) {
			if (event.type === 'keydown' && event.key !== 'Enter') {
				return
			}
			if (!this.contactsMenuOpenState) {
				await this.fetchContactsMenu()
			}
			this.contactsMenuOpenState = !this.contactsMenuOpenState
		},
		closeMenu() {
			this.contactsMenuOpenState = false
		},
		async fetchContactsMenu() {
			this.contactsMenuLoading = true
			try {
				const user = encodeURIComponent(this.user)
				const { data } = await axios.post(generateUrl('contactsmenu/findOne'), `shareType=0&shareWith=${user}`)
				this.contactsMenuActions = data.topAction ? [data.topAction].concat(data.actions) : data.actions
			} catch (e) {
				this.contactsMenuOpenState = false
			}
			this.contactsMenuLoading = false
			this.isMenuLoaded = true
		},

		/**
		 * Handle avatar loading if user or url defined
		 */
		loadAvatarUrl() {
			this.isAvatarLoaded = false

			/** Only run avatar image loading if either user or url property is defined */
			if (!this.isUrlDefined && (!this.isUserDefined || this.isNoUser)) {
				this.isAvatarLoaded = true
				this.userDoesNotExist = true
				return
			}

			// Directly use the url if defined
			if (this.isUrlDefined) {
				this.updateImageIfValid(this.url)
				return
			}

			if (this.size <= 64) {
				const avatarUrl = this.avatarUrlGenerator(this.user, 64)
				const srcset = [
					avatarUrl + ' 1x',
					this.avatarUrlGenerator(this.user, 512) + ' 8x',
				].join(', ')

				this.updateImageIfValid(avatarUrl, srcset)
			} else {
				const avatarUrl = this.avatarUrlGenerator(this.user, 512)
				this.updateImageIfValid(avatarUrl)
			}
		},

		/**
		 * Generate an avatar url from the server's avatar endpoint
		 *
		 * @param {string} user the user id
		 * @param {number} size the desired size
		 * @return {string}
		 */
		avatarUrlGenerator(user, size) {
			const darkTheme = window.getComputedStyle(document.body)
				.getPropertyValue('--background-invert-if-dark') === 'invert(100%)'
			let url = '/avatar/{user}/{size}' + (darkTheme ? '/dark' : '')
			if (this.isGuest) {
				url = '/avatar/guest/{user}/{size}' + (darkTheme ? '/dark' : '')
			}

			let avatarUrl = generateUrl(
				url,
				{
					user,
					size,
				})

			// eslint-disable-next-line camelcase
			if (user === getCurrentUser()?.uid && typeof oc_userconfig !== 'undefined') {
				avatarUrl += '?v=' + oc_userconfig.avatar.version
			}

			return avatarUrl
		},

		/**
		 * Check if the provided url is valid and update Avatar if so
		 *
		 * @param {string} url the avatar url
		 * @param {Array} srcset the avatar srcset
		 */
		updateImageIfValid(url, srcset = null) {
			// skip loading
			const userHasAvatar = getUserHasAvatar(this.user)
			if (this.isUserDefined && typeof userHasAvatar === 'boolean') {
				this.isAvatarLoaded = true
				this.avatarUrlLoaded = url
				if (srcset) {
					this.avatarSrcSetLoaded = srcset
				}
				if (userHasAvatar === false) {
					this.userDoesNotExist = true
				}
				return
			}

			const img = new Image()
			img.onload = () => {
				this.avatarUrlLoaded = url
				if (srcset) {
					this.avatarSrcSetLoaded = srcset
				}
				this.isAvatarLoaded = true
				// re-get to avoid concurrent access
				setUserHasAvatar(this.user, true)
			}
			img.onerror = () => {
				console.debug('Invalid avatar url', url)
				// Avatar is invalid, reset
				this.avatarUrlLoaded = null
				this.avatarSrcSetLoaded = null

				this.userDoesNotExist = true
				this.isAvatarLoaded = false
				setUserHasAvatar(this.user, false)
			}

			if (srcset) {
				img.srcset = srcset
			}
			img.src = url
		},
	},
}
</script>

<style scoped lang="scss">
.avatardiv {
	position: relative;
	display: inline-block;
	width: var(--size);
	height: var(--size);

	&--unknown {
		position: relative;
		background-color: var(--color-main-background);
		white-space: normal;
	}

	&:not(&--unknown) {
		// White/black background for avatars with transparency
		background-color: var(--color-main-background) !important;
		box-shadow: 0 0 5px rgba(0, 0, 0, 0.05) inset;
	}

	&--with-menu {
		cursor: pointer;
		.action-item {
			position: absolute;
			top: 0;
			left: 0;
		}
		:deep(.action-item__menutoggle) {
			cursor: pointer;
			opacity: 0;
		}
		&:focus-within,
		&:hover,
		&#{&}-loading {
			:deep(.action-item__menutoggle) {
				opacity: 1;
			}
			img {
				opacity: 0.3;
			}
		}
		:deep(.action-item__menutoggle),
		img {
			transition: opacity var(--animation-quick);
		}
		:deep() {
			.button-vue,
			.button-vue__icon {
				height: var(--size);
				min-height: var(--size);
				width: var(--size) !important;
				min-width: var(--size);
			}
		}
	}

	.avatardiv__initials-wrapper {
		display: block;
		height: var(--size);
		width: var(--size);
		background-color: var(--color-main-background);
		border-radius: 50%;

		.avatardiv__initials {
			position: absolute;
			top: 0;
			left: 0;
			display: block;
			width: 100%;
			text-align: center;
			font-weight: normal;
		}
	}

	img {
		// Cover entire area
		width: 100%;
		height: 100%;
		// Keep ratio
		object-fit: cover;
	}

	.material-design-icon {
		width: var(--size);
		height: var(--size);
	}

	.avatardiv__user-status {
		box-sizing: border-box;
		position: absolute;
		right: -4px;
		bottom: -4px;
		min-height: 18px;
		min-width: 18px;
		max-height: 18px;
		max-width: 18px;
		height: 40%;
		width: 40%;
		line-height: 15px;
		font-size: var(--default-font-size);
		border: 2px solid var(--color-main-background);
		background-color: var(--color-main-background);
		background-repeat: no-repeat;
		background-size: 16px;
		background-position: center;
		border-radius: 50%;

		.acli:hover & {
			border-color: var(--color-background-hover);
			background-color: var(--color-background-hover);
		}
		.acli.active & {
			border-color: var(--color-primary-element-light);
			background-color: var(--color-primary-element-light);
		}

		&--icon {
			border: none;
			background-color: transparent;
		}
	}

	.popovermenu-wrapper {
		position: relative;
		display: inline-block;
	}
}

.avatar-class-icon {
	display: block;
	border-radius: 50%;
	background-color: var(--color-background-darker);
	height: 100%;
}

</style>

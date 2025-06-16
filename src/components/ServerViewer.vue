<script setup lang="ts">
import { useFetch, useLocalStorage, useSessionStorage, useUrlSearchParams } from '@vueuse/core'
import { computed, watchEffect } from 'vue'

const discordToken = useSessionStorage<string | null>('discord_token', null)
const oauthState = useSessionStorage<string | null>('oauth_state', null)

// access_token, token_type, expires_in, scope, and state

function startDiscordLoginButton() {
  oauthState.value = crypto.randomUUID()

  const params = new URLSearchParams({
    response_type: 'token',
    redirect_uri: window.location.origin,
    // discord public client.
    client_id: '1383942247464304660',
    state: oauthState.value,
    scope: ['identify', 'guilds'].join(' '),
    prompt: 'consent',
  })

  window.location.href = 'https://discord.com/oauth2/authorize?' + params.toString()
}

function logoutDiscord() {
  discordToken.value = null
  oauthState.value = null
  window.location.hash = ''
}

function useStringHashParam(name: string) {
  const params = useUrlSearchParams('hash-params')
  return computed(() => {
    const value = params[name]

    if (value === undefined || value === null) {
      return undefined
    }

    return typeof value === 'string' ? value : ''
  })
}

const accessToken = useStringHashParam('access_token')
const tokenType = useStringHashParam('token_type')
const expiresIn = useStringHashParam('expires_in')
const scope = useStringHashParam('scope')
const state = useStringHashParam('state')

const validRedirectParams = computed(() =>
  Boolean(accessToken.value && tokenType.value && expiresIn.value && scope.value && state.value),
)

const validState = computed(() => validRedirectParams.value && state.value === oauthState.value)

watchEffect(() => {
  if (validState.value) {
    discordToken.value = accessToken.value
    // clear oauth state
    oauthState.value = null

    // clear hash params
    window.location.hash = ''
  }
})

interface DiscordUserData {
  id: string
  username: string
  discriminator: string
  avatar: string | null
  global_name: string | null // Optional field for global name
}

// if has valid token, fetch user data
const {
  data: discordUserData,
  execute: updateDiscordUserData,
  onFetchError: onDiscordUserDataFetchError,
} = useFetch('https://discord.com/api/v10/users/@me', {
  async beforeFetch({ options, cancel }) {
    const token = discordToken.value

    if (!token) {
      console.warn('No Discord token available, cannot fetch user data.')
      cancel()
    }

    options.headers = {
      ...options.headers,
      Authorization: `Bearer ${token}`,
      Accept: 'application/json',
    }

    return {
      options,
    }
  },
  immediate: false,
}).json<DiscordUserData>()

watchEffect(async () => {
  if (discordToken.value) {
    console.log('Fetching Discord user data with token:', discordToken.value)
    await updateDiscordUserData()
    await updateDiscordGuildData()
  } else {
    discordUserData.value = null // Clear user data if no token
    discordGuildData.value = null // Clear guild data if no token
  }
})

onDiscordUserDataFetchError((error) => {
  console.error('Failed to fetch Discord user data:', error)
  discordToken.value = null // Clear token on error
  oauthState.value = null // Clear oauth state on error
  window.location.hash = '' // Clear hash params on error
})

interface DiscordGuildData {
  id: string
  name: string
  icon: string | null
  // Add other fields as needed
}

// if has valid token, fetch user data
const {
  data: discordGuildData,
  execute: updateDiscordGuildData,
  onFetchError: onDiscordGuildDataFetchError,
} = useFetch('https://discord.com/api/v10/users/@me/guilds', {
  async beforeFetch({ options, cancel }) {
    const token = discordToken.value

    if (!token) {
      console.warn('No Discord token available, cannot fetch guild data.')
      cancel()
    }

    options.headers = {
      ...options.headers,
      Authorization: `Bearer ${token}`,
      Accept: 'application/json',
    }

    return {
      options,
    }
  },
  immediate: false,
}).json<DiscordGuildData[]>()

const sortedGuilds = computed(() => {
  if (!discordGuildData.value) return []
  return [...discordGuildData.value].sort((a, b) => a.name.localeCompare(b.name))
})

onDiscordGuildDataFetchError((error) => {
  console.error('Failed to fetch Discord user data:', error)
  discordToken.value = null // Clear token on error
  oauthState.value = null // Clear oauth state on error
  window.location.hash = '' // Clear hash params on error
})
</script>

<template>
  <div>
    <h1>Discord Server Viewer</h1>
    <div v-if="!discordToken">
      <button @click="startDiscordLoginButton">Login with Discord</button>
      <p class="note">
        NOTE: This app is fully client-sided, <br />
        and only stores the token in session storage: <br />
        closing the tab should delete the token.
      </p>
    </div>
    <button @click="logoutDiscord" v-else>Logout</button>

    <div v-if="discordUserData">
      <h2>You are logged in as</h2>
      <div class="info">
        <img v-if="discordUserData.avatar"
          :src="`https://cdn.discordapp.com/avatars/${discordUserData.id}/${discordUserData.avatar}.png`" alt="Avatar"
          class="icon" />

        <img v-else :src="`https://cdn.discordapp.com/embed/avatars/${(BigInt(discordUserData.id) >> 22n) % 6n}.png`"
          alt="Default Avatar" class="icon" />

        <p>{{ discordUserData.global_name ?? discordUserData.username }}</p>
      </div>
    </div>

    <div v-if="discordGuildData !== null">
      <h2>You are a member of {{ discordGuildData.length }} Guilds</h2>
      <ul v-if="discordGuildData.length > 0" class="guild-list">
        <li v-for="(guild, i) in sortedGuilds" :key="guild.id" class="guild-item">
          <div class="info">
            <p>{{ i + 1 }} / {{ discordGuildData.length }}:</p>

            <img v-if="guild.icon" :src="`https://cdn.discordapp.com/icons/${guild.id}/${guild.icon}.png`"
              alt="Guild Icon" class="icon" />
            <img v-else src="https://cdn.discordapp.com/embed/avatars/0.png" alt="Default Guild Icon" class="icon" />
            <p>{{ guild.name }}</p>
          </div>
        </li>
      </ul>
    </div>
  </div>
</template>

<style scoped>
.icon {
  width: 3em;
  height: 3em;
  border-radius: 50%;
}

.info {
  display: flex;
  align-items: center;
  gap: 1em;
}

.guild-list {
  list-style: none;
  padding: 0;
}

.guild-item {
  display: flex;
  align-items: center;
  gap: 1em;
  margin-bottom: 1em;
}

.note {
  font-size: 0.8em;
  background-color: #d4d421;
  border-radius: 1em;
  padding: 1em;
  color: black;
}
</style>

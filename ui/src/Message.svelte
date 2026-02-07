<script lang="ts" context="module">
  import { writable } from 'svelte/store'
  export let messageDestination = writable(0)
  export let replyToId = writable<number | null>(null) 
</script>

<script lang="ts">
  import { channels, messagePrefix, messageSuffix } from 'api/src/vars'
  import Card from './lib/Card.svelte'
  import { filteredNodes, smallMode } from './Nodes.svelte'
  import axios from 'axios'
  import { getNodeName } from './lib/util'

  let inputElement: HTMLInputElement
  let message = ''

  $: maxLength = 230 - ($messagePrefix?.length || 0) - ($messageSuffix?.length || 0)
  $: remainingChars = maxLength - message.length
  $: charCountClass = remainingChars <= 0 ? 'text-red-700' : remainingChars <= 40 ? 'text-yellow-700' : 'text-gray-600'

  $: if (inputElement && $messageDestination || $replyToId) {
    inputElement.focus()
  }

  function cancelReply() {
    $replyToId = null
  }

  function send() {
    if (!message) return

    let payload = { message }
    if (channels.value.some((c) => c.index == $messageDestination)) {
      payload['channel'] = $messageDestination
      if ($replyToId) {
        payload['replyToId'] = $replyToId
      }

    } else {
      payload['destination'] = $messageDestination
    }

    axios.post('/send', payload).then(() => {
      message = ''
    })

    cancelReply();
  }
</script>

<Card class="shrink-0">
  <h2 slot="title" class="rounded-t flex items-center h-full gap-2">
    {#if !$smallMode}
      <div class="grow">Message</div>
      <div class="text-xs {charCountClass}">{remainingChars}</div>
    {/if}

    <select bind:value={$messageDestination} class="input font-normal text-sm border border-blue-500/50 !bg-blue-950" name="" id="">
      <option disabled>== Channels ==</option>
      {#each $channels as channel}
        {#if channel.role != 'DISABLED'}
          <option value={channel.index}>{channel.settings.name || `Channel ${channel.index}`}</option>
        {/if}
      {/each}

      <option disabled>== Nodes ==</option>
      {#each [...$filteredNodes].sort((a, b) => {
        return getNodeName(a).localeCompare(getNodeName(b))
      }) as node}
        <option value={node.num}>{getNodeName(node)}</option>
      {/each}
    </select>
  </h2>

  <form on:submit|preventDefault={send} class="p-2 flex flex-col gap-1 text-sm">
    <div class="flex gap-1" class:flex-col={$smallMode}>
      <input maxlength={maxLength} bind:this={inputElement} class="input w-full" size="3" type="text" bind:value={message} />
      {#if $replyToId}
      <div class="flex items-center gap-2">
        <button class="btn">
          Reply
        </button>
        <button 
          class="btn bg-red-600 hover:bg-red-700 border-red-700"
          type="button"
          on:click={() => $replyToId = null}
          title="Cancel reply"
        >
          ✕
        </button>
      </div>
    {:else}
      <button class="btn">
        Send
      </button>
    {/if}
    </div>
  </form>
</Card>

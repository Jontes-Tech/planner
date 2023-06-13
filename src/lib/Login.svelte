<script lang="ts">
  import { currentUser, pb } from "./pocketbase";
  let email = "";
  let password = "";

  async function login() {
    await pb.collection("users").authWithPassword(email, password);
  }
</script>

{#if $currentUser}
  <p class="text-gray-300">Logged in as {$currentUser.email}</p>
{:else}
  <form on:submit|preventDefault={login} class="flex flex-col gap-2">
    <input
      class="p-2 bg-stone-800 text-white max-w-96"
      type="email"
      bind:value={email}
      placeholder="Email"
    />
    <input
      class="p-2 bg-stone-800 text-white max-w-96"
      type="password"
      bind:value={password}
      placeholder="Password"
    />
    <button class="mt-2 p-2 bg-stone-400 rounded" type="submit">Sign In</button>
  </form>
{/if}

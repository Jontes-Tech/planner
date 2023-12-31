<script lang="ts">
  import { writable } from "svelte/store";
  import Login from "./lib/Login.svelte";
  import { currentUser, pb } from "./lib/pocketbase";
  function getNextHundredDays() {
    const dates = [];
    const today = new Date();
    for (let i = 0; i < 100; i++) {
      const newDate = new Date(today);
      newDate.setDate(today.getDate() + i);
      dates.push(newDate);
    }
    return dates as Date[];
  }
  function englishOrdinalSuffix(dt) {
    return (
      dt.getDate() +
      (dt.getDate() % 10 == 1 && dt.getDate() != 11
        ? "st"
        : dt.getDate() % 10 == 2 && dt.getDate() != 12
        ? "nd"
        : dt.getDate() % 10 == 3 && dt.getDate() != 13
        ? "rd"
        : "th")
    );
  }
  const weekdays = [
    "Sunday",
    "Monday",
    "Tuesday",
    "Wednesday",
    "Thursday",
    "Friday",
    "Saturday",
  ];
  const months = [
    "January",
    "February",
    "March",
    "April",
    "May",
    "June",
    "July",
    "August",
    "September",
    "October",
    "November",
    "December",
  ];
  function formatDate(date: Date) {
    const weekday = weekdays[date.getDay()];
    const day = englishOrdinalSuffix(date);
    const month = months[date.getMonth()];
    return `${weekday} the ${day} ${month}`;
  }
  function getMidnightUnix(date: Date) {
    const midnight = new Date(date);
    midnight.setHours(0, 0, 0, 0);
    return midnight.getTime();
  }
  // Write an object that satisfies User interface
  const myUserdata = writable({
    collectionId: "",
    collectionName: "",
    created: "",
    emailVisibility: false,
    friends: [],
    id: "",
    not_free: [],
    updated: "",
    username: "",
    verified: false,
  });
  async function toggleDate(date: Date) {
    const midnightUnix = getMidnightUnix(date);
    myUserdata.set({
      ...$myUserdata,
      not_free: $myUserdata.not_free.includes(midnightUnix)
        ? $myUserdata.not_free.filter((time: number) => time !== midnightUnix)
        : [...$myUserdata.not_free, midnightUnix],
    });
    await pb.collection("users").update($currentUser.id, {
      not_free: $myUserdata.not_free,
    });
    display = false;
    doFetch();
  }
  interface User {
    collectionId: string;
    collectionName: string;
    created: string;
    emailVisibility: boolean;
    friends: string[];
    id: string;
    not_free: number[];
    updated: string;
    username: string;
    verified: boolean;
  }
  interface Users {
    page: number;
    perPage: number;
    totalItems: number;
    totalPages: number;
    items: User[];
  }
  let users: Users;
  let display = false;
  async function doFetch() {
    users = await pb.collection("users").getList(1, 50);
    const updatedUser = (await pb
      .collection("users")
      .getOne($currentUser.id)) as User;
    myUserdata.set(updatedUser);
    display = true;
  }
  function busyPeople(date: Date) {
    const busyUserIds = users.items
      .filter((user: User) => user.not_free.includes(getMidnightUnix(date)))
      .map((user: User) => user.username);
    return busyUserIds;
  }
  async function addFriend() {
    const friendUsername = prompt(
      "Enter your friend's ID (not username or email)"
    );
    const me = await pb.collection("users").getOne($currentUser.id);
    const existingFriends = me.friends;
    if (friendUsername) {
      if (existingFriends) {
        await pb.collection("users").update($currentUser.id, {
          friends: [...existingFriends, friendUsername],
        });
      } else {
        await pb.collection("users").update($currentUser.id, {
          friends: [friendUsername],
        });
      }
    }
  }
  doFetch();
</script>

<main class="m-8">
  <h1 class="text-white text-4xl">Planner</h1>
  <div>
    <Login />
  </div>
  {#if $currentUser}
    <button
      class="mt-2 p-2 bg-neutral-800 text-white rounded"
      on:click={() => {
        pb.authStore.clear();
      }}>Sign Out</button
    >
    <button
      class="mt-2 p-2 bg-neutral-800 text-white rounded"
      on:click={() => {
        display = false;
        doFetch();
      }}>Reload Friend's Statuses</button
    >
    <button
      class="mt-2 p-2 bg-neutral-800 text-white rounded"
      on:click={() => {
        addFriend();
      }}>Add Friend</button
    >
    <ol>
      {#each getNextHundredDays() as date}
        <!-- svelte-ignore a11y-click-events-have-key-events -->
        <li
          on:click={() => {
            toggleDate(date);
          }}
          class={"p-2 text-white mt-2 rounded " +
            ($myUserdata.not_free.includes(getMidnightUnix(date))
              ? "bg-stone-700"
              : "bg-stone-800")}
        >
          {formatDate(date)}
          {#if display}{#each busyPeople(date) as person}<span
                class="p-1 bg-violet-500 mr-2 rounded">{person}</span
              >{/each}{/if}
        </li>
      {/each}
    </ol>
    <span class="text-neutral-500"
      >You can only schedule busydays 100 days before</span
    >
  {/if}
</main>

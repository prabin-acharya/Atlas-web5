<!-- pages/todos/index.vue -->
<template>
  <div>
    <div class="container mx-auto p-4 w-2/3">
      <div class="w-3/4 m-auto">
        <header class="flex justify-between mb-4">
          <h1 class="text-4xl font-extrabold mb-4 text-orange-500">Atlas</h1>
          <p v-if="myDID" class="cursor-pointer" id="copy-did" @click="copyDID">
            <!-- Copy your DID -->
            {{ myDID.substr(0, 15) }}...
          </p>
        </header>
        <hr class="my-5" />

        <div v-if="!showForm">
          <div
            v-if="essayList.length === 0"
            class="empty-state text-center pt-20"
          >
            <p class="mb-4 text-gray-500">
              No Essays saved yet. Start Writing!
            </p>
          </div>

          <div v-else>
            <!-- my essays -->
            <p class="text-lg font-extrabold text-gray-500">My Essays</p>
            <ul class="mb-10">
              <li
                v-for="(essay, index) in essayList.filter(
                  (essay) => essay.data.author == myDID
                )"
                :key="index"
                class="p-2"
              >
                <nuxt-link :to="`/${essay.id}`" class="text-blue-500">
                  <h2 class="text-xl font-bold">{{ essay.data.title }}</h2>
                  <!-- <p>{{ essay.data.content }}</p> -->
                  <!-- <p class="text-gray-500">
                Created by: {{ essay.data.author.substr(0, 22) }}...
              </p> -->
                </nuxt-link>
              </li>
            </ul>

            <!-- shared with me -->
            <p
              v-if="
                essayList.filter((essay) => essay.data.author != myDID).length >
                0
              "
              class="text-lg font-extrabold text-gray-500"
            >
              Shared with me
            </p>
            <ul class="mb-10">
              <li
                v-for="(essay, index) in essayList.filter(
                  (essay) => essay.data.author != myDID
                )"
              >
                <nuxt-link :to="`/${essay.id}`" class="text-blue-500">
                  <h2 class="text-xl font-bold">{{ essay.data.title }}</h2>
                </nuxt-link>
              </li>
            </ul>
          </div>

          <div v-if="!showForm" class="text-center">
            <button
              @click="showForm = true"
              class="btn btn-primary flex items-center"
            >
              <plus-icon-mini class="w-4 h-4 m-2 mb-3" /> Write a New Essay
            </button>
          </div>
        </div>
      </div>
    </div>
    <!-- --------------------------------- -->
    <div class="w-1/2 flex m-auto p-4">
      <form
        v-if="showForm"
        @submit.prevent="createNewEssay"
        class="mb-4 w-full"
      >
        <div class="mb-4">
          <!-- <label for="title" class="block mb-2">Title:</label> -->
          <input
            type="text"
            id="title"
            placeholder="Title"
            v-model="newEssay.title"
            class="w-full p-2 rounded outline-none text-4xl font-bold"
            required
          />
        </div>
        <div class="mb-4">
          <textarea
            id="description"
            placeholder="write something amazing..."
            v-model="newEssay.content"
            rows="10"
            class="w-full p-2 outline-none rounded text-xl overflow-hidden"
            required
          ></textarea>
        </div>
        <div class="mb-4 flex align-middle items-center">
          <label for="userId" class="block mb-2 mr-4">Share with:</label>
          <input
            type="text"
            id="userId"
            placeholder="Enter recipients DID"
            v-model="newEssay.recipientDID"
            class="w-2/3 p-2 border rounded outline-none"
            required
          />
        </div>
        <button type="submit" class="btn btn-primary">Save</button>
        <button
          type="button"
          class="btn ml-4 btn-secondary"
          @click="showForm = false"
        >
          Cancel
        </button>
      </form>
    </div>
  </div>
</template>

<script setup>
import { ref } from "vue";
import protocolDefinition from "assets/shared-essays-protocol.json";
import { PlusIcon as PlusIconMini } from "@heroicons/vue/solid";

const { $web5: web5, $myDID: myDID } = useNuxtApp();

const showForm = ref(false);

//
const newEssay = ref({
  title: "",
  content: "",
  recipientDID: "",
});

const essayList = ref([]);

onBeforeMount(async () => {
  console.log("this is your DID", myDID);

  await configureProtocol();

  // Fetch shared todo lists.
  const { records } = await web5.dwn.records.query({
    message: {
      filter: {
        schema: protocolDefinition.types.essay.schema,
      },
      dateSort: "createdAscending",
    },
  });

  console.log("Saved records", records);

  // add entry to sharedList
  for (let record of records) {
    const data = await record.data.json();
    const list = { record, data, id: record.id };
    essayList.value.push(list);
  }
});

const copyDID = async () => {
  await navigator.clipboard.writeText(myDID);
  alert("DID copied to clipboard");
};

const configureProtocol = async () => {
  const { protocols, status } = await web5.dwn.protocols.query({
    message: {
      filter: {
        protocol: protocolDefinition.protocol,
      },
    },
  });

  if (status.code !== 200) {
    alert("Error querying protocols");
    console.error("Error querying protocols", status);
    return;
  }

  if (protocols.length > 0) {
    console.log("Protocol already exists");
    return;
  }

  // configure protocol on local DWN
  const { status: configureStatus, protocol } =
    await web5.dwn.protocols.configure({
      message: {
        definition: protocolDefinition,
      },
    });

  console.log("Protocol configured", configureStatus, protocol);

  // configuring protocol on remote DWN
  const { status: configureRemoteStatus } = protocol.send(myDID);
  console.log("Protocol configured on remote DWN", configureRemoteStatus);
};

// ============================================================================
// ============================================================================
// ============================================================================
// ============================================================================
// ============================================================================

const createNewEssay = async () => {
  let recipientDID = newEssay.value.recipientDID;
  const essayListData = {
    "@type": "essay",
    title: newEssay.value.title,
    content: newEssay.value.content,
    author: myDID,
    recipient: newEssay.value.recipientDID,
  };

  console.log(essayListData);

  newEssay.value = { title: "", description: "", recipientDID: "" };

  try {
    const { record } = await web5.dwn.records.create({
      data: essayListData,
      message: {
        protocol: protocolDefinition.protocol,
        protocolPath: "essay",
        schema: protocolDefinition.types.essay.schema,
        dataFormat: protocolDefinition.types.essay.dataFormats[0],
        recipient: recipientDID,
      },
    });

    const data = await record.data.json();
    const essay = { record, data, id: record.id };

    essayList.value.push(essay);
    showForm.value = false;

    const { status: sendStatus } = await record.send(recipientDID);

    if (sendStatus.code !== 202) {
      console.log("Unable to send to target did:" + sendStatus);
      return;
    } else {
      console.log("Shared list sent to recipient");
    }
  } catch (e) {
    console.error(e);
    return;
  }
};

// ============================================================================
</script>

<style scoped>
.btn {
  padding: 0.5rem 1rem;
  background-color: #499122;
  color: white;
  border: none;
  border-radius: 0.25rem;
  cursor: pointer;
  transition: background-color 0.3s;
}

.btn-secondary {
  padding: 0.5rem 1rem;
  background-color: #ccc;
  color: #333;
  border: none;
  border-radius: 0.25rem;
  cursor: pointer;
  transition: background-color 0.3s;
}

.btn-secondary:hover {
  background-color: #aaa;
}

.btn:hover {
  background-color: #238214;
}
</style>

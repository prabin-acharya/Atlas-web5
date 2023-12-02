<!-- pages/todos/index.vue -->
<template>
  <div>
    <div class="container mx-auto p-4 w-1/3">
      <header class="flex-col text-center mb-4">
        <h1 class="text-2xl font-bold mb-4">Shared Todo</h1>
        <p>Manage a set of todos towards your goals with friends.</p>
        <button v-if="myDID" class="btn" id="copy-did" @click="copyDID">
          Copy your DID
        </button>
      </header>
      <hr class="my-5" />
      <div v-if="essayList.length === 0" class="empty-state text-center pt-20">
        <p class="mb-4 text-gray-500">No Shared Todos yet.</p>
      </div>

      <div v-else>
        <ul class="mb-10">
          <li
            v-for="(essay, index) in essayList"
            :key="index"
            class="mb-2 p-4 border rounded"
          >
            <nuxt-link :to="`/${essay.id}`" class="text-blue-500">
              <h2 class="text-xl font-bold">{{ essay.data.title }}</h2>
              <p>{{ essay.data.content }}</p>
              <p class="text-gray-500">
                Created by: {{ essay.data.author.substr(0, 22) }}...
              </p>
            </nuxt-link>
          </li>
        </ul>
      </div>
      <div v-if="!showForm" class="text-center">
        <button @click="showForm = true" class="btn btn-primary">
          Create New Essay
        </button>
      </div>
    </div>
    <!-- --------------------------------- -->
    <div
      class="border border-red-400 w-2/3 flex m-auto p-4 items-center justify-center"
    >
      <form v-if="showForm" @submit.prevent="createNewEssay" class="mb-4">
        <div class="mb-4">
          <label for="title" class="block mb-2">Title:</label>
          <input
            type="text"
            id="title"
            v-model="newEssay.title"
            class="w-full p-2 border rounded"
            required
          />
        </div>
        <div class="mb-4">
          <label for="description" class="block mb-2">content:</label>
          <textarea
            id="description"
            v-model="newEssay.content"
            class="w-full p-2 border rounded"
            required
          ></textarea>
        </div>
        <div class="mb-4">
          <label for="userId" class="block mb-2">Recipient's DID:</label>
          <input
            type="text"
            id="userId"
            v-model="newEssay.recipientDID"
            class="w-full p-2 border rounded"
            required
          />
        </div>
        <button type="submit" class="btn btn-primary">Create</button>
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
import protocolDefinition from "assets/shared-todo-protocol.json";

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

    console.log(record, "+++");

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
  background-color: #3490dc;
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
  background-color: #2779bd;
}
</style>

<!-- /_essayId.vue -->
<template>
  <div class="container mx-auto p-4 w-1/3">
    <header class="flex-col text-center mb-4">
      <h1 class="text-2xl font-bold mb-4">Shared Essay</h1>
      <p>Share essay drafts with your friends.</p>
      <hr class="my-5" />

      <div v-if="!fetchingListInfo">
        <h2 class="text-xl font-bold">{{ essay.title }}</h2>
        <p>{{ essay.content }}</p>
        <br />
        <p>{{ essay.author?.substr(0, 22) }}...</p>
        <p>{{ essay.recipient?.substr(0, 22) }}...</p>
      </div>
      <div v-else>
        <p>Fetching essay...</p>
      </div>
    </header>
    <div>
      <!-- Add Comment Form -->
      <div class="mt-16 mb-5">
        <form class="flex space-x-4" @submit.prevent="addComment">
          <div class="border-b border-gray-200 sm:w-full">
            <label for="add-todo" class="sr-only">Add a comment</label>
            <textarea
              rows="1"
              name="add-todo"
              id="add-todo"
              v-model="newCommentContent"
              @keydown.enter.exact.prevent="addComment"
              class="block border-0 border-transparent focus:ring-0 p-0 pb-2 resize-none sm:text-lg w-96"
              placeholder="Add a comment"
            />
          </div>
          <button
            type="submit"
            class="bg-indigo-600 border border-transparent focus:outline-none focus:ring-2 focus:ring-indigo-500 focus:ring-offset-2 hover:bg-indigo-700 inline-flex items-center p-1 rounded-full shadow-sm text-white"
          >
            <PlusIconMini class="h-5 w-5" aria-hidden="true" />
          </button>
        </form>
      </div>
      <h2 v-if="!fetchingListInfo" class="text-xl font-bold mb-2">Comments</h2>
      <ul>
        <li
          v-for="(item, index) in commentItems"
          :key="index"
          class="flex items-center mb-2 p-4 border rounded"
        >
          <div class="font-light ml-3 text-gray-500 text-xl">
            {{ item.data.content }}
            <p class="text-gray-400 text-sm">
              Added by: {{ item.data.author.substr(0, 22) }}...
            </p>
          </div>
          <div class="ml-auto">
            <!-- <div @click="deleteTodo(item)" class="cursor-pointer" v-show="myDID == item.data.author">
                            <TrashIcon class="h-8 text-gray-200 w-8" :class="'text-red-500'" />
                        </div> -->
            <div v-show="myDID != item.data.author && item.data.completed">
              <CheckCircleIcon
                class="h-8 text-gray-200 w-8"
                :class="{ 'text-green-500': item.data.completed }"
              />
            </div>
          </div>
        </li>
      </ul>
    </div>
    <div class="mt-4">
      <nuxt-link to="/" class="text-blue-500">Back to Essay Lists</nuxt-link>
    </div>
  </div>
</template>

<script setup>
import { ref } from "vue";
import { useRoute } from "vue-router";
import { PlusIcon as PlusIconMini } from "@heroicons/vue/solid";
import { CheckCircleIcon, TrashIcon } from "@heroicons/vue/outline";
import protocolDefinition from "assets/shared-todo-protocol.json";

const route = useRoute();
const essayId = ref(route.params.essayId);

const { $web5: web5, $myDID: myDID } = useNuxtApp();

let commentRecipient;

let essay = ref({});

let commentItems = ref([]);
const fetchingListInfo = ref(true);

// Adding comments
const newCommentContent = ref("");

// get all the collaborators in the essay(esssayRecipient or commentRecipient??confused. was todoRecipient)
const getCommentRecipient = () => {
  if (myDID === essay.author) {
    return essay.recipient;
  } else {
    return essay.author;
  }
};

onBeforeMount(async () => {
  console.log("this is your DID", myDID);
  console.log(essayId.value);

  // fetch shared list details.
  const { record } = await web5.dwn.records.read({
    message: {
      filter: {
        recordId: essayId.value,
      },
    },
  });

  essay = await record.data.json();

  // fetch comments under list.
  const { records: comments } = await web5.dwn.records.query({
    message: {
      filter: {
        parentId: essayId.value,
      },
    },
  });

  commentRecipient = await getCommentRecipient();

  // Add entry to comments array
  for (let record of comments) {
    const data = await record.data.json();
    const comment = { record, data, id: record.id };
    commentItems.value.push(comment);
    console.log(commentItems, "&&");
  }

  fetchingListInfo.value = false;
});

async function addComment() {
  const commentData = {
    content: newCommentContent.value,
    author: myDID,
    parentId: essayId.value,
  };

  newCommentContent.value = "";

  console.log(commentData);

  const { record: commentRecord, status: createStatus } =
    await web5.dwn.records.create({
      data: commentData,
      message: {
        protocol: protocolDefinition.protocol,
        protocolPath: "essay/comment",
        schema: protocolDefinition.types.comment.schema,
        dataFormat: protocolDefinition.types.comment.dataFormats[0],
        parentId: essayId.value,
        contextId: essayId.value,
      },
    });

  const data = await commentRecord.data.json();
  const comment = { commentRecord, data, id: commentRecord.id };
  commentItems.value.push(comment);

  const { status: sendStatus } = await commentItems.send(commentRecipient);

  if (sendStatus.code !== 202) {
    console.log("Unable to send to target did:" + sendStatus);
    return;
  } else {
    console.log("Sent todo to recipient");
  }
}
</script>

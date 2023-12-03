<!-- /_essayId.vue -->
<template>
  <div class="container mx-auto p-4 w-1/2 pb-36">
    <!-- <header class="flex-col text-center mb-4">
      <h1 class="text-2xl font-bold mb-4">Shared Essay</h1>
      <p>Share essay drafts with your friends.</p>
      <hr class="my-5" />
    </header> -->

    <header class="flex justify-between mb-4">
      <h1 class="text-4xl font-extrabold mb-4 text-orange-500">Atlas</h1>
      <p v-if="myDID" class="cursor-pointer" id="copy-did" @click="copyDID">
        <!-- Copy your DID -->
        {{ myDID.substr(0, 15) }}...
      </p>
    </header>
    <hr class="my-5" />

    <div v-if="!fetchingListInfo" class="text-left pt-16">
      <div class="pb-8">
        <h2 class="text-5xl font-bold mb-4">{{ essay.title }}</h2>
        <!-- <p class="text-lg">{{ essay.content }}</p> -->
        <div
          v-for="(paragraph, index) in essay.content.split('\n')"
          :key="index"
          class="mb-4 text-lg hover:cursor"
          @click="showCommentInput(index)"
          :style="{ cursor: getParagraphCursor(index) }"
        >
          <p>
            {{ paragraph }}
          </p>
          <div v-if="isInputVisible(index)" class="flex flex-row-reverse">
            <div
              class="mt-1 border-2 border-green-600 rounded-full p-2 w-fit py-3"
            >
              <input
                autofocus
                type="text"
                placeholder="Add comment..."
                class="outline-none font-medium px-2"
                @blur="hideCommentInput"
                @keydown.enter="saveComment(index, $event)"
              />
            </div>
          </div>
          <!-- <div v-if="commentsByParagraph[index]"> -->
          <div v-if="getCommentsForParagraph(index).length" class="flex">
            <!-- <div
                v-for="(comment, commentIndex) in commentsByParagraph[index]"
                :key="commentIndex"
                class="w-fit p-2 rounded-full bg-gray-900 text-white px-6 shadow-2xl"
              > -->
            <div
              v-for="(comment, commentIndex) in getCommentsForParagraph(index)"
              :key="commentIndex"
              class="w-fit p-2 rounded-xl border border-green-700 px-4 shadow-md m-1"
            >
              <p class="font-light">{{ comment.data.content }}</p>
              <p class="text-gray-400 text-sm">
                Added by: {{ comment.data.author.substr(0, 12) }}...
              </p>
            </div>
          </div>
        </div>
      </div>
      <br />
      <p><b>Written by: </b> {{ essay.author?.substr(0, 22) }}...</p>
      <p><b>Shared with: </b>{{ essay.recipient?.substr(0, 22) }}...</p>

      <hr />
    </div>
    <div v-else>
      <p>Fetching essay...</p>
    </div>
    <div></div>
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

const showAddCommentInput = ref(null);
const commentsByParagraph = ref({});

// let comments = ref([]);

const showCommentInput = (index) => {
  showAddCommentInput.value = index;
};

const hideCommentInput = () => {
  showAddCommentInput.value = null;
};

async function saveComment(index, event) {
  const commentData = {
    content: event.target.value,
    paragraphIndex: index,
    author: myDID,
    parentId: essayId.value,
  };

  newCommentContent.value = "";

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

  console.log(commentItems, "########@@@@@@@@@@@@@@@@@");

  const { status: sendStatus } = await commentRecord.send(commentRecipient);

  if (sendStatus.code !== 202) {
    console.log("Unable to send to target did:" + sendStatus);
    return;
  } else {
    console.log("Sent todo to recipient");
  }

  hideCommentInput();
}

const isHovered = (index) => {
  return showAddCommentInput.value !== index;
};

const getParagraphCursor = (index) =>
  isInputVisible(index) ? "auto" : "pointer";

const isInputVisible = (index) => showAddCommentInput.value === index;

const getCommentsForParagraph = (paragraphIndex) => {
  const a = commentItems.value.filter(
    (comment) => comment.data.paragraphIndex == paragraphIndex
  );

  return a;
};

// async function addComment() {
//   const commentData = {
//     content: newCommentContent.value,
//     author: myDID,
//     parentId: essayId.value,
//   };

//   newCommentContent.value = "";

//   console.log(commentData);

//   const { record: commentRecord, status: createStatus } =
//     await web5.dwn.records.create({
//       data: commentData,
//       message: {
//         protocol: protocolDefinition.protocol,
//         protocolPath: "essay/comment",
//         schema: protocolDefinition.types.comment.schema,
//         dataFormat: protocolDefinition.types.comment.dataFormats[0],
//         parentId: essayId.value,
//         contextId: essayId.value,
//       },
//     });

//   const data = await commentRecord.data.json();
//   const comment = { commentRecord, data, id: commentRecord.id };
//   commentItems.value.push(comment);

//   const { status: sendStatus } = await commentRecord.send(commentRecipient);

//   if (sendStatus.code !== 202) {
//     console.log("Unable to send to target did:" + sendStatus);
//     return;
//   } else {
//     console.log("Sent todo to recipient");
//   }
// }
</script>

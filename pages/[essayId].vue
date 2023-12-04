<!-- /_essayId.vue -->
<template>
  <div class="container mx-auto p-4 w-1/2 pb-36">
    <!-- <header class="flex-col text-center mb-4">
      <h1 class="text-2xl font-bold mb-4">Shared Essay</h1>
      <p>Share essay drafts with your friends.</p>
      <hr class="my-5" />
    </header> -->

    <header class="flex justify-between mb-4">
      <router-link to="/"
        ><h1 class="text-4xl font-extrabold mb-4 text-orange-500">
          Atlas
        </h1></router-link
      >
      <p v-if="myDID" class="cursor-pointer" id="copy-did" @click="copyDID">
        <!-- Copy your DID -->
        {{ myDID.substr(0, 15) }}...
      </p>
    </header>
    <hr class="my-5" />

    <div class="flex flex-row-reverse">
      <p
        @click="isEditing = true"
        v-if="essay.author == myDID && !isEditing"
        class="px-3 hover:bg-gray-300 w-fit rounded cursor-pointer"
      >
        edit
      </p>
    </div>

    <div v-if="!fetchingListInfo" class="text-left">
      <div v-if="!isEditing" class="pb-8">
        <h2 class="text-5xl font-bold mb-4">
          {{ essay.title }}
        </h2>
        <div
          v-for="(paragraph, index) in essay.content.split('\n')"
          :key="index"
          class="mb-4 text-lg hover:cursor"
          @click="handleParagraphClick(index)"
          :style="{ cursor: getParagraphCursor(index) }"
        >
          <p>
            {{ paragraph }}
          </p>

          <div
            v-if="getCommentsForParagraph(index).length"
            class="flex flex-col"
          >
            <div
              v-for="(comment, commentIndex) in getCommentsForParagraph(index)"
              :key="commentIndex"
              class="flex flex-row-reverse p-2 py-0 rounded-xl w-full px-4 m-1 items-center"
            >
              <div
                class="w-fit border border-green-700 flex flex-row items-center rounded-md p-2 py-0"
              >
                <svg
                  xmlns="http://www.w3.org/2000/svg"
                  fill="none"
                  viewBox="0 0 24 24"
                  stroke-width="1.5"
                  stroke="currentColor"
                  class="w-6 h-6 mr-3"
                >
                  <path
                    stroke-linecap="round"
                    stroke-linejoin="round"
                    d="M12 20.25c4.97 0 9-3.694 9-8.25s-4.03-8.25-9-8.25S3 7.444 3 12c0 2.104.859 4.023 2.273 5.48.432.447.74 1.04.586 1.641a4.483 4.483 0 01-.923 1.785A5.969 5.969 0 006 21c1.282 0 2.47-.402 3.445-1.087.81.22 1.668.337 2.555.337z"
                  />
                </svg>
                <div>
                  <p class="font-small">{{ comment.data.content }}</p>
                  <p class="text-gray-400 text-sm">
                    Added by: {{ comment.data.author.substr(0, 12) }}...
                  </p>
                </div>
              </div>
            </div>
          </div>
          <div v-if="isInputVisible(index)" class="flex flex-row-reverse">
            <div
              class="mt-1 border-2 border-green-600 rounded-lg p-2 w-fit py-3 shadow-lg"
            >
              <input
                autofocus
                type="text"
                placeholder="Add a comment..."
                class="outline-none font-medium px-2"
                @keydown.enter="saveComment(index, $event.target.value)"
                ref="commentInput"
              />
              <div class="flex flex-row-reverse">
                <svg
                  xmlns="http://www.w3.org/2000/svg"
                  fill="none"
                  viewBox="0 0 24 24"
                  stroke-width="1.5"
                  stroke="currentColor"
                  class="w-6 h-6 cursor-pointer"
                  @click.stop="saveComment(index, $refs.commentInput[0].value)"
                >
                  <path
                    stroke-linecap="round"
                    stroke-linejoin="round"
                    d="M6 12L3.269 3.126A59.768 59.768 0 0121.485 12 59.77 59.77 0 013.27 20.876L5.999 12zm0 0h7.5"
                  />
                </svg>
              </div>
            </div>
          </div>
        </div>
      </div>

      <div v-if="isEditing" class="flex m-auto p-4">
        <form @submit.prevent="updateEssay" class="mb-4 w-full">
          <div class="mb-4">
            <!-- <label for="title" class="block mb-2">Title:</label> -->
            <input
              v-model="editedEssayTitle"
              type="text"
              id="title"
              placeholder="Title"
              class="w-full p-2 rounded outline-none text-4xl font-bold"
              required
            />
          </div>
          <div class="mb-4">
            <textarea
              v-model="editedEssayContent"
              id="description"
              placeholder="write something amazing..."
              rows="10"
              class="w-full p-2 px-4 outline-none text-xl overflow-hidden border rounded-md"
              required
            ></textarea>
          </div>
          <button type="submit" class="btn btn-primary">Save</button>
          <button
            type="button"
            class="btn ml-4 btn-secondary"
            @click="isEditing = false"
          >
            Cancel
          </button>
        </form>
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
import protocolDefinition from "assets/shared-essays-protocol.json";

const route = useRoute();
const essayId = ref(route.params.essayId);

const { $web5: web5, $myDID: myDID } = useNuxtApp();

let commentRecipient;

let essay = ref({});

let editedEssayContent = ref("");
let editedEssayTitle = ref("");

let commentItems = ref([]);
const fetchingListInfo = ref(true);

let isEditing = ref(false);

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

  editedEssayTitle = essay.title;
  editedEssayContent = essay.content;

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

const handleParagraphClick = (index) => {
  console.log("pa click");
  if (showAddCommentInput.value == index) showCommentInput.value = null;
  else {
    showCommentInput(index);
  }
};

async function saveComment(index, commentValue) {
  hideCommentInput();

  console.log(commentValue, "commentVale");
  const commentData = {
    content: commentValue,
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

const updateEssay = async () => {
  try {
    const { record } = await web5.dwn.records.read({
      message: {
        filter: {
          recordId: essayId.value,
        },
      },
    });

    const { status } = await record.update({
      data: {
        title: editedEssayTitle,
        content: editedEssayContent,
        author: myDID,
        recipient: essay.recipient,
      },
    });

    console.log(status, "########################+++++++++++++++++++++++=");

    const { record: editedRecord } = await web5.dwn.records.read({
      message: {
        filter: {
          recordId: essayId.value,
        },
      },
    });

    //
    isEditing.value = false;

    essay.title = editedEssayTitle;
    essay.content = editedEssayContent;

    const { status: sendStatus } = await editedRecord.send(essay.recipient);

    if (sendStatus.code !== 202) {
      console.log("Unable to send to target did:" + sendStatus);
      return;
    } else {
      console.log("Shared list sent to recipient");
    }

    console.log("here..........", isEditing);
  } catch (e) {
    console.error(e);
    return;
  }
};

onMounted(() => {
  document.title = "Atlas";
});
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

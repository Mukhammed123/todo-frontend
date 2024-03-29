<template>
  <div class="app-container">
    <aside
      v-if="isLoggedIn && route.path !== '/login' && route.path !== '/register'"
      class="w-64"
      aria-label="Sidebar"
    >
      <div
        class="overflow-y-auto py-4 px-3 bg-gray-50 rounded dark:bg-gray-800"
      >
        <ul class="space-y-2" style="height: 100%; position: relative">
          <li><span class="collaections-title">Collections</span></li>
          <hr />
          <li>
            <router-link
              :to="`/`"
              class="flex items-center p-2 text-base font-normal text-gray-900 rounded-lg dark:text-white hover:bg-gray-100 dark:hover:bg-gray-700"
            >
              <svg
                class="w-6 h-6 text-gray-500 transition duration-75 dark:text-gray-400 group-hover:text-gray-900 dark:group-hover:text-white"
                fill="currentColor"
                viewBox="0 0 20 20"
                xmlns="http://www.w3.org/2000/svg"
              >
                <path
                  d="M21 13v10h-6v-6h-6v6h-6v-10h-3l12-12 12 12h-3zm-1-5.907v-5.093h-3v2.093l3 3z"
                  xmlns="http://www.w3.org/2000/svg"
                  width="20"
                  height="20"
                  viewBox="0 0 20 20"
                />
              </svg>
              <span class="flex-1 ml-3">Home</span>
            </router-link>
          </li>
          <li v-for="(cat, index) in todoCats" :key="`${cat.title}-${index}`">
            <router-link
              :to="`/todo-detail/${cat.id}`"
              class="flex items-center p-2 text-base font-normal text-gray-900 rounded-lg dark:text-white hover:bg-gray-100 dark:hover:bg-gray-700"
            >
              <svg
                class="w-6 h-6 text-gray-500 transition duration-75 dark:text-gray-400 group-hover:text-gray-900 dark:group-hover:text-white"
                fill="currentColor"
                viewBox="0 0 20 20"
                xmlns="http://www.w3.org/2000/svg"
              >
                <path d="M2 10a8 8 0 018-8v8h8a8 8 0 11-16 0z"></path>
                <path d="M12 2.252A8.014 8.014 0 0117.748 8H12V2.252z"></path>
              </svg>
              <span class="flex-1 ml-3">{{ cat.title }}</span>
              <span
                v-if="Number(todosCounter[cat.title]) > 0"
                class="inline-flex justify-center items-center p-3 ml-3 w-3 h-3 text-sm font-medium text-blue-600 bg-blue-200 rounded-full dark:bg-blue-900 dark:text-blue-200"
                >{{ todosCounter[cat.title] }}</span
              >
            </router-link>
          </li>
          <li class="new-collection-li px-2 py-0" style="margin: 0">
            <button class="add-todo-btn" @click="openAddDialog">
              <div>
                <span class="plus-sign">+</span>
                <span class="add-new-todo">New Collection</span>
              </div>
            </button>
          </li>
          <li style="position: absolute; bottom: 0; width: 100%">
            <button class="add-todo-btn" style="width: 100%" @click="logout">
              <div style="display: flex; padding: 8px; align-items: start">
                <svg
                  width="24"
                  height="24"
                  xmlns="http://www.w3.org/2000/svg"
                  fill-rule="evenodd"
                  clip-rule="evenodd"
                >
                  <path
                    fill="rgb(40, 103, 213)"
                    d="M11 21h8v-2l1-1v4h-9v2l-10-3v-18l10-3v2h9v5l-1-1v-3h-8v18zm10.053-9l-3.293-3.293.707-.707 4.5 4.5-4.5 4.5-.707-.707 3.293-3.293h-9.053v-1h9.053z"
                  />
                </svg>
                <span
                  style="margin-left: 3px; padding-top: 2px"
                  class="add-new-todo"
                  >Log out</span
                >
              </div>
            </button>
          </li>
        </ul>
      </div>
      <div
        class="hidden overflow-x-hidden overflow-y-auto fixed inset-0 z-50 outline-none focus:outline-none justify-center items-center"
        id="addCollectionModel"
      >
        <AddEditTaskDialog
          name="Collection"
          :key="`add-edit-${modalKey}`"
          :operation="operation"
          :description="collectionTitle"
          @close-dialog="toggleModal('addCollectionModel')"
          @confirmed-add="addCollection"
        />
      </div>
      <div
        class="hidden opacity-25 fixed inset-0 z-40 bg-black"
        :id="`addCollectionModel-backdrop`"
      ></div>
    </aside>
    <RouterView :key="route.path" />
    <Snackbar
      snackbar-id="appSnackbarError"
      :hide-snackbar="hideSnackbar"
      :snackbar-message="snackbarMessage"
      @close="hideSnackbar = true"
    />
  </div>
</template>

<script>
import { onMounted, ref } from "vue";
import { storeToRefs } from "pinia";
import { useTodoStore } from "@/stores/todo";
import axios from "axios";
import jwt_decode from "jwt-decode";
import { todoPath, checkPath } from "@/services/apiPaths";
import { useRoute, useRouter } from "vue-router";
import AddEditTaskDialog from "@/components/AddEditTaskDialog.vue";
import { updateCounter } from "@/utils/UpdateTodosCounter";
import { getUserData } from "@/utils/GetUsersData";
import Snackbar from "@/components/Snackbar.vue";

export default {
  name: "App",
  components: {
    AddEditTaskDialog,
    Snackbar,
  },
  setup() {
    const todoStore = useTodoStore();
    const { todoCats, isLoggedIn, accessToken, todosCounter } =
      storeToRefs(todoStore);
    const route = useRoute();
    const modalKey = ref(0);
    const operation = ref(undefined);
    const collectionTitle = ref("");
    const hideSnackbar = ref(true);
    const snackbarMessage = ref("");
    const router = useRouter();

    onMounted(async () => {
      const accessTokenLS = localStorage.getItem("accessToken");
      if ((accessTokenLS || "").length > 0) {
        try {
        const response = await axios.post(
          checkPath,
          {},
          {
            headers: {
              "Content-Type": "Application/json",
              Authorization: `Bearer ${accessTokenLS}`,
            },
          }
        );
        if (response.status === 200) {
          todoStore.setIsLoggedIn(true);
          todoStore.setAccessToken(accessTokenLS);
          const decodedData = jwt_decode(accessToken.value);
          const userResponse = await getUserData(
            decodedData.user_id,
            accessToken.value
          );
          if (userResponse) getCollections();
          else logout();
        } else logout();
      } catch {
        logout();
      }
      } else logout();
    });

    const getCollections = async () => {
      try {
        const response = await axios.get(todoPath, {
          headers: {
            "Content-Type": "application/json",
            Authorization: `Bearer ${accessToken.value}`,
          },
        });
        if (response.status === 200) {
          todoStore.setTodoCats(response.data);
          todoCats.value.forEach((el) => {
            updateCounter(el.title, el.num_tasks);
          });
        }
      } catch (err) {
        hideSnackbar.value = false;
        snackbarMessage.value = err.message;
      }
    };

    const openAddDialog = () => {
      operation.value = "add";
      toggleModal("addCollectionModel");
    };
    const addCollection = async (text) => {
      const newCollection = {
        title: text,
      };
      toggleModal("addCollectionModel");
      try {
        await axios.post(todoPath, newCollection, {
          headers: {
            "Content-Type": "application/json",
            Authorization: `Bearer ${accessToken.value}`,
          },
        });
        const response = await axios.get(todoPath, {
          headers: {
            "Content-Type": "application/json",
            Authorization: `Bearer ${accessToken.value}`,
          },
        });
        if (response.status === 200) {
          todoCats.value = response.data;
          todoStore.setTodoCats(response.data);
        }
      } catch (err) {
        hideSnackbar.value = false;
        snackbarMessage.value = err.message;
      }
    };

    const toggleModal = (modalId) => {
      modalKey.value += 1;
      const dialog = document.getElementById(modalId);
      dialog.classList.toggle("hidden");
      document.getElementById(modalId + "-backdrop").classList.toggle("hidden");
      document.getElementById(modalId).classList.toggle("flex");
      document.getElementById(modalId + "-backdrop").classList.toggle("flex");
    };

    const logout = async () => {
      todoStore.setIsLoggedIn(false);
      todoStore.setAccessToken(undefined);
      todoStore.setRefreshToken(undefined);
      localStorage.clear();
      router.push("/login");
    };

    return {
      hideSnackbar,
      snackbarMessage,
      todosCounter,
      isLoggedIn,
      todoCats,
      route,
      modalKey,
      operation,
      collectionTitle,
      openAddDialog,
      addCollection,
      toggleModal,
      logout,
    };
  },
};
</script>

<style>
@import "@/assets/base.css";
</style>

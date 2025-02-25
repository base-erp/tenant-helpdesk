<template>
  <div class="pb-8 text-base text-gray-700">
    <div class="flex flex-col gap-4">
      <TopBar
        title="New ticket"
        :back-to="hideBackButton ? null : { name: CUSTOMER_PORTAL_LANDING }"
      />
      <div
        v-if="!hideAbout && template.data?.about"
        class="prose prose-sm mx-9 max-w-full"
        v-html="sanitize(template.data.about)"
      />
      <div class="mx-9 grid grid-cols-3 gap-4">
        <UniInput
          v-for="field in visibleFields"
          :key="field.fieldname"
          :field="field"
          :value="templateFields[field.fieldname]"
          @change="templateFields[field.fieldname] = $event.value"
        />
      </div>
      <div
        v-if="!isEmpty(articles.data)"
        class="mx-9 flex flex-col gap-4 rounded-lg border bg-yellow-50 p-4"
      >
        <div class="font-medium">
          📚 Did you know? These articles might cover what you looking for!
        </div>
        <RouterLink
          v-for="article in articles.data"
          :key="article.name"
          class="group flex cursor-pointer gap-2 transition hover:text-gray-900"
          :to="getArticleLink(article.name, article.title)"
          target="_blank"
        >
          {{ article.title }}
          <div class="opacity-0 group-hover:opacity-100">&rightarrow;</div>
        </RouterLink>
      </div>
      <div class="mx-9 space-y-2">
        <FormControl
          v-model="subject"
          type="text"
          label="Subject"
          placeholder="A short description"
          @input="(v) => searchArticles(v)"
        />
        <TextEditor
          ref="textEditor"
          placeholder="Detailed explanation"
          :content="description"
          @change="(v) => (description = v)"
        >
          <template #bottom="{ editor }">
            <TextEditorBottom
              v-model:attachments="attachments"
              :editor="editor"
            >
              <template #actions-right>
                <Button
                  label="Create"
                  :disabled="isDisabled"
                  theme="gray"
                  variant="solid"
                  @click="newTicket.submit()"
                />
              </template>
            </TextEditorBottom>
          </template>
        </TextEditor>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onUnmounted, computed, reactive, onMounted } from "vue";
import { RouterLink, useRouter } from "vue-router";
import {
  createResource,
  createListResource,
  Button,
  FormControl,
} from "frappe-ui";
import sanitizeHtml from "sanitize-html";
import { isEmpty } from "lodash";
import {
  CUSTOMER_PORTAL_LANDING,
  CUSTOMER_PORTAL_TICKET,
  KB_PUBLIC_ARTICLE,
} from "@/router";
import { useConfigStore } from "@/stores/config";
import { createToast } from "@/utils/toasts";
import TextEditor from "@/components/text-editor/TextEditor.vue";
import TextEditorBottom from "@/components/text-editor/TextEditorBottom.vue";
import TopBar from "@/components/TopBar.vue";
import UniInput from "@/components/UniInput.vue";

interface P {
  templateId?: string;
  hideBackButton?: boolean;
  hideAbout?: boolean;
  showHiddenFields?: boolean;
}

const props = withDefaults(defineProps<P>(), {
  templateId: "Default",
  hideBackButton: false,
  hideAbout: false,
  showHiddenFields: false,
});
const router = useRouter();
const configStore = useConfigStore();
const textEditor = ref();
const subject = ref("");
const description = ref("");
const attachments = ref([]);
const templateFields = reactive({});

const template = createResource({
  url: "helpdesk.helpdesk.doctype.hd_ticket_template.api.get_one",
  makeParams: () => ({
    name: props.templateId,
  }),
  auto: true,
});

const articles = createListResource({
  doctype: "HD Article",
  fields: ["name", "title"],
  pageLimit: 5,
  debounce: 500,
});

const visibleFields = computed(() =>
  template.data?.fields.filter(
    (f) => props.showHiddenFields || !f.hide_from_customer
  )
);
const newTicket = createResource({
  url: "helpdesk.helpdesk.doctype.hd_ticket.api.new",
  debounce: 300,
  makeParams: () => ({
    doc: {
      description: description.value,
      subject: subject.value,
      template: props.templateId,
      ...templateFields,
    },
    attachments: attachments.value,
  }),
  validate(params) {
    const fields = visibleFields.value.filter((f) => f.required);
    const toVerify = [...fields, "subject", "description"];
    for (const field of toVerify) {
      if (isEmpty(params.doc[field.fieldname || field])) {
        return `${field.label || field} is required`;
      }
    }
  },
  onError(error) {
    const title = error.message ? error.message : error.messages?.join("\n");

    createToast({
      title,
      icon: "x",
      iconClasses: "text-red-500",
    });
  },
  onSuccess(data) {
    router.push({
      name: CUSTOMER_PORTAL_TICKET,
      params: {
        ticketId: data.name,
      },
    });
  },
});

const isDisabled = computed(
  () =>
    newTicket.loading ||
    isEmpty(subject.value) ||
    textEditor.value?.editor.isEmpty
);

function sanitize(html: string) {
  return sanitizeHtml(html, {
    allowedTags: sanitizeHtml.defaults.allowedTags.concat(["img"]),
  });
}

function getArticleLink(name: string, title: string) {
  return {
    name: KB_PUBLIC_ARTICLE,
    params: {
      articleId: name,
      articleTitleSlug: title.toLowerCase().replaceAll(" ", "-"),
    },
  };
}

function searchArticles(term: string) {
  if (term.length < 5) {
    delete articles.data;
    return;
  }

  articles.update({
    filters: {
      title: ["like", `%${term}%`],
    },
  });

  articles.reload();
}

onMounted(() => configStore.setTitle("New ticket"));
onUnmounted(() => configStore.setTitle());
</script>

<template>
  <!--<q-header elevated class="bg-primary text-white">
    <q-toolbar class="toolbar" id="print-select-toolbar">Select elements to print!</q-toolbar>
  </q-header>-->
  <q-page padding>
    <div v-if="!isGalleryEmpty" class="row justify-center q-gutter-sm" id="print-gallery-container">
      <template :key="item.id" v-for="(item, index) in this.mediacollectionStore.collection">
        <div v-if="item.media_type == 'image' || item.media_type == 'collage'">
          <q-intersection once class="preview-item" :id="`preview-item-${index}`">
            <q-card class="q-ma-sm" @click="toggleSelectPic(index)">
              <q-img :src="getImageDetail(index)" loading="eager" no-transition no-spinner :ratio="1" class="rounded-borders" fit="contain">
                <div class="absolute-full text-subtitle2 flex flex-center" v-if="this.itemsSelected.has(index)">
                  <q-icon size="150px" name="o_check" class="text-primary"></q-icon></div
              ></q-img>
            </q-card>
          </q-intersection>
        </div>
      </template>
    </div>
    <div v-else v-html="uiSettingsStore.uiSettings.GALLERY_EMPTY_MSG"></div>

    <q-dialog v-model="confirmPrintMult">
      <q-card class="q-pa-sm" style="min-width: 350px" id="gallery-confirm-print-dialog">
        <q-card-section class="row items-center">
          <q-avatar icon="delete" color="primary" text-color="white" />
          <span class="q-ml-sm">{{ $t("MSG_CONFIRM_PRINT_IMAGE", { count: numItemsSelected }) }}</span>
        </q-card-section>

        <q-card-actions align="right">
          <q-btn flat :label="$t('BTN_LABEL_CANCEL')" v-close-popup />
          <q-btn
            :label="$t('BTN_LABEL_PRINT_IMAGE')"
            color="primary"
            v-close-popup
            to="/"
            @click="
              confirmPrint();
              $emit('closeEvent');
            "
          />
        </q-card-actions>
      </q-card>
    </q-dialog>

    <q-page-sticky position="bottom-right" :offset="[25, 25]" v-if="numItemsSelected > 0">
      <div class="q-gutter-sm">
        <q-btn color="primary" rounded no-caps @click="confirmPrintMult = true" class="action-button" id="gallery-confirm-print">
          <q-icon left name="print" />
          <div>{{ $t("BTN_LABEL_CONFIRM_PRINT", { count: numItemsSelected }) }}</div>
        </q-btn>
      </div>
    </q-page-sticky>
  </q-page>
</template>
<style lang="sass" scoped>
.preview-item
  height: 400px
  width: 400px
</style>

<script>
import { useMainStore } from "../stores/main-store.js";
import { useUiSettingsStore } from "../stores/ui-settings-store.js";
import { useMediacollectionStore } from "../stores/mediacollection-store.js";
import { ref } from "vue";
import { scroll } from "quasar";

export default {
  // name: 'PageName',
  components: {},
  setup() {
    const store = useMainStore();
    const uiSettingsStore = useUiSettingsStore();
    const mediacollectionStore = useMediacollectionStore();
    const itemsSelected = new Set();
    const { getScrollTarget, setVerticalScrollPosition } = scroll;

    return {
      // you can return the whole store instance to use it in the template
      store,
      uiSettingsStore,
      mediacollectionStore,
      itemsSelected,
      getScrollTarget,
      setVerticalScrollPosition,
      numItemsSelected: ref(0),
      confirmPrintMult: ref(false),
    };
  },
  computed: {
    itemId() {
      console.log("ID is ", this.$route.params);
      this.itemsSelected.add(parseInt(this.$route.params.selectedItemId));
      return this.$route.params.selectedItemId;
    },

    isGalleryEmpty() {
      return this.mediacollectionStore.collection_number_of_items == 0;
    },

    numSelected() {
      return this.itemsSelected.size;
    },
  },
  mounted() {
    let initialIndex = parseInt(this.$route.params.selectedItemId);

    if (initialIndex != undefined) {
      // auto-select initial item
      this.toggleSelectPic(initialIndex);

      // auto-scroll to it
      let domElem = document.getElementById(`preview-item-${initialIndex}`);
      if (domElem) {
        console.log("Scroll to: ", domElem);
        const scrollTarget = this.getScrollTarget(domElem);
        const offset = domElem.offsetTop;
        const duration = 1000;
        this.setVerticalScrollPosition(scrollTarget, offset, duration);
      }
    }
  },
  methods: {
    getImageDetail(index, detail = "thumbnail") {
      return this.mediacollectionStore.collection[index][detail];
    },

    toggleSelectPic(index) {
      index = parseInt(index);
      this.itemsSelected.has(index) ? this.itemsSelected.delete(index) : this.itemsSelected.add(index);

      this.numItemsSelected = this.itemsSelected.size;
    },
    confirmPrint() {
      let printIds = [];
      for (let printIndex of this.itemsSelected) {
        printIds.push(this.mediacollectionStore.collection[printIndex].id);
      }
      let commaSeparatedIds = printIds.join(",");
      console.log("Printing elements: ", commaSeparatedIds);

      this.$api
        .get(`/print/item/${commaSeparatedIds}`)
        .then((response) => {
          console.log(response);
          this.$q.notify({
            message: "Started printing...",
            type: "positive",
            spinner: true,
            //timeout: 2000
          });
        })
        .catch((error) => {
          if (error.response) {
            // The request was made and the server responded with a status code
            // that falls out of the range of 2xx
            console.log(error.response);

            if (error.response.status == 425) {
              this.$q.notify({
                message: error.response.data["detail"],
                caption: "Print Service",
                type: "info",
              });
            } else {
              this.$q.notify({
                message: error.response.data["detail"],
                caption: "Print Service",
                type: "negative",
              });
            }
          } else if (error.request) {
            // The request was made but no response was received
            // `error.request` is an instance of XMLHttpRequest in the browser and an instance of
            // http.ClientRequest in node.js
            console.error(error.request);
          } else {
            // Something happened in setting up the request that triggered an Error
            console.error("Error", error.message);
          }
          //console.error(error.config);
        });
    },
  },
};
</script>

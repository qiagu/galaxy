<template>
    <div>
        <div id="path-bar" />

        <div class="form-inline d-flex align-items-center mb-2">
            <a class="mr-1 btn btn-secondary" :href="getHomeUrl" data-toggle="tooltip" title="Go to first page">
                <font-awesome-icon icon="home" />
            </a>
            <div>
                <form class="form-inline">
                    <b-input-group size="sm">
                        <b-form-input
                            class="mr-1"
                            v-on:input="updateSearch($event)"
                            type="search"
                            id="filterInput"
                            placeholder="Search"
                        />
                    </b-input-group>
                    <button
                        v-if="metadata.can_add_library_item"
                        title="Create new folder"
                        class="btn btn-secondary toolbtn-create-folder add-library-items add-library-items-folder mr-1"
                        type="button"
                        @click="newFolder"
                    >
                        <font-awesome-icon icon="plus" />
                        Folder
                    </button>
                    <div v-if="metadata.can_add_library_item">
                        <div
                            v-if="multiple_add_dataset_options"
                            title="Add datasets to current folder"
                            class="dropdown add-library-items add-library-items-datasets mr-1"
                        >
                            <button type="button" class="btn btn-secondary dropdown-toggle" data-toggle="dropdown">
                                <span class="fa fa-plus"></span> Datasets <span class="caret" />
                            </button>
                            <div class="dropdown-menu">
                                <a class="dropdown-item cursor-pointer" @click="addDatasets('history')">
                                    from History</a
                                >
                                <a
                                    v-if="user_library_import_dir"
                                    class="dropdown-item cursor-pointer"
                                    @click="addDatasets('userdir')"
                                >
                                    from User Directory
                                </a>
                                <div v-if="library_import_dir || allow_library_path_paste">
                                    <h5 class="dropdown-header cursor-pointer">Admins only</h5>
                                    <a
                                        v-if="library_import_dir"
                                        class="dropdown-item cursor-pointer"
                                        @click="addDatasets('importdir')"
                                    >
                                        from Import Directory
                                    </a>
                                    <a
                                        v-if="allow_library_path_paste"
                                        class="dropdown-item cursor-pointer"
                                        @click="addDatasets('path')"
                                    >
                                        from Path
                                    </a>
                                </div>
                            </div>
                        </div>
                    </div>
                    <div class="dropdown mr-1">
                        <button
                            type="button"
                            class="primary-button dropdown-toggle add-to-history"
                            data-toggle="dropdown"
                        >
                            <font-awesome-icon icon="book" />
                            Export to History <span class="caret"></span>
                        </button>
                        <div class="dropdown-menu" role="menu">
                            <a
                                href="javascript:void(0)"
                                role="button"
                                class="toolbtn-bulk-import add-to-history-datasets dropdown-item"
                                @click="importToHistoryModal(false)"
                            >
                                as Datasets
                            </a>
                            <a
                                href="javascript:void(0)"
                                role="button"
                                class="toolbtn-collection-import add-to-history-collection dropdown-item"
                                @click="importToHistoryModal(true)"
                            >
                                as a Collection
                            </a>
                        </div>
                    </div>
                    <div
                        title="Download items as archive"
                        class="dropdown dataset-manipulation mr-1"
                        v-if="dataset_manipulation"
                    >
                        <button
                            type="button"
                            id="download-dropdown-btn"
                            class="primary-button dropdown-toggle"
                            data-toggle="dropdown"
                        >
                            <font-awesome-icon icon="download" />
                            Download <span class="caret"></span>
                        </button>
                        <div class="dropdown-menu" role="menu">
                            <a class="dropdown-item cursor-pointer" @click="downloadData('tgz')">.tar.gz</a>
                            <a class="dropdown-item cursor-pointer" @click="downloadData('tbz')">.tar.bz</a>
                            <a class="dropdown-item cursor-pointer" @click="downloadData('zip')">.zip</a>
                        </div>
                    </div>
                    <button
                        v-if="logged_dataset_manipulation"
                        data-toggle="tooltip"
                        title="Mark items deleted"
                        class="primary-button toolbtn-bulk-delete logged-dataset-manipulation mr-1"
                        type="button"
                        @click="deleteSelected"
                    >
                        <font-awesome-icon icon="trash" />
                        Delete
                    </button>
                    <span class="mr-1" data-toggle="tooltip" title="Show location details">
                        <button @click="showDetails" class="primary-button toolbtn-show-locinfo" type="button">
                            <font-awesome-icon icon="info-circle" />
                            Details
                        </button>
                    </span>
                    <div class="form-check logged-dataset-manipulation mr-1" v-if="logged_dataset_manipulation">
                        <b-form-checkbox
                            id="checkbox-1"
                            :checked="include_deleted"
                            @input="toggle_include_deleted($event)"
                            name="checkbox-1"
                        >
                            include deleted
                        </b-form-checkbox>
                    </div>
                </form>
            </div>
        </div>
    </div>
</template>
<script>
import BootstrapVue from "bootstrap-vue";
import { getGalaxyInstance } from "app";
import Vue from "vue";
import { FontAwesomeIcon } from "@fortawesome/vue-fontawesome";
import { showLocInfo } from "./details-modal";
import { deleteSelectedItems } from "./delete-selected";
import { initTopBarIcons } from "components/LibraryFolder/icons";
import mod_path_bar from "components/LibraryFolder/path-bar";
import mod_import_dataset from "./import-to-history/import-dataset";
import mod_import_collection from "./import-to-history/import-collection";
import mod_add_datasets from "./add-datasets";
import { Toast } from "ui/toast";
import download from "./download";
import mod_utils from "utils/utils";
import { getAppRoot } from "onload/loadConfig";

initTopBarIcons();

Vue.use(BootstrapVue);

export default {
    name: "FolderTopBar",
    props: {
        folder_id: {
            type: String,
            required: true,
        },
        include_deleted: {
            type: Boolean,
            required: true,
        },
        selected: {
            type: Array,
            required: true,
        },
        metadata: {
            type: Object,
            required: true,
        },
        folderContents: {
            type: Array,
            required: true,
        },
    },
    components: {
        FontAwesomeIcon,
    },
    data() {
        return {
            dataset_manipulation: false,
            logged_dataset_manipulation: false,
            is_admin: false,
            multiple_add_dataset_options: false,
            user_library_import_dir: false,
            library_import_dir: false,
            allow_library_path_paste: false,
            list_genomes: [],
            list_extensions: [],
            // datatype placeholder for extension auto-detection
            auto: {
                id: "auto",
                text: "Auto-detect",
                description: `This system will try to detect the file type automatically.
                     If your file is not detected properly as one of the known formats,
                     it most likely means that it has some format problems (e.g., different
                     number of columns on different rows). You can still coerce the system
                     to set your data to the format you think it should be.
                     You can also upload compressed files, which will automatically be decompressed`,
            },
        };
    },
    created() {
        const Galaxy = getGalaxyInstance();
        this.is_admin = Galaxy.user.attributes.is_admin;
        this.user_library_import_dir = Galaxy.config.user_library_import_dir;
        this.library_import_dir = Galaxy.config.library_import_dir;
        this.allow_library_path_paste = Galaxy.config.allow_library_path_paste;
        if (
            this.user_library_import_dir !== null ||
            this.allow_library_path_paste !== false ||
            this.library_import_dir !== null
        ) {
            this.multiple_add_dataset_options = true;
        }
        const contains_file_or_folder = this.folderContents.find((el) => el.type === "folder" || el.type === "file");

        // logic from legacy code
        if (contains_file_or_folder) {
            if (Galaxy.user) {
                this.dataset_manipulation = true;
                if (!Galaxy.user.isAnonymous()) {
                    this.logged_dataset_manipulation = true;
                }
            }
        }
        this.fetchExtAndGenomes();
    },
    mounted() {
        new mod_path_bar.PathBar({
            full_path: this.metadata.full_path,
            id: this.folder_id,
            parent_library_id: this.metadata.parent_library_id,
        });
    },
    computed: {
        getHomeUrl: () => {
            return `${getAppRoot()}library/list`;
        },
        allDatasets: function () {
            return this.folderContents.filter((element) => element.type === "file");
        },
    },
    methods: {
        updateSearch: function (value) {
            this.$emit("updateSearch", value);
        },
        deleteSelected: function () {
            deleteSelectedItems(
                this.selected,
                (deletedItem) => this.$emit("deleteFromTable", deletedItem),
                () => this.$emit("refreshTable")
            );
        },
        newFolder() {
            this.folderContents.unshift({
                editMode: true,
                isNewFolder: true,
                type: "folder",
                name: "",
                description: "",
            });
            this.$emit("refreshTable");
        },
        downloadData(format) {
            const { datasets, folders } = this.findCheckedItems();
            if (this.selected.length === 0) {
                Toast.info("You must select at least one dataset to download");
                return;
            }

            download(format, datasets, folders);
        },
        addDatasets(source) {
            new mod_add_datasets.AddDatasets({
                source: source,
                id: this.folder_id,
                updateContent: this.updateContent,
                list_genomes: this.list_genomes,
                list_extensions: this.list_extensions,
            });
        },
        // helper function to make legacy code compatible
        findCheckedItems: function (idOnly = true) {
            const datasets = [];
            const folder = [];
            this.selected.forEach((item) => {
                item.type === "file" ? datasets.push(idOnly ? item.id : item) : idOnly ? item.id : item;
            });
            return { datasets: datasets, folders: folder };
        },
        importToHistoryModal: function (isCollection) {
            const { datasets, folders } = this.findCheckedItems(!isCollection);
            const checkedItems = this.selected;
            checkedItems.dataset_ids = datasets;
            checkedItems.folder_ids = folders;
            if (isCollection) {
                new mod_import_collection.ImportCollectionModal({
                    selected: checkedItems,
                    allDatasets: this.allDatasets,
                });
            } else {
                new mod_import_dataset.ImportDatasetModal({
                    selected: checkedItems,
                });
            }
        },
        /*
            Slightly adopted Bootstrap code
             */
        /**
         * Request all extensions and genomes from Galaxy
         * and save them in sorted arrays.
         */
        fetchExtAndGenomes: function () {
            mod_utils.get({
                url: `${getAppRoot()}api/datatypes?extension_only=False`,
                success: (datatypes) => {
                    this.list_extensions = [];
                    for (const key in datatypes) {
                        this.list_extensions.push({
                            id: datatypes[key].extension,
                            text: datatypes[key].extension,
                            description: datatypes[key].description,
                            description_url: datatypes[key].description_url,
                        });
                    }
                    this.list_extensions.sort((a, b) => (a.id > b.id ? 1 : a.id < b.id ? -1 : 0));
                    this.list_extensions.unshift(this.auto);
                },
                cache: true,
            });
            mod_utils.get({
                url: `${getAppRoot()}api/genomes`,
                success: (genomes) => {
                    this.list_genomes = [];
                    for (const key in genomes) {
                        this.list_genomes.push({
                            id: genomes[key][1],
                            text: genomes[key][0],
                        });
                    }
                    this.list_genomes.sort((a, b) => (a.id > b.id ? 1 : a.id < b.id ? -1 : 0));
                },
                cache: true,
            });
        },
        showDetails() {
            showLocInfo(Object.assign({ id: this.folder_id }, this.metadata));
        },
        toggle_include_deleted: function (value) {
            this.$emit("fetchFolderContents", value);
        },
        updateContent: function () {
            this.$emit("fetchFolderContents");
        },
    },
};
</script>

<style scoped>
@import "./pointer.css";
</style>

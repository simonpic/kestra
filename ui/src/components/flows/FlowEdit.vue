<template>
    <div>
        <flow-editor @save="save" v-model="content" lang="yaml" @update:model-value="onChange($event)" />
        <bottom-line v-if="canSave || canDelete || canExecute">
            <ul>
                <li>
                    <el-button :icon="icon.Delete" type="danger" v-if="canDelete" @click="deleteFile">
                        {{ $t('delete') }}
                    </el-button>
                </li>

                <li>
                    <router-link v-if="flow" :to="{name: 'flows/create', query: {copy: true}}">
                        <el-button :icon="icon.ContentCopy">
                            {{ $t('copy') }}
                        </el-button>
                    </router-link>
                </li>

                <li>
                    <trigger-flow v-if="flow && canExecute" :disabled="flow.disabled" :flow-id="flow.id" :namespace="flow.namespace" />
                </li>

                <li>
                    <el-button :icon="icon.ContentSave" @click="save" v-if="canSave" type="primary">
                        {{ $t('save') }}
                    </el-button>
                </li>
            </ul>
        </bottom-line>
    </div>
</template>

<script>
    import flowTemplateEdit from "../../mixins/flowTemplateEdit";
    import {mapGetters} from "vuex";
    import TriggerFlow from "./TriggerFlow.vue"
    import ContentCopy from "vue-material-design-icons/ContentCopy.vue";
    import ContentSave from "vue-material-design-icons/ContentSave.vue";
    import Delete from "vue-material-design-icons/Delete.vue";
    import {shallowRef} from "vue";

    export default {
        components: {
            TriggerFlow,
        },
        mixins: [flowTemplateEdit],
        data() {
            return {
                dataType: "flow",
                icon: {
                    ContentCopy: shallowRef(ContentCopy),
                    ContentSave: shallowRef(ContentSave),
                    Delete: shallowRef(Delete),
                }
            };
        },
        computed: {
            ...mapGetters("flow", ["flow"]),
        },
        created() {
            this.loadFile();
        },
        beforeUnmount() {
        },
    };
</script>

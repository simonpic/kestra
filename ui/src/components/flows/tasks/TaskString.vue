<template>
    <editor
        :model-value="editorValue"
        :navbar="false"
        :full-height="false"
        :input="true"
        lang="text"
        @update:model-value="onInput"
    />
</template>
<script>
    import Task from "./Task";
    import Editor from "../../../components/inputs/Editor.vue";

    export default {
        mixins: [Task],
        components: {Editor},
        computed: {
            isValid() {
                if (this.required && !this.modelValue) {
                    return false;
                }

                if (this.schema.regex && this.modelValue) {
                    return RegExp(this.schema.regex).test(this.modelValue);
                }

                return true;
            },
        },
    };
</script>

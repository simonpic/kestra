<template>
    <div class="wrapper" v-if="enabled">
        <el-button @click="kill" type="warning" class="me-1">
            <stop-circle-outline />
            {{ $t("kill") }}
        </el-button>
    </div>
</template>
<script>
    import StopCircleOutline from "vue-material-design-icons/StopCircleOutline.vue";
    import {mapState} from "vuex";
    import permission from "../../models/permission";
    import action from "../../models/action";
    import State from "../../utils/state";

    export default {
        components: {StopCircleOutline},
        props: {
            execution: {
                type: Object,
                required: true
            }
        },
        methods: {
            kill() {
                this.$toast()
                    .confirm(this.$t("killed confirm", {id: this.execution.id}), () => {
                        return this.$store
                            .dispatch("execution/kill", {
                                id: this.execution.id,
                            })
                            .then(() => {
                                this.$toast().success(this.$t("killed done"));
                            })
                    });
            }
        },
        computed: {
            ...mapState("auth", ["user"]),
            enabled() {
                if (!(this.user && this.user.isAllowed(permission.EXECUTION, action.DELETE, this.execution.namespace))) {
                    return false;
                }

                return State.isKillable(this.execution.state.current);
            }
        }
    };
</script>

<style lang="scss">
.wrapper {
    display: inline;
}
</style>

<template>
    <div class="create-layout clearfix" v-bkloading="{ isLoading: $loading() }">
        <label class="create-label fl">{{$t('添加主机')}}</label>
        <div class="create-hosts">
            <bk-button class="select-host-button" theme="default"
                @click="handleAddHost">
                <i class="bk-icon icon-plus"></i>
                {{$t('添加主机')}}
            </bk-button>
            <div class="create-tables" ref="createTables">
                <transition-group name="service-table-list" tag="div">
                    <service-instance-table class="service-instance-table"
                        v-for="(data, index) in hosts"
                        ref="serviceInstanceTable"
                        deletable
                        expanded
                        :key="data.host.bk_host_id"
                        :index="index"
                        :id="data.host.bk_host_id"
                        :name="data.host.bk_host_innerip"
                        :source-processes="sourceProcesses"
                        @delete-instance="handleDeleteInstance">
                    </service-instance-table>
                </transition-group>
            </div>
        </div>
        <div class="buttons" :class="{ 'is-sticky': hasScrollbar }">
            <cmdb-auth class="mr5" :auth="$authResources({ type: $OPERATION.C_SERVICE_INSTANCE })">
                <bk-button slot-scope="{ disabled }"
                    theme="primary"
                    :disabled="!hosts.length || disabled"
                    @click="handleConfirm">
                    {{$t('确定')}}
                </bk-button>
            </cmdb-auth>
            <bk-button @click="handleBackToModule">{{$t('取消')}}</bk-button>
        </div>
        <cmdb-dialog v-model="dialog.show" :width="850" :height="460" :show-close-icon="false">
            <component
                :is="dialog.component"
                v-bind="dialog.props"
                @confirm="handleDialogConfirm"
                @cancel="handleDialogCancel">
            </component>
        </cmdb-dialog>
    </div>
</template>

<script>
    import HostSelector from '@/views/business-topology/host/host-selector'
    import serviceInstanceTable from '@/components/service/instance-table.vue'
    import { addResizeListener, removeResizeListener } from '@/utils/resize-events'
    export default {
        name: 'clone-to-other',
        components: {
            serviceInstanceTable,
            [HostSelector.name]: HostSelector
        },
        props: {
            module: {
                type: Object,
                default () {
                    return {}
                }
            },
            sourceProcesses: {
                type: Array,
                default () {
                    return {}
                }
            }
        },
        data () {
            return {
                dialog: {
                    show: false,
                    component: null,
                    props: {}
                },
                hosts: [],
                hasScrollbar: false
            }
        },
        computed: {
            business () {
                return this.$store.getters['objectBiz/bizId']
            },
            hostId () {
                return parseInt(this.$route.params.hostId)
            },
            moduleId () {
                return parseInt(this.$route.params.moduleId)
            },
            setId () {
                return parseInt(this.$route.params.setId)
            }
        },
        mounted () {
            addResizeListener(this.$refs.createTables, this.resizeHandler)
        },
        beforeDestroy () {
            removeResizeListener(this.$refs.createTables, this.resizeHandler)
        },
        methods: {
            handleAddHost () {
                this.dialog.component = HostSelector.name
                this.dialog.props = {
                    exist: this.hosts,
                    exclude: [this.hostId]
                }
                this.dialog.show = true
            },
            handleDialogConfirm (selected) {
                this.hosts = selected
                this.dialog.show = false
            },
            handleDialogCancel () {
                this.dialog.show = false
            },
            handleDeleteInstance (index) {
                this.hosts.splice(index, 1)
            },
            async handleConfirm () {
                try {
                    const serviceInstanceTables = this.$refs.serviceInstanceTable
                    await this.$store.dispatch('serviceInstance/createProcServiceInstanceWithRaw', {
                        params: this.$injectMetadata({
                            name: this.module.bk_module_name,
                            bk_module_id: this.moduleId,
                            instances: serviceInstanceTables.map(table => {
                                return {
                                    bk_host_id: table.id,
                                    processes: table.processList.map(item => {
                                        return {
                                            process_info: item
                                        }
                                    })
                                }
                            })
                        }, { injectBizId: true })
                    })
                    this.$success(this.$t('克隆成功'))
                    this.handleBackToModule()
                } catch (e) {
                    console.error(e)
                }
            },
            handleBackToModule () {
                this.$routerActions.back()
            },
            resizeHandler () {
                this.$nextTick(() => {
                    const scroller = this.$el.parentElement
                    this.hasScrollbar = scroller.scrollHeight > scroller.offsetHeight
                })
            }
        }
    }
</script>

<style lang="scss" scoped>
    .create-layout {
        margin: 35px 0 0 0;
        font-size: 14px;
        color: #63656E;
    }
    .create-label{
        display: block;
        width: 100px;
        position: relative;
        line-height: 32px;
        text-align: right;
        &:after {
            content: "*";
            margin: 0 0 0 4px;
            color: $cmdbDangerColor;
            @include inlineBlock;
        }
    }
    .create-hosts {
        padding-left: 10px;
        padding-right: 20px;
        height: 100%;
        overflow: hidden;
    }
    .select-host-button {
        height: 32px;
        line-height: 30px;
        font-size: 0;
        .bk-icon {
            position: static;
            height: 30px;
            line-height: 30px;
            font-size: 12px;
            font-weight: bold;
            @include inlineBlock(top);
        }
        /deep/ span {
            font-size: 14px;
        }
    }
    .create-tables {
        height: calc(100% - 54px);
        margin: 20px 0 0 0;
        @include scrollbar-y;
        position: relative;
    }
    .buttons {
        position: sticky;
        bottom: 0;
        left: 0;
        padding: 10px 0 10px 110px;

        &.is-sticky {
            background-color: #FFF;
            border-top: 1px solid $borderColor;
            z-index: 100;
        }
    }
    .service-instance-table {
        margin-bottom: 12px;
    }

    .service-table-list-enter-active, .service-table-list-leave-active {
        transition: all .7s ease-in;
    }
    .service-table-list-leave-to {
        opacity: 0;
        transform: translateX(30px);
    }
</style>

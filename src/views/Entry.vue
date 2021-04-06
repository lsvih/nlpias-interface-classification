<template>
    <div class="interface-panel">
        <div class="text-panel">
            <div class="text" v-html="text"/>
        </div>
        <div class="label-panel">
            <button @click="handleLabel(label)" class="label" v-for="(label, index) in labels"
                    :style="{backgroundColor: colors[index]}"
                    v-bind:key="index">
                {{ label }}
                <span v-if="index === labels.indexOf(predict)" class="auto"></span>
                <span class="shortcut" v-if="shortcuts[index] !== ''">{{ shortcuts[index] }}</span>
            </button>
            <button class="label">+</button>
        </div>
        <div class="config-btn">
            <document-icon v-if="!!information.project && (!!information.project.document)"
                           @click.native="openDocument"/>
            <setting-icon @click.native="show_config = true"/>
        </div>
        <a-progress :percent="idx / total * 100" :show-info="false" class="progress" :stroke-width="14"
                    stroke-color="blue" stroke-linecap="square"/>
        <config-panel :visible="show_config" :labels="labels" :shortcuts="shortcuts" @close="show_config = false"
                      @save="changeHotkey"/>
    </div>
</template>

<script>

import {schemeSet2, schemeSet3} from 'd3-scale-chromatic'
import DocumentIcon from '../components/icons/document'
import SettingIcon from '../components/icons/setting'
import ConfigPanel from '../components/ConfigPanel'
import commonMethods from '../common'

/*
Events who end with [Annotation] is the event for annotaion panel, would be destroy while closing the annotation interface.

schema
emit:
getData(task_id, idx) 向 Main 索取 task_id 任务的第 idx 条数据
labelData(task_id, idx, label) 通知 Main 标注成功 task_id 任务的第 idx 条数据，结果为 label

on:
sentData(text, ?predict) 推送回 getData 请求的数据与预测

 */

export default {
    name: 'Entry',
    components: {DocumentIcon, SettingIcon, ConfigPanel},
    data() {
        return {
            title: 'Interface',
            information: {},
            total: null,
            task_id: null,
            idx: 0,
            colors: [...schemeSet2, ...schemeSet3],
            shortcuts: [],
            labels: [],
            predict: [],
            text: '',
            show_config: false
        }
    },
    created() {
        console.log('Remote Interface Created.')
        this.registerEvents()
    },
    methods: {
        ...commonMethods,
        registerEvents() {
            this.$eventBus.on('Destroy[Annotation]', () => {
                this.destroyAnnotator()
            })
            this.$eventBus.on('Mount[Annotation]', params => {
                this.initAnnotator(params)
            })
            this.$eventBus.on('sentData[Annotation]', data => {
                let text, label, predict
                [text, label, predict] = data
                this.text = text.replace(/\\n/g, '<br />')
                this.predict = predict || []
                this.predict = label || []
            })
        },
        handleLabel(label) {
            if (label === 'Previous') { // get previous data
                if (this.idx === 0) { // prevent event while on first data
                    this.$message.info('It\'s already the first data.')
                    return
                }
                this.idx -= 1
                this.getData(this.idx)
            } else {
                if (label === 'Next')
                    this.$eventBus.emit('labelData[Annotation]', [this.task_id, this.idx, ''])
                else
                    this.$eventBus.emit('labelData[Annotation]', [this.task_id, this.idx, label])
                if (this.idx === this.information.task.size - 1) {
                    this.$message.info('It\'s already the last data.')
                    return
                }
                this.idx += 1
                this.getData(this.idx)
            }
        }
    }
}
</script>

<style scoped>
.interface-panel {
    width: 100%;
    height: 100%;
    z-index: 2;
    background-color: #fff;
    position: relative;
}

.text-panel {
    border-radius: 20px;
    border: 1px solid #ccc;
    padding: 20px;
    width: 80vw;
    max-height: 60vh;
    overflow-y: auto;
    margin: 10vh 10vw;
    font-size: 22px;
}

.label-panel {
    position: fixed;
    bottom: 5vh;
    margin: 0 10vw;
}

.label {
    position: relative;
    background-color: #ddd;
    height: 60px;
    font-size: 25px;
    padding: 0 25px;
    border-radius: 10px;
    border: none;
    margin-right: 70px;
    cursor: pointer;
}

.label:hover {
    filter: brightness(70%);
}

.shortcut {
    display: inline-block;
    height: 25px;
    width: 25px;
    line-height: 25px;
    vertical-align: middle;
    font-size: 15px;
    color: #000;
    font-weight: bold;
    padding: 0;
    border: 2px solid #000;
    border-radius: 5px;
    opacity: 0.2;
}

.config-btn {
    position: fixed;
    right: 20px;
    bottom: 5vh;
    padding-bottom: 10px;
}

.auto {
    display: inline-block;
    position: absolute;
    height: 10px;
    width: 10px;
    background-color: yellow;
    border-radius: 5px;
    left: 5px;
    top: 5px;
}

.progress {
    position: fixed;
    bottom: 0;
    left: 0;
}
</style>

<style>
.interface-container .progress .ant-progress-inner {
    border-radius: 0 !important;
    vertical-align: bottom !important;
}
</style>
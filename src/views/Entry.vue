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
            predict: null,
            text: '',
            show_config: false
        }
    },
    created() {
        console.log('Remote Interface Created.')
        this.registerEvents()
    },
    methods: {
        registerEvents() {
            this.$eventBus.on('Destroy[Annotation]', () => {
                Object.keys(this.$eventBus.getEvents())
                    .filter(e => e.endsWith('[Annotation]'))
                    .forEach(eventName => {
                            for (let eventFunc of this.$eventBus.getEvents()[eventName].values())
                                this.$eventBus.off(eventName, eventFunc)
                        }
                    )
            })
            this.$eventBus.on('Mount[Annotation]', params => {
                console.log('Loading interface: params:', params)
                this.information = params
                this.labels = JSON.parse(this.information['project']['label']).concat(['Previous', 'Next'])
                this.idx = this.information['task']['current']
                this.total = this.information['task']['size']
                this.task_id = this.information['task']['id']
                this.initKeyboardEvents()
                this.getData(this.idx)
            })
            this.$eventBus.on('sentData[Annotation]', data => {
                let text, predict
                if (data.length === 2)
                    [text, predict] = data
                else
                    [text, predict] = [data[0], null]

                this.text = text.replace(/\\n/g, '<br />')
                this.predict = predict
            })
        },
        initKeyboardEvents() {
            let project_key = `Project:${this.information.project.id}`
            if (localStorage.getItem(project_key) === null) {
                // Initialize project config
                let config = {hotkey: {}} // {label: key}
                this.shortcuts = []
                localStorage.setItem(project_key, JSON.stringify(config))
            } else {
                let config = JSON.parse(localStorage.getItem(project_key))
                let hotkey = config['hotkey']
                for (let label of this.labels) {
                    if (hotkey[label] !== undefined && hotkey[label] !== null && hotkey[label] !== '') {
                        this.shortcuts.push(hotkey[label].toUpperCase())
                        this.$eventBus.on(`PressKey[${hotkey[label]}][Annotation]`, () => this.handleLabel(label))
                        console.log(this.$eventBus.getEvents())
                    } else {
                        this.shortcuts.push('')
                    }
                }
            }
        },
        changeHotkey(hotkeys) {
            this.show_config = false
            this.shortcuts = hotkeys
            // Delete previous key event
            Object.keys(this.$eventBus.getEvents())
                .filter(e => e.startsWith('PressKey'))
                .forEach(eventName => {
                        for (let eventFunc of this.$eventBus.getEvents()[eventName].values())
                            this.$eventBus.off(eventName, eventFunc)
                    }
                )
            let hotkey_config = {}
            for (let i = 0; i < this.labels.length; i++) {
                let label = this.labels[i]
                hotkey_config[label] = this.shortcuts[i].toLowerCase()
                console.log(`Register ${hotkey_config[label]} for ${label}`)
                if (hotkey_config[label] !== undefined && hotkey_config[label] !== null && hotkey_config[label] !== '')
                    this.$eventBus.on(`PressKey[${hotkey_config[label]}][Annotation]`, () => this.handleLabel(label))
            }
            let project_key = `Project:${this.information.project.id}`
            let config = JSON.parse(localStorage.getItem(project_key))
            config['hotkey'] = hotkey_config
            localStorage.setItem(project_key, JSON.stringify(config))
        },
        getData(idx) {
            // 获取第 idx 条数据
            this.$eventBus.emit('getData[Annotation]', [this.task_id, idx])
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
                this.idx += 1
                this.getData(this.idx)
            }
        },
        openDocument() {
            this.$eventBus.emit('openDocument[Annotation]', this.information.project.document)
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
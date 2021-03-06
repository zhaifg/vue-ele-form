<template>
<div class="ele-form" :class="{ 'ele-form--inline': inline }" ref="wrapper">
    <el-row justify="center" type="flex">
        <el-col :span="computedSpan">
            <el-form :label-position="computedLabelPosition" :label-width="computedLabelWidth" :model="formData" :rules="computedRules" @submit.native.prevent="handleSubmitForm" ref="form" :disabled="disabled" v-bind="formAttrs">
                <!-- 默认插槽作为表单项 -->
                <slot />
                <el-row :gutter="20">
                    <slot :formData="formData" :formDesc="computedFormDesc" :formErrorObj="formErrorObj" :props="$props" name="form-content">
                        <template v-for="(formItem, field) of computedFormDesc">
                            <slot :name="field + '-wrapper'" :data="formData[field]" :desc="formItem" :field="field" :props="$props" :formData="formData" :disabled="formItem._disabled" :type="formItem._type" :options="formItem._options">
                                <el-col :key="field" v-bind="formItem._colAttrs" v-if="formItem._vif">
                                    <el-form-item :error="formErrorObj ? formErrorObj[field] : null" :label="
                        isShowLabel && formItem.isShowLabel !== false
                          ? formItem.label
                          : null
                      " :label-width="formItem.labelWidth || null" :prop="field">
                                        <!-- 具名 作用域插槽(用于用户自定义显示) -->
                                        <slot :data="formData[field]" :desc="formItem" :props="$props" :field="field" :formData="formData" :name="field" :disabled="formItem._disabled" :type="formItem._type" :options="formItem._options">
                                            <component :disabled="formItem._disabled" :readonly="readonly" :desc="formItem" :is="formItem._type" :options="formItem._options" :ref="field" :field="field" :value="getValue(field)" @input="setValue(field, $event)" />
                                        </slot>
                                        <div class="ele-form-tip" v-if="formItem._tip" v-html="formItem._tip"></div>
                                    </el-form-item>
                                </el-col>
                            </slot>
                        </template>
                    </slot>
                    <!-- 操作按钮区 -->
                    <el-col class="ele-form-btns" v-if="btns.length">
                        <el-form-item :label-width="inline ? '10px' : null">
                            <!-- 按钮插槽 -->
                            <slot :btns="btns" name="form-btn">
                                <el-button :key="index" @click="btn.click" v-bind="btn.attrs" v-for="(btn, index) of btns">{{ btn.text }}</el-button>
                            </slot>
                        </el-form-item>
                    </el-col>
                </el-row>
            </el-form>
        </el-col>
    </el-row>
</div>
</template>

<script>
import responsiveMixin from './mixins/responsiveMixin'
import {
    isUnDef,
    isFunction,
    is,
    castArray,
    getDeepVal,
    setDeepVal,
    isEmpty
} from './tools/utils'
import {
    throttle
} from 'throttle-debounce'
import localeMixin from 'element-ui/src/mixins/locale'
import {
    t
} from './locale'
import {
    loadMockJs
} from './tools/mock'
import {
    equal,
    intersection
} from './tools/set'
const isNumber = require('is-number')
const cloneDeep = require('lodash.clonedeep')

export default {
    name: 'EleForm',
    // 响应式单独抽离出来作为mixin, 具体实现请到 responsiveMixin 中查看
    mixins: [responsiveMixin, localeMixin],
    model: {
        prop: 'formData',
        event: 'input'
    },
    props: {
        // 表单描述
        formDesc: {
            type: Object,
            required: true
        },
        // 表单数据
        formData: {
            type: Object,
            required: true
        },
        // 行内模式
        inline: {
            type: Boolean,
            default: false
        },
        // 表单自身属性
        formAttrs: Object,
        // 校检规则
        rules: {
            type: Object,
            default () {
                return {}
            }
        },
        // 模拟数据
        mock: {
            type: Boolean,
            default: false
        },
        // 提交状态
        isLoading: {
            type: Boolean,
            default: false
        },
        // 表单错误信息
        formError: Object,
        // 提交函数
        requestFn: Function,
        // 自定义表单按钮
        formBtns: Array,
        // 表单按钮大小
        formBtnSize: String,
        // 是否显示submit按钮
        isShowSubmitBtn: {
            type: Boolean,
            default: true
        },
        // 是否显示 cancel 取消按钮
        // 默认值: isDialog = true 时, 默认值为 true, 具体查看: computedIsShowCancelBtn
        isShowCancelBtn: {
            type: Boolean,
            default: null
        },
        // 是否显示back按钮
        // 默认值: 当 inline = true OR isDialog = true, 默认值为 false; 其它情况true. 具体请看计算属性: computedIsShowBackBtn
        isShowBackBtn: {
            type: Boolean,
            default: null
        },
        // 是否显示reset按钮
        isShowResetBtn: {
            type: Boolean,
            default: false
        },
        // 提交按钮文本
        // 默认值: 当 inline 为true时, 值为 '查询'; inline 为 false 时,  值为 '提交'. 具体请看计算属性: computedSubmitBtnText
        submitBtnText: {
            type: String,
            default: null
        },
        // 返回按钮
        backBtnText: String,
        // 重置按钮
        resetBtnText: String,
        // 取消按钮
        cancelBtnText: String,
        // 是否显示标签
        isShowLabel: {
            type: Boolean,
            default: true
        },
        // 标签宽度
        labelWidth: {
            type: [Number, String],
            default: 'auto'
        },
        // 全局禁用表单
        disabled: {
            type: Boolean,
            default: false
        },
        // 全局的readonly
        readonly: {
            type: Boolean,
            default: false
        },
        // 是否为弹窗
        isDialog: {
            type: Boolean,
            default: false
        },
        // 弹窗变量控制
        visible: {
            type: Boolean,
            default: false
        },
        // options 的请求方法
        optionsFn: Function
    },
    data() {
        return {
            // 是否正在请求中
            innerIsLoading: false,
            // 内部请求出错
            innerFormError: {}
        }
    },
    computed: {
        isMock() {
            return (
                this.mock ||
                Object.values(this.computedFormDesc).some(item => item.mock)
            )
        },
        // 按钮
        btns() {
            const formBtnSize = this.formBtnSize
            let btns = []
            // 模拟数据
            if (this.isMock) {
                btns.push({
                    attrs: {
                        type: 'primary',
                        size: formBtnSize
                    },
                    text: t('ele-form.mockBtnText'),
                    click: this.reMockData
                })
            }

            // 提交按钮
            if (this.isShowSubmitBtn) {
                btns.push({
                    attrs: {
                        type: 'primary',
                        size: formBtnSize,
                        loading: this.isLoading || this.innerIsLoading,
                        'native-type': 'submit'
                    },
                    text: this.computedSubmitBtnText,
                    click() {}
                })
            }

            // 自定义按钮
            if (this.formBtns) {
                const customBtns = this.formBtns.map(btn => ({
                    attrs: {
                        type: btn.type,
                        size: formBtnSize
                    },
                    text: btn.text,
                    click: btn.click
                }))
                btns = [...btns, ...customBtns]
            }

            // 返回按钮
            if (this.computedIsShowBackBtn) {
                btns.push({
                    attrs: {
                        size: formBtnSize
                    },
                    text: this.backBtnText || t('ele-form.backBtnText'),
                    click: this.goBack
                })
            }

            // 取消按钮
            if (this.computedIsShowCancelBtn) {
                btns.push({
                    attrs: {
                        size: formBtnSize
                    },
                    text: this.cancelBtnText || t('ele-form.cancelBtnText'),
                    click: this.handleCancelClick
                })
            }

            // 重置按钮
            if (this.isShowResetBtn) {
                btns.push({
                    attrs: {
                        size: formBtnSize
                    },
                    text: this.resetBtnText || t('ele-form.resetBtnText'),
                    click: this.resetForm
                })
            }
            return btns
        },
        computedIsShowCancelBtn() {
            if (is(this.isShowCancelBtn, 'Boolean')) {
                // 如果指定了, 则使用指定的值
                return this.isShowCancelBtn
            } else {
                // 如果未指定, 根据 isDialog
                return this.isDialog
            }
        },
        // 是否显示返回按钮(inline和layout模式下不同)
        computedIsShowBackBtn() {
            if (is(this.isShowBackBtn, 'Boolean')) {
                return this.isShowBackBtn
            } else {
                return !(this.inline || this.isDialog)
            }
        },
        // 提交按钮默认值(inline和layout模式下不同)
        computedSubmitBtnText() {
            if (is(this.submitBtnText, 'String')) {
                return this.submitBtnText
            } else {
                return this.inline ?
                    t('ele-form.submitBtnTextInline') :
                    t('ele-form.submitBtnText')
            }
        },
        // 标签宽度(数字和字符串两种处理)
        computedLabelWidth() {
            if (isNumber(this.labelWidth)) {
                return this.labelWidth + 'px'
            } else {
                return this.labelWidth
            }
        },
        // 表单错误信息
        formErrorObj() {
            return Object.assign({}, this.innerFormError, this.formError)
        },
        // 校检规则 (支持局部定义和全局定义)
        // 即: rules: { rules: { a: [xxx, xxx], b:{ xxx } } } 和 formDesc: { name: { rules: {xxx} }, age: { rules: [xxx] } }
        // 此函数即将局部定义转为全局定义
        computedRules() {
            return this.formDescKeys.reduce((rules, field) => {
                // 合并 (全局 和 局部) 的rules
                let formRules = castArray(this.rules[field])
                let formItemRules = castArray(this.computedFormDesc[field].rules)
                rules[field] = [...formRules, ...formItemRules]

                // 如果采用required, 则判断已有的规则有无, 如果没有, 则添加
                if (
                    this.computedFormDesc[field].required &&
                    !rules[field].some(rule => rule.required)
                ) {
                    rules[field].push({
                        required: true,
                        message: this.computedFormDesc[field].label + t('ele-form.required')
                    })
                }
                return rules
            }, {})
        },
        // formDesc的key
        formDescKeys() {
            return Object.keys(this.computedFormDesc)
        },
        // 对formDesc做处理
        computedFormDesc() {
            const desc = this.getDeepFormDesc(this.formDesc)
            Object.keys(desc).forEach(field => {
                // 当全局设置mock为true时, 所有子项都标记为true
                if (this.mock && isUnDef(desc[field].mock)) {
                    desc[field].mock = true
                }

                // 转换 tip, 内部属性不显示
                if (desc[field].tip) {
                    desc[field]._tip = String(desc[field].tip).replace(
                        /`(.+?)`/g,
                        '<code>$1</code>'
                    )
                }

                // layout值, 内部属性不显示
                desc[field]._colAttrs = this.getColAttrs(desc[field].layout)

                // 老数据, 用于options切换不同类型和type切换不懂类型时, 保留旧数据
                // 例如 原type为 switch, 后改为 input, 出现类型和值不兼容情况, 就需要保留原数据
                if (!desc[field]._oldValue) {
                    desc[field]._oldValue = {}
                }

                // 隐藏属性
                Object.keys(desc[field]).forEach(key => {
                    if (key.startsWith('_')) {
                        Object.defineProperty(desc[field], key, {
                            enumerable: false,
                            writable: true,
                            configurable: true,
                            value: desc[field][key]
                        })
                    }
                })
            })
            return desc
        }
    },
    watch: {
        // 处理options参数
        computedFormDesc: {
            handler(desc) {
                if (desc) {
                    Object.keys(desc).forEach(field => {
                        // 设置深度边遍历的默认值
                        this.setDeepFormDataVal(field)
                    })

                    // 检查联动
                    this.checkLinkage()
                }
            },
            immediate: true
        },
        formErrorObj(obj) {
            // 后端异常的弹窗警告
            if (obj) {
                this.processError(obj)
            }
        },
        formData: {
            handler(formData) {
                if (formData) {
                    // 联动属性检测
                    this.checkLinkage()
                }
            },
            deep: true
        }
    },
    methods: {
        // 获取值
        // 'a.b.c' => this.formData.a.b.c
        getValue(field) {
            return getDeepVal(this.formData, field)
        },
        // 设置值
        // 例如 ('a.b.c', 'val') => this.formData.a.b.c = 'val'
        setValue(field, value) {
            const formData = cloneDeep(this.formData)
            setDeepVal(formData, field, value, true)
            this.$emit('input', formData)
        },
        // 给需要深度遍历的对象赋值空对象, 以便 v-model 时不出错
        // 例如 formDesc: { info: { children: { name:{ type: 'input', xxx }, nickname: {type: 'input', xxx } } } } => formData => { info: {} }
        setDeepFormDataVal(field) {
            const deepPath = field
                .split('.')
                .slice(0, -1)
                .join('.')
            if (deepPath) {
                setDeepVal(this.formData, deepPath, {})
            }
        },
        // 深度遍历 formDesc
        // 并且子项目继承 联动属性
        getDeepFormDesc(formDesc, parentField, parent) {
            return Object.keys(formDesc).reduce((acc, field) => {
                // 改为 => a.b.c
                const formField = parentField ? `${parentField}.${field}` : field
                // 合并 vif 和 disabled, children 要合并父的 vif 和 disabled
                let vifs = []
                let disableds = []
                if (formDesc[field].vif !== undefined) {
                    vifs.push(formDesc[field].vif)
                }
                if (formDesc[field].disabled !== undefined) {
                    disableds.push(formDesc[field].disabled)
                }
                if (parentField) {
                    vifs = [...parent._vifs, ...vifs]
                    disableds = [...parent._disableds, ...disableds]
                }
                formDesc[field]._vifs = vifs
                formDesc[field]._disableds = disableds

                // 递归
                if (formDesc[field].children) {
                    Object.assign(
                        acc,
                        this.getDeepFormDesc(
                            formDesc[field].children,
                            formField,
                            formDesc[field]
                        )
                    )
                } else {
                    // 改变 options
                    this.changeOptions(formDesc[field], formDesc[field].options, field)

                    acc[formField] = formDesc[field]
                }

                return acc
            }, {})
        },
        // 获取col的属性(是否为inline模式)
        getColAttrs(layout) {
            return this.inline ? {
                span: layout || 6
            } : {
                md: layout || 24,
                xs: 24
            }
        },
        // 重新模拟数据
        reMockData() {
            this.formDescKeys.forEach(field => {
                this.$refs[field][0].mockData()
            })
        },
        // 检测联动
        checkLinkage() {
            if (this.checkVifFn) {
                this.checkLinkageFn()
            } else {
                this.checkLinkageFn = throttle(300, () => {
                    const formData = this.formData
                    Object.keys(this.computedFormDesc).forEach(field => {
                        const formItem = this.computedFormDesc[field]
                        // 1.设置 type
                        let type = formItem.type
                        if (isFunction(formItem.type)) {
                            type = this.getComponentName(formItem.type(formData))

                            // 是否需要载入旧数据
                            if (
                                formItem.isTypeChangeReloadValue !== false &&
                                formItem._type &&
                                formItem._type !== type
                            ) {
                                // 保存老数据并获取新类型的数据
                                const oldKey = `${field}._oldValue.type-${formItem._type}`
                                const newKey = `${field}._oldValue.type-${type}`
                                setDeepVal(this.formDesc, oldKey, formData[field])
                                const newVal = getDeepVal(formItem, newKey, null)
                                setDeepVal(this.formData, field, newVal)
                            }
                        } else {
                            type = this.getComponentName(formItem.type)
                        }

                        // 2.触发 v-if 显示 / 隐藏
                        const vif = formItem._vifs.every(vif =>
                            isFunction(vif) ? vif(this.formData) : vif
                        )

                        // 3.触发 disabled 禁用 / 启用
                        const disabled = formItem._disableds.some(disabled =>
                            isFunction(disabled) ? disabled(formData) : disabled
                        )

                        // 4.默认值
                        let defaultValue = isFunction(formItem.default) ?
                            formItem.default(formData) :
                            formItem.default
                        // 默认值不为空  & (值为空 || 老值和当前值)
                        if (
                            !isEmpty(defaultValue) &&
                            (isEmpty(this.formData[field]) ||
                                formItem._defaultValue === this.formData[field])
                        ) {
                            // 判断是否有格式化函数
                            if (this.computedFormDesc[field].displayFormatter) {
                                defaultValue = this.desc.displayFormatter(
                                    defaultValue,
                                    this.formData
                                )
                            }

                            setDeepVal(this.formData, field, defaultValue)
                        }

                        // 如果隐藏, 则删除值
                        if (!vif) {
                            setDeepVal(this.formData, field, defaultValue || null)
                        }

                        // 5.动态属性 attrs
                        let attrs = isFunction(formItem.attrs) ?
                            formItem.attrs(this.formData) :
                            formItem.attrs

                        const fullPath = field.split('.').join('.children.')
                        setDeepVal(this.formDesc, fullPath + '._type', type)
                        setDeepVal(this.formDesc, fullPath + '._vif', vif)
                        setDeepVal(this.formDesc, fullPath + '._disabled', disabled)
                        setDeepVal(this.formDesc, fullPath + '._defaultValue', defaultValue)
                        setDeepVal(this.formDesc, fullPath + '._attrs', attrs)

                        // 5.重新获取 options
                        if (formItem.isReloadOptions) {
                            this.changeOptions(
                                getDeepVal(this.formDesc, fullPath),
                                formItem.options,
                                field,
                                true
                            )
                        }
                    })
                })
                this.checkLinkageFn()
            }
        },
        // 组件名称
        getComponentName(type) {
            if (this.$EleFormBuiltInNames.includes(type)) {
                // 内置组件
                return 'ele-form-' + type
            } else {
                // 外部组件
                return type
            }
        },
        // 转对象的key
        // 例如 option: { label: '女', val: 1 }, prop: { text: 'label', value: 'val' }
        // 转换后 -> option: { text: '女', value: 1 }
        changeProp(options, prop) {
            if (prop) {
                return options.map(option => ({
                    text: option[prop.text || 'text'],
                    value: option[prop.value || 'value']
                }))
            } else {
                return options
            }
        },
        // 将options转为对象数组
        getObjArrOptions(options) {
            return options.map(option => {
                if (is(option, ['Number', 'String', 'Boolean'])) {
                    // 例如 ['男', '女'] => [ { text: '男', value: '男' }, { text: '女', value: '女' } ]
                    return {
                        text: option,
                        value: option
                    }
                } else {
                    // 对象 直接返回
                    return option
                }
            })
        },
        // 将四种类型: 字符串数组, 对象数组, Promise对象和函数统一为 对象数组
        changeOptions(formItem, options, field, reloadOptions = false) {
            // 当 _options 存在 且 非重新请求
            if (formItem._options !== undefined && reloadOptions === false) {
                return
            }

            if (options) {
                if (options instanceof Array) {
                    // 当options为数组时: 直接获取
                    this.setOptions(formItem, options, field, reloadOptions)
                } else if (options instanceof Function) {
                    // 当options为函数: 执行函数并递归
                    this.changeOptions(
                        formItem,
                        options(this.formData),
                        field,
                        reloadOptions
                    )
                } else if (options instanceof Promise) {
                    // 当options为Promise时: 等待Promise结束, 并获取值
                    options.then(newOptions => {
                        this.changeOptions(formItem, newOptions, field, reloadOptions)
                    })
                } else if (typeof options === 'string' && this.optionsFn) {
                    // options为url地址
                    this.changeOptions(
                        formItem,
                        this.optionsFn(options),
                        field,
                        reloadOptions
                    )
                } else {
                    if (typeof options === 'string') {
                        throw new TypeError(
                            `options值为: ${options}, 为字符串, 但是未配置options-fn参数, 具体请参考: https://www.yuque.com/chaojie-vjiel/vbwzgu/rgenav#ZVYtf`
                        )
                    } else {
                        // 其他报错
                        throw new TypeError(
                            'options的类型不正确, options及options请求结果类型可为: 字符串数组, 对象数组, 函数和Promise或者URL地址, 当前值为: ' +
                            options +
                            ', 不属于以上四种类型, 具体请参考: https://www.yuque.com/chaojie-vjiel/vbwzgu/rgenav'
                        )
                    }
                }
            } else {
                if (formItem._options) {
                    // 如果new options为null / undefined, 且 old Options 存在, 则设置
                    this.setOptions(formItem, [], field, reloadOptions)
                }
            }
        },
        // 设置options
        setOptions(formItem, options, field, reloadOptions) {
            // 原 options值
            const oldOptions = cloneDeep(formItem._options)

            // 将options每一项转为对象
            let newOptions = this.getObjArrOptions(options)
            // 改变prop为规定的prop
            newOptions = this.changeProp(newOptions, formItem.prop)
            this.$set(formItem, '_options', newOptions)

            // 判断是否需要重置值
            // isOptionsChangeReloadValue 默认为 true
            // 当值变化时 reloadOptions 为 true
            // 当第一次时, oldOptions 是为 undefined, 不需要
            if (
                formItem.isOptionsChangeReloadValue !== false &&
                reloadOptions &&
                oldOptions !== undefined
            ) {
                const newOptionValues = new Set(
                    Array.isArray(newOptions) ? newOptions.map(item => item.value) : []
                )
                const oldOptionValues = new Set(
                    Array.isArray(oldOptions) ? oldOptions.map(item => item.value) : []
                )

                // 1.没并集 & 2.原 oldOptions 且和 newOptions 不相等, 则重置 value 值
                const isIntersection = intersection(newOptionValues, oldOptionValues)
                    .size
                const isEqual = equal(newOptionValues, oldOptionValues)

                if (!isIntersection && !isEqual) {
                    // 保留旧值
                    const oldKey = `options-${[...oldOptionValues].join('_')}`
                    const newKey = `options-${[...newOptionValues].join('_')}`
                    formItem['_oldValue'][newKey] = this.formData[field]
                    const newVal = formItem['_oldValue'][oldKey] || null
                    this.setValue(field, newVal)
                }
            }
        },
        // 验证表单
        validateForm() {
            return new Promise((resolve, reject) => {
                if (this.computedRules) {
                    // 当传递了验证规则
                    this.$refs['form'].validate((valid, invalidFields) => {
                        if (valid) {
                            // 验证通过
                            resolve()
                        } else {
                            // 显示错误
                            reject(invalidFields)
                        }
                    })
                } else {
                    resolve()
                }
            })
        },

        // 验证所有组件的内置验证方法
        validateComponent() {
            const validators = this.formDescKeys
                .map(key =>
                    this.$refs[key] && this.$refs[key][0] ?
                    this.$refs[key][0].validate :
                    null
                )
                .filter(Boolean)

            return Promise.all(validators.map(fn => fn()))
        },

        // 处理错误
        processError(errObj) {
            try {
                const messageArr = Object.keys(errObj).reduce((acc, key) => {
                    const formItem = this.computedFormDesc[key]
                    const label =
                        formItem && formItem.label ? formItem.label + ': ' : key + ': '
                    if (errObj[key] instanceof Array) {
                        // errorObj: { name: [ { filed: 'name',  message: 'name is required' }] }
                        // 内部校检结果返回的错误信息样式
                        errObj[key].forEach(item => {
                            acc.push(label + item.message)
                        })
                    } else {
                        // errorObj: { name: 'name is required' }
                        // 外部校检返回的错误信息
                        acc.push(label + errObj[key])
                    }

                    return acc
                }, [])
                this.showError(messageArr)
                // eslint-disable-next-line
            } catch {}
        },

        // 显示错误
        showError(messageArr) {
            if (messageArr.length) {
                const h = this.$createElement
                messageArr = messageArr.map(msg => {
                    return h('div', {
                        style: {
                            marginBottom: '8px'
                        }
                    }, msg)
                })
                this.$notify.error({
                    title: t('ele-form.formError'),
                    message: h(
                        'div', {
                            style: {
                                minWidth: '300px',
                                marginTop: '12px'
                            }
                        },
                        messageArr
                    )
                })
            }
        },

        // 提交表单
        async handleSubmitForm() {
            try {
                // 验证表单
                await this.validateForm()
                await this.validateComponent()

                // 为了不影响原值, 这里进行 clone
                let data = cloneDeep(this.formData)
                // valueFormatter的处理
                for (const field in data) {
                    const formItem = this.computedFormDesc[field]

                    let a;

                    switch (formItem.type) {
                        case 'date':
                            data[field] = days(data[field] * 1000).format("YYYY-MM-DD");
                            break;
                        case 'datetime':
                            data[field] = days(data[field] * 1000).format("YYYY-MM-DD HH:mm:ss");
                            break;
                        case 'daterange':
                            a = new Array()
                            Array.isArray(data[field]) ? data[field].forEach(e => a.push(dayjs(e * 1000).format("YYYY-MM-DD"))) : data[field]
                            data[field] = a
                            break;
                        case 'time':
                            data[field] = days(data[field] * 1000).format("HH:mm:ss");
                            break;
                        case 'timerange':
                            a = new Array()
                            Array.isArray(data[field]) ? data[field].forEach(e => a.push(dayjs(e * 1000).format("HH:mm:ss"))) : data[field]
                            data[field] = a
                            break;
                        case 'datetimerange':
                            a = new Array()
                            Array.isArray(data[field]) ? data[field].forEach(e => a.push(dayjs(e * 1000).format("YYYY-MM-DD HH:mm:ss"))) : data[field]
                            data[field] = a
                            break;
                        default:
                            if (formItem && formItem.valueFormatter) {
                                data[field] = formItem.valueFormatter(data[field], data);
                            }
                            break;
                    }

                    // if (formItem && formItem.valueFormatter) {
                    //   data[field] = formItem.valueFormatter(data[field], data)
                    // }
                }

                if (this.requestFn) {
                    // 在内部请求
                    if (this.innerIsLoading) return
                    this.innerIsLoading = true
                    try {
                        const response = await this.requestFn(data)
                        this.$nextTick(() => {
                            this.$emit('request-success', response)
                        })
                    } catch (error) {
                        // 处理异常情况
                        if (error instanceof Error) {
                            // 返回的是Error类型, 则进行解析
                            try {
                                const msg = JSON.parse(error.message)
                                if (msg instanceof Object) {
                                    this.innerFormError = msg
                                }
                                // eslint-disable-next-line
                            } catch {}
                        } else if (error instanceof Object) {
                            // 返回的是对象类型, 则直接设置
                            this.innerFormError = error
                        }
                        this.$emit('request-error')
                    } finally {
                        this.innerIsLoading = false
                        this.$emit('request-end')
                    }
                } else {
                    // 在外部请求
                    if (this.isLoading) return
                    this.$emit('request', data)
                }
            } catch (error) {
                return this.processError(error)
            }
        },
        // 返回按钮
        goBack() {
            this.$emit('back')
            if (this.$router) {
                // vue-router
                this.$router.back()
            } else {
                // 浏览器history API
                history.back()
            }
        },
        // 点击取消按钮
        handleCancelClick() {
            this.$emit('close')
            this.$emit('cancel')
            this.$emit('update:visible', false)
        },
        // 重置表单
        resetForm() {
            this.$emit('reset')
            this.$refs.form.resetFields()

            // 调用内部方法进行值的重置
            this.$refs.form.fields.forEach(field => {
                this.formData[field.prop] = field.initialValue
            })
        }
    },
    mounted() {
        if (this.isMock && !window.Mock) {
            loadMockJs()
        }
    }
}
</script>

<style>
.ele-form--inline .ele-form-btns {
    width: auto;
}

.ele-form-tip {
    color: #909399;
    line-height: 1.5em;
    margin-top: 3px;
}

.ele-form-tip code {
    padding: 2px 4px;
    font-size: 90%;
    color: #c7254e;
    background-color: #f9f2f4;
    border-radius: 4px;
}

.ele-form-full-line.el-date-editor.el-input,
.el-date-editor.el-input__inner,
.ele-form-full-line.el-date-editor--daterange.el-input__inner,
.ele-form-full-line.el-date-editor--datetimerange.el-input__inner,
.ele-form-full-line.el-date-editor--timerange.el-input__inner,
.ele-form-full-line.el-date-editor--monthrange.el-input__inner,
.ele-form-full-line.el-cascader,
.ele-form-full-line.el-select,
.ele-form-full-line.el-autocomplete {
    width: 100%;
}
</style>
